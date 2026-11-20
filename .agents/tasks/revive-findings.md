# OctoFarm revive: findings (investigation only, no source/config edited)

Environment: Windows, Node v24.16.0, npm 12.2.3. OctoFarm 1.7.3. Baseline of `git status --porcelain` is in `revive-baseline.txt` (55 modified files under server/assets from an earlier build, plus `.agents/`).

## Layout and run model
- Root `package.json`: `build` = `cd client && npm install && npm run build`. `start` = `cd server && npm run start`, which runs `prestart: npm ci` then `pm2 start ecosystem.config.js`. So `npm start` needs pm2 (not checked/installed) and a working `npm ci`.
- Server entry: `server/app.js`. Direct run: `cd server && node app.js` (or `npm run dev` = nodemon, nodemon is a devDependency of server).
- Client: webpack 5 (`client/`), output goes to `server/assets/` (versioned `*.1.7.3.*` files, tracked in git).
- Root and server `engines`: node >= 14, npm >= 6.10. No `.nvmrc`.

## Config the app needs
- README: `.env` in repo root (`OctoFarm/.env`, i.e. `d:\Projects\OctoFarm\OctoFarm\.env`):
  ```
  NODE_ENV=development
  MONGO=mongodb://127.0.0.1:27017/octofarm
  OCTOFARM_PORT=4000
  ```
- `MONGO` is mandatory. `OCTOFARM_PORT` selects the port (use 4000 or similar, not 8080/8081). `SUPER_SECRET_KEY` is auto-generated and appended to `.env` on first run.
- First run auto-creates (all gitignored): `.env` (containing only `SUPER_SECRET_KEY=...`, because I passed the other values via process env), `images/bg.jpg`, `logs/`. They now exist. `.env` should be completed with the three lines above.

## MongoDB status
- Not available: no `mongod`, `mongosh` or `docker` on PATH, no MongoDB service. Nothing listens on 127.0.0.1:27017.
- Required: MongoDB reachable at `MONGO` (mongoose 6, so MongoDB server 3.6 to 7.x is fine). Options: install MongoDB Community as a Windows service, or use a MongoDB Atlas connection string. Docker is not installed, so the docker route needs Docker Desktop first.

## Failures observed (real output)

### 1. `npm ci` in server fails (npm 12 strict lockfile check)
```
npm error code EUSAGE
npm error `npm ci` can only install packages when your package.json and package-lock.json are in sync. Please update your lock file with `npm install` before continuing.
npm error Missing: fsevents@2.3.3 from lock file
```
This breaks `npm run install-server`, `npm run setup-dev` and `npm start` (prestart runs `npm ci`).
Root cause: the lockfile lacks the optional macOS-only `fsevents@2.3.3` entry (nodemon/chokidar), which newer npm treats as out of sync.
Smallest fix: run `npm install` in `server/` and commit the regenerated `server/package-lock.json`. I did this; it added 22 lines (the fsevents entries). No major upgrade needed.
`npm install` side effect: modifies tracked `server/package-lock.json` (now shows ` M`).

### 2. `sharp@0.30.7` has no native binary
npm 12 blocks install scripts that are not allow-listed:
```
npm warn install-scripts 4 packages had install scripts blocked because they are not covered by allowScripts:
  bufferutil@4.0.6, sharp@0.30.7, storage-engine@3.0.7, utf-8-validate@5.0.9
```
Then `require('sharp')`:
```
Something went wrong installing the "sharp" module
Cannot find module '../build/Release/sharp-win32-x64.node'
```
Impact: none found at present. grep shows no `sharp` usage in server source (declared dependency only), so app startup is unaffected. `bufferutil` loads fine (prebuilt).
Candidate fix: either leave as is, remove the unused `sharp` dependency, or approve scripts (`npm install-scripts approve sharp`). Bumping sharp would be a minor-version change (0.30 to 0.33+, bundled prebuilds). Not needed to run.

### 3. MongoDB connection refused (startup blocker)
Started with `NODE_ENV=development MONGO=mongodb://127.0.0.1:27017/octofarm OCTOFARM_PORT=4000 node app.js` (killed afterwards):
```
ERROR MESSAGE: MongooseServerSelectionError: connect ECONNREFUSED 127.0.0.1:27017
    at Connection.openUri (...\mongoose\lib\connection.js:824:32)
    ... at Object.<anonymous> (...\server\app.js:45:6)
(node:75156) [DEP0060] DeprecationWarning: The `util._extend` API is deprecated.
```
Env, logger and config setup before this point all work on Node 24. Root cause: no MongoDB. Fix: install/start MongoDB (or Atlas URL in `MONGO`). Whether the app is otherwise healthy past the DB connect (migrations, routes, pm2) is untested.

### 4. Non-blocking warnings
- Client `npm run build` (webpack 5.74) succeeds: "compiled with 3 warnings" (asset/entrypoint size and performance recommendations only). No OpenSSL problem (webpack 5 here, so `--openssl-legacy-provider` is not needed).
- Client `npm install` blocks `core-js@3.25.3` postinstall (cosmetic message only). Client lockfile unchanged.
- `npm audit`: server 74 vulnerabilities (8 critical), client 30. Fixing them would involve major upgrades (mongoose, express-related, request is deprecated, etc.). Out of scope for getting it running.
- `DEP0060 util._extend` deprecation warning (from a dependency), harmless on Node 24.

## Tracked-file changes caused by my runs (vs baseline)
- `server/package-lock.json`: modified by `npm install` (fsevents entries). Keep if committing fix #1.
- Client build regenerated `server/assets/*1.7.3*` files, which were already modified in the baseline. The count stayed at 55 (same set); contents were rewritten again. Decide whether to keep or `git checkout server/assets`.
- Nothing else tracked changed. `server/node_modules` is new (gitignored).

## Recommended plan
1. Install MongoDB (service) or choose an Atlas URI.
2. Create `OctoFarm/.env` with NODE_ENV, MONGO, OCTOFARM_PORT=4000 (plus the generated SUPER_SECRET_KEY).
3. Commit the regenerated `server/package-lock.json` (npm install in server).
4. Optionally add `.nvmrc`/engines note for Node 24 and decide on sharp (leave or remove).
5. Run with `cd server && node app.js` (or `npm run dev`). `npm start` needs pm2 installed globally and `prestart npm ci` to pass (fixed by step 3). Verify the app loads at http://localhost:4000.

No fix requires a major upgrade to get the app running. A major upgrade would only be needed to clear audit findings.
