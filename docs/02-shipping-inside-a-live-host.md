# Deep dive: shipping inside a live host app

[Back to README](../README.md)

---

The module went into an app that was already in production, on a branch the host's AI-assisted builder pushed to several times an hour while in use, through a platform where merging code does not deploy it. Most of the project's risk lived here rather than in the bank API, and most of the rules below exist because the host could change underneath the module at any moment.

---

## The host as a moving target

| Risk | Rule |
|---|---|
| The host's builder pushes around a merge | Module work on its own branches cut from the shared branch; rebase daily and again before each merge, and diff what the shared branch changed in module files |
| A shared host file conflicts | Edits to host files kept as small as possible; the navigation entry was a one-line diff |
| A generated file is overwritten | The generated database types file is never hand-edited; the platform regenerates it |
| A restore rolls module files back | A daily check that the module's paths still exist on the shared branch |
| A frontend release ships unfinished host work | Every release waits for the client's OK, and module UI sits behind the per-company switch, so an owner who is not switched on sees nothing |

Nothing is pushed directly to the shared branch or to main. Every change arrives as a pull request.

---

## A merge is not a deploy

On this platform functions, gateway settings, secrets, migrations and the frontend each have their own release step, and none of them happens on merge. So each artefact has its own live check:

| Artefact | Released by | Verified live by |
|---|---|---|
| Edge functions | An explicit deploy naming the exact functions | A plain GET returns the module's own 405; the platform's 404 means the function is not there |
| Gateway config | Merged, then applied through the platform | An unauthenticated POST to the webhook gets the module's own 401, not the gateway's; probed again after later merges |
| Secrets | Pasted by hand | The health check lists names and the key canary, never values |
| Migrations | The reviewed reference SQL, applied through the platform | The SQL shown for approval is diffed against the reference first; the read-back after |
| Frontend | A release, after the client's OK | Fetch the live index, every script it loads and every chunk those load, and search for a string only the change adds |

The frontend check exists because there is no fixed build identifier to compare. New wording that only the merged change introduces is the most reliable proof that the live bundle carries it.

**Which functions to deploy is a small graph problem.** A function needs a redeploy if its own folder changed, or if it imports a changed shared file directly or through another shared file. So the search covers the whole functions folder rather than only entry files, and repeats for every shared file it finds; a shared module imported by just one other shared module is the kind that gets missed. A change to type-only shared code needs no redeploy, because every function imports it as types. Shared files that the web app also bundles need a frontend release as well, or the browser and the functions disagree about the same rule.

---

## Migrations that survive being applied twice

The platform can re-emit migrations as its own files, so every migration is written to be safe to apply twice. Each one has a reviewed reference file, a down file and a read-back query. Before approval, the SQL the platform shows is compared with the reference; afterwards, the file the platform wrote is diffed with it. Every recorded comparison matched: one byte for byte, the rest apart from the final newline and, once, line endings from a Windows checkout.

Before going near production, each of the first six migrations ran in a local harness: PGlite on Postgres 17, with the platform's roles and default privileges reproduced. The successive migrations passed 113 (the first two together), 22, 75, 89 and 85 checks.

**Idempotent is not the same as order-independent.** Applying the same file twice in a row is safe. Re-applying an older file after a newer one puts back the older version of every function the newer one replaced. So after any re-apply, every later file is applied again in order, each read-back is run again, and any per-company setting support had changed is set again from the log, which is why every such change is logged with its date.

Rollback has a cliff in it. Down files exist, but the core one refuses once a production bank has been linked, because the ledger of spent connections must never be dropped. After that point, rolling back means switching companies off and reverting code.

---

## Deploy order

The order that works:

1. Edge functions written to tolerate the old schema, for example skipping a new input until its table exists.
2. The frontend release that labels the new figures correctly, checked on the live bundle.
3. Only then a migration that changes what the figures mean, its read-back, and a check as a test company.

In the other order, the live view would show new totals under old labels, with drill-downs that do not add up, for as long as the release lagged.

The exception is a migration that only widens what is allowed, such as a new exclusion reason. That one goes first, because functions that produce the new value fail against the old constraint, and here that would mean every bank update fails.

---

## Deploying through an AI-assisted builder

The host is developed with an AI-assisted builder that commits to the shared branch and applies migrations. That made it the deploy path, and a deploy path that edits files on its own needs tight instructions.

- **Closed instructions.** Each one names the exact functions or SQL, forbids creating, editing or deleting any other file or touching settings or secrets, and ends with a show-back step: what was deployed and which files changed, which must be none.
- **Forbid fixing, not only changing.** An AI-assisted builder told to apply a migration and change nothing else can still treat a type error as part of the job and edit files to fix it. So every instruction also says not to fix type or build errors, and the result is diffed against the repository.
- **Leave the warnings.** Its security linter flags the sealed tables. It is told not to "fix" them; the zero-policy shape is the point.
- **No side-effect commits.** Builder modes that commit files to the shared branch as a side effect are avoided for checks. Read-only checks are plain SQL that a person runs in the editor.

---

## Not breaking the host's features

- **The host's own webhook logic** for its existing features keeps its behaviour. The shared handler matches each event to exactly one company, and the module's branch in it runs only for companies whose switch is on.
- **Routing the host's functions through the one locked refresh path** was mechanical: the same calls, with one place that refreshes the token.
- **The host's import button** keeps its own import. The module's history import runs from setup and when the view opens, because rewiring the button would have removed an import other companies may use.
- **API versions.** The module pins its own CRM API version and does not change the host's.
- **Regression watch.** The checks defined for a deploy are smoke tests on signup, invites, the push self-test and the CRM page, plus 24 hours of webhook events with no 5xx. They are run again when a change touches those paths.

Before any bank data, the hardening passed 17 of 17 live security tests. A team member's login was later checked on the live site, 13 of 13: no menu entry, an owner-only notice on every module page, no module data, and the admin page refused.

---

## The gateway principle, generalised

"Every function checks its own caller" sounds like belt and braces. It is the only arrangement in which a missing or drifted gateway setting cannot expose a function, so the module never relied on the gateway.

Each module function names its accepted callers in its own code:

| Caller | Check |
|---|---|
| Signed-in owner | Session resolved by the platform client, then the owner predicate must be non-null |
| Service | The bearer must equal the service key |
| Admin | The caller's role is checked with the service role |

Three functions run with no sign-in by design: the two webhook receivers and the OAuth callback. They authenticate by signature or by a single-use state row instead, and each is listed explicitly in the gateway config so its behaviour is deliberate rather than a default.

**In a host you do not own, every assumption about the platform is a test you have not written yet.**
