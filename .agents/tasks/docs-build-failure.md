# Docs site `npm run build` failure

## Summary
The docs site is not in the OctoFarm repo. It is the sibling repo `d:\Projects\OctoFarm\OctoFarm-Documentation` (VuePress 1.x, webpack 4). `npm run build` there fails on Node v24.16.0 with `Error: error:0308010C:digital envelope routines::unsupported`. Webpack 4 hashes with MD4, which OpenSSL 3 (Node 17+) disables. Setting `NODE_OPTIONS=--openssl-legacy-provider` makes the build pass. I tested this.

## Evidence
- `d:\Projects\OctoFarm\OctoFarm\package.json`: `build` is `cd client && npm install && npm run build`. That builds the app's webpack 5 client bundle, not the docs. It succeeded (3 asset-size warnings only). If the user ran `npm run build` in the main repo, it is not failing.
- `OctoFarm-Documentation\package.json`: scripts `dev` = `vuepress dev src`, `build` = `vuepress build src`. Dependencies: `vuepress ^1.5.3`, `@vuepress/plugin-pwa`, `vuepress-plugin-code-copy`, `vuepress-plugin-plausible`. There is no `engines` field and no `.nvmrc`.
- Command: `npm run build` in `d:\Projects\OctoFarm\OctoFarm-Documentation`. Output during "Compiling Client/Server":
  ```
  Error: error:0308010C:digital envelope routines::unsupported
      at new Hash (node:internal/crypto/hash:103:19)
      at Object.createHash (node:crypto:146:10)
      at ...node_modules\webpack\lib\util\createHash.js:135:53)
      at NormalModule._initBuildHash (...webpack\lib\NormalModule.js:417:16)
  opensslErrorStack: ['error:03000086:digital envelope routines::initialization error', ...
  ```
- With `$env:NODE_OPTIONS="--openssl-legacy-provider"`, the build output ends with: Server and Client compiled successfully, then `success Generated static files in src\.vuepress\dist`.

## Root cause
VuePress 1.x bundles webpack 4, which calls `crypto.createHash('md4')`. Node 17+ uses OpenSSL 3, which does not provide MD4 by default. The local Node is v24.16.0.

## Recommended fix (nothing applied)
Pick one:
1. Per session (PowerShell): `$env:NODE_OPTIONS="--openssl-legacy-provider"; npm run build`. The same variable works for `npm run dev`.
2. Make it permanent in `OctoFarm-Documentation\package.json`. `cross-env` is not a dependency, so add it, or use the `set` form below (cmd only, since npm runs scripts through cmd on Windows):
   - `"dev": "set NODE_OPTIONS=--openssl-legacy-provider && vuepress dev src"`
   - `"build": "set NODE_OPTIONS=--openssl-legacy-provider && vuepress build src"`
   
   This breaks on Linux CI. Prefer `npm i -D cross-env` with `"build": "cross-env NODE_OPTIONS=--openssl-legacy-provider vuepress build src"`.
3. Use Node 16 (for example `nvm use 16`). This needs no config change. Check `.github\workflows\deploy.yml` for the Node version CI uses, and keep local and CI consistent.

Option 3 is cleanest. Option 2 with cross-env is best if you want it to work on any Node version.

## Side effects of my investigation (flagging for you)
- Running the main repo's `npm run build` regenerated tracked bundles under `OctoFarm\server\assets\` (for example `css/dashboard.runner.1.7.3.css` and `js/bootstrap.1.7.3.min.js`). They now show as modified in `git status`. Revert with `git -C d:\Projects\OctoFarm\OctoFarm checkout -- server/assets` if you don't want them.
- `OctoFarm-Documentation\package-lock.json` already showed a `0.0.1` to `0.0.2` version bump. This is only a lockfile version sync, not caused by the failure.
- The `Language does not exist: mnavarro@paintedgreen:~>` and browserslist messages in the docs build are harmless warnings.
