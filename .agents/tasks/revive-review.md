# OctoFarm revived on Node 24: only the lockfile changed

The only source-tracked change is `server/package-lock.json` (+22 lines, adding the optional macOS-only `fsevents@2.3.3` entries). It fixes `npm ci` failing on npm 12 and no dependency versions change. The client build is reported as passing and the server boots to the MongoDB connection, which fails because MongoDB is not installed.

Watch for: HTTP serving was not demonstrated (confirmed blocked only by missing MongoDB). `sharp` has no native binary but is unused by server source (likely).

**Verdict**: APPROVED

## High-level view

The fix is one regenerated lockfile. It is the smallest change that makes `npm ci`, and therefore `npm start`, work on npm 12. No source, dependency bumps, or commits were made, and `.env` (gitignored) was completed with the README values and port 4000, not 8080/8081.

The 55 modified `server/assets/*1.7.3*` files match the baseline exactly. They are client build output, they predate this task, and the report lists them accurately with a revert command. The report also lists `package-lock.json` as the one intended change.

The app cannot serve HTTP here. `server/app.js` calls `mongoose.connect` and only calls `app.listen` after it resolves, so the ECONNREFUSED to 127.0.0.1:27017 stops it before listening. Port 27017 is confirmed unused on this machine. The report explains this and gives three ways to provide MongoDB.

<details>
<summary>Issues (2)</summary>

1. **No raw boot log saved** (possible, non-blocking) — the "boots to Mongo ECONNREFUSED" claim rests on the report text. Attach the log next time.
2. **Docs repo has unrelated modifications** (confirmed, out of scope) — `OctoFarm-Documentation` shows `package.json` and `package-lock.json` modified. This is attributed to the earlier docs-build fix, not this task. Confirm that is intended.

</details>

<details><summary>Details</summary>

## Lockfile fix scope

The diff adds `fsevents` under both `packages` and legacy `dependencies`, marked optional and darwin-only. It does nothing on Windows and matches the root cause the findings note describes.

## Build-only modified files

Status shows 55 asset files plus the lockfile, which equals the baseline plus the lockfile. LF/CRLF warnings appear on the assets, so some of the churn may be line endings, not content. Reverting with `git checkout -- server/assets` is safe if unwanted.

## Blocked on MongoDB

A narrow check of `app.js` confirmed the listen-after-connect ordering. No other blocker was identified.

</details>

<details><summary>File map</summary>

- `server/package-lock.json`: adds optional fsevents entries.
- `server/assets/**/*1.7.3*` (55 files): client build output, pre-existing in baseline.
- `.agents/`: untracked task notes.
- `.env` (gitignored): MONGO, port, NODE_ENV.

Full diff: `git -C d:\Projects\OctoFarm\OctoFarm diff`

</details>
