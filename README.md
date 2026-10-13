# Bank Feeds Inside a Live Multi-Tenant App

Engineering notes from building **a financial module inside an existing, live field-service SaaS app**: business bank and card feeds through Plaid, revenue from a field-service CRM, a deterministic sorter and calculation engine, a self-service setup flow and an owner-only view, all built into a codebase and pipeline I did not own while the host kept shipping.

The bank API was the easy part. **The real problem is adding regulated bank data to a system whose tenancy model, deploy pipeline and integrations were designed before anyone planned to hold it, without pausing that system or breaking it.** So the module brings its own tenancy predicate, caller checks, encryption and deploy verification.

Design notes and reasoning only. No client name, no host app name, no source code, no credentials, no endpoints, no table or function names, no customer or bank data, no business figures, and none of the business rules that decide what counts as revenue or cost. Those rules are the client's product and stay theirs.

**Deep dives:** [Bank feed ingestion](docs/01-bank-feed-ingestion.md) · [Shipping inside a live host app](docs/02-shipping-inside-a-live-host.md)

---

## Contents

1. [Problem shape](#1-problem-shape)
2. [Why the host's defaults were not enough](#2-why-the-hosts-defaults-were-not-enough)
3. [Architecture](#3-architecture)
4. [Tenancy: one owner predicate](#4-tenancy-one-owner-predicate)
5. [Bank tokens and a budget you cannot refund](#5-bank-tokens-and-a-budget-you-cannot-refund)
6. [The CRM feed](#6-the-crm-feed)
7. [The sorter: deterministic, total, property-tested](#7-the-sorter-deterministic-total-property-tested)
8. [The calculation engine](#8-the-calculation-engine)
9. [Self-service setup](#9-self-service-setup)
10. [No bank data in AI tools](#10-no-bank-data-in-ai-tools)
11. [Failure modes](#11-failure-modes)
12. [What I'd do differently](#12-what-id-do-differently)

---

## 1. Problem shape

The host app is multi-tenant: a company is an owner account with team members and subcontractors attached, and the owner's profile id is the company. The module gives the business owner one view built from two feeds:

| Feed | Shape | What makes it awkward |
|---|---|---|
| Bank and card accounts, through Plaid | Link, cursor sync, signed webhooks | Regulated data, long-lived tokens, a connection budget where every exchange is permanent |
| A field-service CRM | OAuth per company, HMAC-signed webhooks | A rotating refresh token, an uncapped history, and an integration the host already runs, which the module has to share without inheriting its assumptions |

The constraints: **owner only** (not team members, subcontractors or the host's admins); **self-service** (no keys or provider account needed from a company); **no bank rows or payee names ever reach an AI tool**; and **do not break the host**, whose AI-assisted builder pushed to the same branch several times an hour while in use.

---

## 2. Why the host's defaults were not enough

The obvious build copies the host's row policies, reuses its CRM integration, trusts the gateway, and ships by merging. A read-only pass over the code showed that each of those defaults fits the data the host was built for, and none of them fits bank data:

| Obvious move | Why it does not fit bank data | Instead |
|---|---|---|
| Copy the existing row policies | They were written for data an admin role is meant to see; on bank data, one admin login would read every bank feed | An owner predicate with no admin path (§4) |
| Rely on gateway-level JWT verification | A gateway setting is configuration, and configuration can be missing or drift | Every function checks its caller |
| Share the CRM webhook handler as it stands | With bank data beside it, every event must be tied to exactly one company, every signature compared in constant time and every lookup scoped to one owner | Route by account id, constant-time HMAC, owner-scoped lookups (§6) |
| Refresh the CRM token wherever it is used | The refresh token rotates, so two concurrent refreshes race and one can invalidate the other's token | One locked refresh path (§6) |
| Merge to ship | A merge deploys nothing | Each artefact verified live ([deep dive](docs/02-shipping-inside-a-live-host.md)) |

The identity layer is part of the attack surface too, so every path that writes identity rows is reviewed before the first bank row is stored: a profile self-insert must not be able to choose its own role or parent, an invite must not move an existing account into another team unless that person signs in, and a function the module does not need but that can write identity rows is retired behind a 410 Gone stub. **Isolation on new tables is only as strong as the identity rows underneath them**: where "company" means "a profile with no parent", whatever can rewrite a parent can rewrite tenancy.

---

## 3. Architecture

```
 Owner's browser             Host backend: Postgres + edge functions        External
 setup flow  -- invoke -->   link token, exchange, setup functions -------> Plaid API
                             bank webhook receiver <----------------------- Plaid webhooks
                             bank sync: page upserts, cursor CAS
 owner view  -- invoke -->   snapshot function -> SQL aggregates
 drill-downs -- RLS read ->  the owner's own rows, paged
 CRM connect -- invoke -->   OAuth start -> callback
                             CRM webhook receiver <------------------------ CRM webhooks
                             CRM import: history, incremental -----------> CRM API

 Shared pure TypeScript, run by edge functions and tests: sorter, engine, dates
```

- **Server-side only.** The browser gets a computed snapshot and, for drill-downs, its own rows through row-level security. Every write goes through a function that checks the caller and validates the input.
- **One core, tested where it runs.** The sorter and engine run unchanged in the Deno functions and in vitest.
- **Totals in SQL**, so the API's 1,000-row cap never truncates a total. A fixture of more than 1,000 rows guards it.

---

## 4. Tenancy: one owner predicate

Every finance policy compares the row's owner with one security-definer SQL function. It returns the caller's id only if they are signed in, have no parent company (so team members get null), are a company account rather than a subcontractor, and have the module's switch on.

- **No admin path, ever.** The predicate never reads roles, so a host admin login gets no finance rows. The admin function returns no money fields.
- **The owner never comes from the browser.** Functions resolve it from the caller's session, not the request body. Another company's id gets a 404.
- **The switch is the kill switch.** Off means null: access ends at once, the data stays.
- **Sealed tables.** Token ciphertext, the connection ledger, CRM sync state and the webhook inbox have row-level security on and zero policies. The linter flags that; it is deliberate.

A migration is not done until a read-back confirms each of these is false:

```
browser role can select from a sealed table
browser role can execute a write function
a finance policy references the host's admin check
browser role holds insert, update or delete on a finance table
```

An isolation script then runs as anonymous, owner A, owner B and team member A1: A sees its own rows, the others none; sealed tables deny; writes fail; a foreign id gets a 404. It went from 114 of 114 to 121 of 121 after a later migration added a table. **The positive control is the check people skip**: if nobody sees anything, a broken system passes.

---

## 5. Bank tokens and a budget you cannot refund

Access tokens are encrypted in the application with AES-256-GCM, one key per environment: 32 random bytes held as a platform secret. A vault extension was available, but app-level encryption means SQL editor access alone cannot read a token: the ciphertext is in the database, the key in the function runtime.

That makes **the key the most important secret in the system**: losing it loses every connection. So linking first checks that the key opens a connection already stored for that environment; a missing, wrong or swapped key refuses before the bank window opens. A health check reports secret names and the canary, never values. The production key has an escrowed backup.

The provider's current plan allows a fixed number of connections. One is used the moment a public token is exchanged, and removing it never gives it back. So the exchange:

1. refuses duplicate accounts, then reserves under an advisory lock, refusing a bank already linked or being linked, a company over its limit, or a provider account already one short of the plan's limit (one stays spare);
2. sets the ledger row to "exchanging" before calling, counted and never expiring;
3. calls once, with no automatic retry; an unclear outcome goes to support;
4. saves connection, ciphertext and ledger state in one transaction, or parks the ciphertext for support.

**When an operation is irreversible and metered, count it before you attempt it and never retry it automatically.** If the first call quietly succeeded, the connection is already spent; support can reconcile that, a retry loop cannot.

---

## 6. The CRM feed

- **Webhooks.** Constant-time HMAC-SHA256 over the raw body; a bad signature gets a 401. Reply 200 at once, process in the background, deduplicate. A failed step retries after 2 s, 10 s and 30 s, then releases its dedupe key so the provider's redelivery is processed.
- **Route by CRM account id** over live connections only; unmatched events are dropped, never sent to another company's token. So the OAuth callback never saves a connection without its account id, and refuses an account another live company holds.
- **One refresh path.** Refresh within 5 minutes of expiry under a 30-second lock; a waiter sleeps 1.5 s and reads again; the rotated pair is saved before returning; a 400 or 401 triggers a re-read before asking the owner to reconnect.
- **Resumable history.** 25 jobs a page, cursor saved per page; each call stops starting pages after 45 s and the caller repeats until done.

The module pins its own CRM API version. On a test account, a job closed in the CRM reached the module 3 s later with nobody opening the view.

---

## 7. The sorter: deterministic, total, property-tested

Each transaction gets one treatment: counted in a category, a refund, or excluded with a reason. The sorter is a pure function with three properties:

| Property | Rules out |
|---|---|
| **Total:** never returns "unsorted" | A transaction missing from every figure |
| **Deterministic:** any input order, same result | Output that depends on database order |
| **Bounded exclusion:** business outflows leave totals only via an enumerated list of reasons | Rules and pairings that quietly drop a real cost |

The third is the property test: examples cover the cases I thought of, the property generates the rest. The dangerous error is a cost that disappears, because the figure improves and nobody asks why.

Each transaction walks a fixed ladder: guards, explicit rules, pairing of matching transactions, inflow and outflow handling, the provider's taxonomy (127 fine-grained codes, generated from its published file), then a default. Rules have a total order, so two never tie.

One sort per company at a time, and **the lock's expiry doubles as the holder's token**, so a sort that outlived its lock cannot release another sort's lock. Known limits are written in the header: greedy rather than maximum pairing, and about 1 s worst case for 8,000 same-amount rows in one window. Four independent review rounds came before real data.

---

## 8. The calculation engine

The engine is the second pure function, from window totals to a snapshot. It never throws on empty data, never returns NaN or Infinity, rounds to cents only at the output, and uses local dates in the company's time zone. Tests pin an early-morning UTC instant to the previous local day and cross a daylight-saving change, because a transaction on the wrong day moves windows silently.

- **Types enforce the caller contract.** Easy-to-forget values, such as when an account's history starts, are required fields. A missing window is a named state, not a zero.
- **Partial is a label.** A window starting before an account's history is reported as partial, with the reason.
- **Never reassuring on bad inputs.** A stale, broken or missing input always surfaces as a warning, never as a clean result.
- **Degraded modes say so.** When an input behind a result fails or is stale, the result names the input and the reason.

Snapshots carry engine and sorter versions and are flagged as changed by syncs, webhooks and owner actions. On open: recompute if changed or older than 10 minutes, refresh a bank older than 6 hours and the CRM older than 30 minutes in the background, and return at once. No nightly job; events plus on-open refresh were enough.

---

## 9. Self-service setup

One CRM app registration and one provider account serve every company; each owner connects their own CRM login and links their own bank. The only non-owner step is a per-company switch protecting the connection budget.

Setup is six resumable steps: consent (recorded with a version), connect the CRM (required, or revenue reads as zero), link accounts, choose business accounts, settings, finish. Each failure has its own message and fix: CRM account used by another company, owner-only action, bank already linked so use Reconnect, link in a browser.

- **No default for a decision only the owner can make.**
- **Data minimisation at finish.** Rows on non-business accounts are deleted, and every later sync deletes any that appear.
- **Self-service afterwards.** Reconnect uses update mode, so no new connection.
- **In-app browsers.** An OAuth bank cannot finish there without a redirect URI. Register it with the provider before setting the secret, or every link token fails.

The acceptance test: a sandbox test owner went from switch-on to a first snapshot with no outside help.

---

## 10. No bank data in AI tools

Bank rows never reach the host's AI features or its automation workflows, and payee names never reach any AI tool used to build, deploy or support the module.

- **Guard tests** fail if a bank-touching function references the AI gateway or automation tooling, if a response can carry an access token, or if module UI code logs to the console.
- **Logs** carry ids and error codes only.
- **Human review** uses read-only query packs. Aggregates go into the evidence log; results with payee names stay with the person running them and never go into any AI tool.

Account deletion removes each connection at the provider before the login goes. **A data rule that a test enforces is a property of the build rather than a promise.**

---

## 11. Failure modes

| Failure | Symptom | Mitigation |
|---|---|---|
| Admin role reads finance rows | One login sees every bank feed | Predicate ignores roles; read-back |
| Gateway setting missing or drifted | Unauthenticated call reaches a function | Own caller checks; post-deploy probe must get the module's own 401 |
| Forged or replayed bank webhook | Fake sync triggers | Signed JWT, body hash, 5-minute window; cursor CAS |
| Token key missing or swapped | Every connection unreadable | Canary refuses to link; escrowed backup |
| Unclear exchange outcome | Connection spent, not stored | Counted before the call; no retry; ciphertext parked |
| CRM event for unknown account | Data under the wrong company | Route by account id; drop unrouted |
| Pinned CRM version rejects a query | A figure silently loses an input | Labelled fallback that names the missing input; queries checked against the pinned version on a CRM test account |

**Treat the gateway as a convenience, not a control.** Every module function authenticates its own caller, so no module function depends on gateway configuration.

---

## 12. What I'd do differently

**Write the deploy order down before the first deploy.** Functions that tolerate the old schema, then the frontend release, then the migration and its read-back. The exception is a migration that only widens what is allowed: a new exclusion reason needs its migration before the functions that produce it, or every bank update would fail against the old constraint.

**Tell deploy tooling what not to fix, from the first instruction.** "Change no other file" is not enough for an AI-assisted builder: asked to apply a migration, it can treat a type error as part of the job and edit files to fix it. Instructions should forbid fixing type or build errors as well as changing files, and every result should be diffed.

**Get a CRM test account on day one.** Without one, the only place to verify a query against a pinned API version is read-only on real data, and a filter the pinned version rejects surfaces there instead of in a test. A labelled fallback keeps the figure honest, but the check belongs on a test account, before any real data.

**Plan sandbox fixtures against the guards.** The default sandbox user has identical accounts at every test bank, so the duplicate guard refuses a second; that needs a custom user.

---

## Stack

Plaid · field-service CRM API (OAuth2, HMAC webhooks) · Supabase (Postgres, row-level security, Deno edge functions) · AES-256-GCM · WebCrypto · TypeScript · React · vitest with property-based tests · PGlite

---

*Engineering notes only, published with the client's permission. Contains no source code, credentials, API keys, tokens, endpoints, table or function names, customer data, bank data, business figures, business rules, commercial terms, or client-identifying information.*
