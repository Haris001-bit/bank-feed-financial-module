# Deep dive: bank feed ingestion

[Back to README](../README.md)

---

Plaid gives you four moving parts: Link, the token exchange, webhooks and the transactions sync. Each looks simple in the documentation. The engineering is in the gaps between them: a sync that runs out of time halfway through, an owner who fixes a bank login in a window that never tells the server, an encryption key that can be wrong without anything failing loudly.

---

## Link tokens: new links and update mode

A new link token asks for transactions only, requests 730 days of history, sets the webhook address per link, and filters accounts to the usual depository types (checking, savings, cash management and the like) plus every credit subtype, so charge cards are offered even though the token endpoint does not accept them by name. The provider's user id is the owner's own id. A redirect URI is sent only when one is configured.

The token is refused before any bank window opens if the company or the provider account has no connection left, or if the token key fails its canary. Refusing here costs nothing; refusing after the exchange costs a connection.

Update mode (Reconnect, Add accounts) passes the existing access token and no products. It never exchanges and never uses a connection, because update mode creates no new item. Two things are easy to miss:

- **The provider sends no webhook for a fix made in your own update mode.** After a successful update, the page calls the sync with an after-update flag: the item is set active and the snapshot is flagged as changed. Without that call, the owner fixes the login and the summary keeps reporting a broken bank until something unrelated triggers a sync.
- **Clear "new accounts available" only when the account picker was shown.** A plain reconnect that never showed the picker must leave the prompt in place, or the owner loses the only signal that a new account is waiting.

---

## The exchange, in order

The README covers why the connection budget shapes this. The full order:

1. The token key must decrypt a connection already stored for that environment.
2. Duplicate accounts are refused: same mask and name as any live account of the company, whatever institution record the provider reports.
3. Consent is recorded.
4. A slot is reserved under an advisory lock: switch on, right environment, bank not already linked or being linked, and both the company's limit and the provider account's limit. Reservations expire after 10 minutes.
5. The ledger row becomes "exchanging" before the provider is called. It is counted and never expires.
6. The exchange runs once. A clear 4xx refusal sets the row to failed. A network error or a 5xx leaves it exchanging and sends the owner to support, and the health check counts those rows.
7. Connection, ciphertext and ledger state are stored in one transaction, retried because it is safe to repeat. If it still fails, the ciphertext is parked on the sealed ledger row for support.
8. Accounts are fetched and saved at once, and the first transaction sync runs after the reply, because the provider only starts sending update webhooks after an item's first sync.

---

## Verifying the webhook

```
raw   = request body as text            -- before any JSON parsing
env   = environment named in the body   -- selects which keys to use
token = verification header (a JWT)

require token.alg == ES256
key = cache[token.kid] or fetch from the provider for env
      provider unreachable  -> 503, so the provider retries
require key not expired by the provider
require ecdsa_p256_verify(token, key)
require now - 5 min <= token.iat <= now
require constant_time_equal(sha256_hex(raw), token.body_hash)

any failure -> 401, nothing stored
```

Reading the raw body first is not optional: a parse and re-serialise can change whitespace or key order, and the signed hash covers the bytes the provider sent. Key ids are format-checked and the key cache is bounded.

---

## Record first, reply, then sync

Before replying, the receiver records what the webhook says about the item. "Sync updates available" sets the history flags and asks for a sync. A login-required error, a revoked permission, a pending expiry, a repaired login, new accounts and a single revoked account each map to a stored state. Legacy and unknown codes are ignored. If recording fails, the reply is 500 so the provider retries.

A webhook for a removed item, for another environment's item or for an unknown item is ignored. **A late webhook can never bring a removed connection back.**

The sync runs after the reply, through the runtime's background mechanism, inline within about 7 s where that is missing, and only for companies whose switch is on. If a sync is already running for the item, it is asked to run once more. A request is cleared only when it is run, and one that lands as a sync is finishing is picked up by that sync.

---

## The dedupe key

The key is item, webhook type, code, the history flags, the error code and a 5-minute bucket.

The flags are there for a specific reason. The provider sends a "30 days of history ready" update and, minutes later, a "full history ready" update with the same type and code. A key without the flags would drop the second one as a repeat, and the older history would wait for some unrelated event to trigger another sync.

Even so, deduplication is only an optimisation. Identical bodies an hour apart both sync, which is fine: the idempotency that matters is the cursor.

---

## The sync loop

```
claim the per-item lock (5 minutes)
start = saved cursor
cursor = start
loop
    page = transactions_sync(cursor, count = 500)
    upsert page.added and page.modified, keyed on transaction id
    delete page.removed, owner-scoped, in chunks of 100
    if page.has_more: cursor = page.next_cursor; continue
    compare_and_swap(saved cursor, expected = start, new = page.next_cursor)
    break
on mutation during pagination: restart from the saved cursor
on running out of time: stop; the next run starts from the saved cursor
release the lock
then: refresh accounts and item status, re-sort the company, flag the snapshot
```

Pages are upserted as they arrive, but the cursor moves only once every page is in. If a run dies on page three, the next run reads again from the saved cursor, upserts pages one to three again (harmless, because they are keyed on transaction id) and carries on. A mutation during pagination restarts from the saved cursor, so the saved cursor has to still be the one the pagination began with; committing per page would leave nothing correct to restart from.

The compare-and-swap backs up the lock: if another process moved the cursor after this run started, the swap fails instead of rewinding or skipping rows nobody saved.

Other rules inside the loop:

- **One sign convention.** The provider reports money out as positive on every account type, cards included, and everything downstream relies on that single rule rather than per-type sign flips.
- **Removed means deleted. Modified means overwrite and re-sort.**
- **Unknown accounts are skipped and logged.** A transaction for an account the provider has not listed is not stored against a guess.

---

## After every sync: accounts and item state

- **Balances** come from the free cached-balance endpoint, not the paid live-balance one.
- **Item status** is read every time: last successful update, errors, institution name. Any error the provider reports moves the bank to login-required, revoked or error, so a broken bank never shows as connected.
- **Accounts the bank stops sharing** are set to closed. Their pending charges are deleted. An empty account list never closes anything: it is far more likely a transient fault than a bank closing every account at once.
- **Any change** to the bank's status or its accounts flags the snapshot, as do new or changed rows.

---

## Environments never mix

Each company has an environment, sandbox or production, and so does each item. Nothing reads one global switch. The admin function refuses to change a company's environment while it still has connections from the other one, whether live, being made, or parked for support. Webhooks for another environment's items are ignored. Test companies run in sandbox and never touch the production budget.

Recovery paths are tested with a sandbox-only tool that puts the caller's own sandbox bank into a real bank state (a login reset, a fired webhook), so Reconnect and Add accounts meet the provider's real signed webhooks. It refuses anything but a sandbox bank owned by the caller, and returns no token, account or amount.

---

## Removal

Removing a bank is support-only. It ends the connection at the provider, deletes the token, accounts and transactions, and keeps the item row as removed and the ledger row, which still counts. It refuses when the stored token cannot be decrypted, which would usually mean the wrong key is deployed, unless forced. There is no owner-facing remove button, because a removed connection's slot is gone for good.

Account deletion ends each connection at the provider, and the login is deleted only after the provider confirms.

---

## What the tests pin

| Layer | Covered |
|---|---|
| Unit | JWT verification (valid, wrong algorithm, old issue time, hash mismatch) with a generated ES256 key; AES-GCM round trip; page upsert plus cursor compare-and-swap; time-bucket dedupe |
| Sandbox, end to end | The dynamic test user synced five times by the provider's own signed webhooks with nobody opening the view, each one flagging the snapshot; pending rows replaced on posting with no stale pending rows left; a login reset then Reconnect returning the same item to active; a duplicate refused before exchange |
| Unit only | States the sandbox cannot simulate, such as a repaired login or a generic error |

The last row is the honest one: some recovery paths have only run against fixtures, and the test record says so.
