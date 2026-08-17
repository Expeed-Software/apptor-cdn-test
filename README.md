# apptor-cdn-test

A stand-in for the CDN a team publishes Apptor apps to. Published at
<https://expeed-software.github.io/apptor-cdn-test/> (GitHub Pages, from `main` root), which is what
the Runtime Server and the Admin Portal's registration wizard read.

To serve it locally instead:

```bash
npx http-server . -p 8082 --cors
```

> A `localhost` address is rejected by the server's SSRF guard before the first fetch, so local
> serving needs `RUNTIME_CDN_SSRF_GUARD_ENABLED=false` on the Runtime Server. The published Pages
> origin needs no such thing — prefer it.

## What is published here

| Directory | Published key | Display name | Versions |
|---|---|---|---|
| `app3/` | `app3` | App 3 | 1.0.0 |
| `attendance-manage/` | `am` | Attendance Manager | 1.0.0, 1.1.0 |
| `crm/` | `crm` | CRM | 1.0.0 – 1.2.0 |
| `customer-portal/` | `customer-portal` | Customer Portal | 1.0.0 – 1.5.0 |
| `ecommerce/` | `ecom` | E-commerce | 1.0.0 – 1.2.0 |
| `field-service/` | `field-service` | Field Service | 1.0.0 |

**The directory name is not the key.** `attendance-manage/` publishes `am` and `ecommerce/`
publishes `ecom`. The Runtime Server binds an application's key from `runtime.json`'s `app.key`, not
from the folder and not from `releases.json`'s top-level `app` (which is only a display label). Two
apps here differ on purpose, so anything reading the wrong one is caught immediately.

## Layout

```
index.json                                 <- the index of what is published here
<app>/releases.json                        <- that app's versions, each with its own path
<app>/releases/<x.y.z>/runtime.json        <- per-version manifest (key, name, description, auth, integrations)
<app>/releases/<x.y.z>/app/app.json        <- the app definition itself
<app>/releases/<x.y.z>/api/<collection>.json
```

Nothing derives a version folder's name: every consumer reads `path` out of `releases.json`, which is
why dropping the old `v` prefix needed no code change anywhere.

### `index.json` — why it exists

Static hosting cannot be listed. nginx defaults to `autoindex off`, and S3 static hosting,
CloudFront, GitHub Pages, Netlify, Cloudflare Pages and Azure SWA offer no directory listing at all —
a bare Pages root 404s. So the publisher declares what it published, exactly as Helm does with
`index.yaml`, APT with `Packages` and Maven with `maven-metadata.xml`.

**Without this file the wizard cannot list anything.** The server can still register one app from its
own address, but pointed at the root it answers `422 source_not_enumerable` and the pick list is
empty. Regenerate after adding or removing an app — it is not maintained by hand:

```bash
node ../apptor-admin-portal/tools/generate-apps-index.mjs .
```

It is idempotent: re-running with no new apps rewrites an identical file. `--check` exits 1 when the
file on disk is stale, for CI.

### `runtime.json` — `app.displayName` and `app.description`

```json
{
  "schemaVersion": 1,
  "app": {
    "key": "am",
    "displayName": "Attendance Manager",
    "description": "Shift attendance, leave requests and timesheet approvals."
  },
  "version": "1.1.0"
}
```

Both are **optional**, and both are read from the **newest** version — a name is a label and should
read as whatever was published last, while the key stays bound to the earliest version because it is
identity.

- `displayName` absent → the console derives a name from the key. Derivation cannot do better than
  `AM` for `am`, which is exactly why publishing the real name matters.
- `description` absent → the console shows none. Nothing derives it: inventing a sentence about
  someone else's application is worse than showing nothing.

An operator can always rename an application in the console afterwards, and that stored name wins.
