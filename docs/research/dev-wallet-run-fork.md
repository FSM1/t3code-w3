# Running the forked server as a remote environment

Research note for [FSM1/t3code#3](https://github.com/FSM1/t3code/issues/3) (dev-wallet map
[#1](https://github.com/FSM1/t3code/issues/1)). Read-only findings against the code and docs at
`fork/dev-wallet` (`5f7294d452`), 2026-10-11. Paths are repo-relative; line numbers are from that commit.

## Correction to the map's premise

The map assumes that a forked server added to the stock desktop app "runs tabs headless". That is
true for **server-runtime** tabs only. In the desktop app, a tab the *user* opens on a non-primary
environment is created with its `runtime` omitted, and the desktop then hosts the page in its own
Electron browser on the client machine. `prepareContext` never runs for such a tab.

- `apps/web/src/browser/previewRuntime.ts:20-29` (`previewRuntimeFor`): returns `undefined` when
  `isPreviewSupportedInRuntime()` (Electron with `window.desktopBridge.preview`,
  `apps/web/src/previewStateStore.ts:498-501`) and the environment is not the primary one. The doc
  comment: "A remote environment's tabs would stream, so the desktop opens them in its own browser
  instead ... The user can move a tab to the environment when only it can reach the page."
- `apps/web/src/components/preview/openPreviewSession.ts:48-57`: an `undefined` or `"desktop"`
  runtime is not sent to the server. `packages/contracts/src/preview.ts:239`: "Omit for a desktop
  tab. `server` requires the `serverBrowser` capability."
- `apps/web/src/components/preview/openDiscoveredPort.ts:21`: port-discovery opens follow the same
  rule.
- Moving a tab: `apps/web/src/components/preview/PreviewView.tsx:452-459` offers
  **Open in `<environment>`'s browser** (`alternatePreviewRuntime`, `previewRuntime.ts:35-45`). After
  that, the tab is a server-runtime tab and everything in section 3 applies. The reverse action is
  **Open on this computer**.
- Agent-opened tabs are server-runtime: `docs/user/remote-access.md:185-191` ("Agents keep using
  them while no device is connected, and `localhost` addresses reach servers on the host"); popups
  from agent tabs are reopened with `runtime: "server"` (`apps/server/src/preview/ServerBrowser.ts:1160`).
- A non-Electron client (the fork's own web UI, or app.t3.codes) has
  `isPreviewSupportedInRuntime() === false`, so every tab it opens is a server-runtime tab.

Consequence for the dev-wallet patch: with the stock desktop app, the user opens the dapp tab and
then moves it to the fork's browser (or lets the agent open it). The injection lands only on
server-runtime tabs. Only the orchestrator edits #1; this note records the finding.

## 1. Which command runs the server

**Recommendation: run the source entry directly, no build.**

```sh
cd <checkout of fork/dev-wallet>
vp i                                  # once; Node ^24.13.1 (root package.json:62-64)
node apps/server/src/bin.ts serve --base-dir ~/.t3-dev-wallet
```

- `apps/server/package.json:17-21`: the package `t3` has `"bin": {"t3": "./dist/bin.mjs"}`, and its
  own `dev` script is `node --watch src/bin.ts`, so Node runs `src/bin.ts` without a build
  (`tsconfig.base.json:7-9` sets `allowImportingTsExtensions`, `rewriteRelativeImportExtensions`,
  `erasableSyntaxOnly`; workspace packages export `.ts` sources). Build-only defines are guarded
  (`apps/server/src/cloud/publicConfig.ts:53-75`), so the `connect` subcommand is a stub from source
  unless the relay env vars are set (`publicConfig.ts:214-218`). Not needed here.
- `t3 serve` = "Run the T3 Code server without opening a browser and print headless pairing
  details" (`apps/server/src/cli/server.ts:60-70`). It forces `noBrowser` and disables
  auto-bootstrapping a project from `cwd` (`apps/server/src/cli/config.ts:352-377`).
- Built alternative, only if you also want the fork's web UI served from the same origin:
  `vp run --filter t3 build && node apps/server/dist/bin.mjs serve --base-dir ~/.t3-dev-wallet`
  (`apps/server/vite.config.ts:71-82` builds `@t3tools/web` first and `apps/server/scripts/cli.ts:93-101`
  copies `apps/web/dist` into `dist/client`; `apps/server/src/config.ts:255-274` `resolveStaticDir`
  also finds `apps/web/dist` when running from `src/`). The desktop app ships its own renderer and
  only needs the API and WebSocket routes, so the web bundle is unnecessary for this use
  (`docs/internals/remote.md:92-103`: "API traffic always goes to the environment's own URL").
  `docs/user/install.md:53-60` documents the same build-and-run path for Intel Macs.
- Do not use the dev runner for this. `vp run dev` / `dev:server` (`scripts/dev-runner.ts:74-86`)
  always set `VITE_DEV_SERVER_URL` (`dev-runner.ts:321-323`); with a dev URL the server 302s loopback
  page requests to Vite and 404s the proxied paths (`apps/server/src/http.ts:546-579`), and
  `dev:server` runs no Vite. `vp run start` runs bare `t3`, which opens a browser and
  auto-bootstraps `cwd` as a project.

### Home directory

The server CLI flag is **`--base-dir`**; `--home-dir` exists only on the dev runner
(`scripts/dev-runner.ts:847-903`). Resolution (`apps/server/src/cli/config.ts:319-327`): `--base-dir`
> `T3CODE_HOME` > desktop bootstrap `t3Home` > `~/.t3` (`apps/server/src/os-jank.ts:105-111`).

The worktree `.t3` default is a **dev-runner** feature (`dev-runner.ts:670-677`,
`packages/shared/src/devHome.ts:93-103`). `node apps/server/src/bin.ts serve` with no flag therefore
resolves to `~/.t3` and its state dir to `~/.t3/userdata` (`apps/server/src/config.ts:140-143`:
`userdata` unless a dev URL is set without an explicit base dir), which is the live database of the
installed app. AGENTS.md rule 2 forbids this; two additional reasons:

- Same `environment-id` file, same ID: the desktop's registry treats the pairing as its own platform
  environment and silently does nothing (`packages/client-runtime/src/connection/registry.ts:563-565`).
- `serve` does not refuse to start over a running server on the same base dir; only `start`/bare `t3`
  pass `rejectRunningServer` (`apps/server/src/cli/server.ts:30-70`).

Recommended layout, one fixed directory outside the checkout, reused by every command:

```
~/.t3-dev-wallet/                       # --base-dir (or T3CODE_HOME)
  userdata/                             # statev2.sqlite, settings.json, secrets/, logs/, server-runtime.json
  userdata/server-browser/profiles/     # persistent browser profiles (ServerBrowser.ts:502)
  tools/chrome-headless-shell/<platform>/<version>/   # headless Chrome, downloaded on first tab
  caches/  worktrees/
```

Seed `userdata/statev2.sqlite` with `vp run migrate-dev-db` into a worktree `.t3` and copy it in if
real projects are wanted (AGENTS.md "Test data"); an empty state dir also works, the server creates
it.

Port: `DEFAULT_PORT = 3773` (`apps/server/src/config.ts:23`); in web mode it moves to the next free
port when the desktop's bundled server already holds 3773 (`cli/config.ts:297-312`). Read the port
from the `Connection string:` line. Bind host defaults to `127.0.0.1` (`apps/server/src/server.ts:287`);
there is no `--allow-lan` flag, use `--host <lan-ip>` or `--host 0.0.0.0` (`cli/config.ts:38-41`,
`T3CODE_HOST` at `:131`).

## 2. Pairing the stock desktop app

`serve` prints (`apps/server/src/startupAccess.ts:122-146`):

```
T3 Code server is ready.
Connection string: http://localhost:<port>
Token: <token>
Pairing URL: http://localhost:<port>/pair#token=<token>
<QR code>
```

- The startup credential carries **administrative** scopes (`AuthAdministrativeScopes`,
  `apps/server/src/auth/EnvironmentAuth.ts:1098-1104`). The host is `localhost` when no `--host` is
  given, the first external IPv4 for a wildcard bind, otherwise the given host
  (`startupAccess.ts:45-78`).
- A fresh link for a running server, **standard** scopes, 5-minute TTL, same base dir so it finds
  `server-runtime.json` (`apps/server/src/cli/pair.ts:239-290, 479-486`):

  ```sh
  node apps/server/src/bin.ts pair --base-dir ~/.t3-dev-wallet
  ```

  `--scope` repeated replaces the default set (`apps/server/src/cli/authScopes.ts:4-11`;
  `docs/user/remote-access.md:250-260`). It prints the loopback note "This URL is only reachable from
  this machine" (`pair.ts:510-514`), which is fine when the desktop app runs on the same machine.
- Standard scopes already include `preview:operate` (`packages/contracts/src/auth.ts:205-218`), which
  is what the browser panel RPCs and the frame stream require
  (`apps/server/src/auth/RpcAuthorization.ts:189-199`, `apps/server/src/preview/ServerBrowserStream.ts:71-76`).
  Admin adds only `access:*` and `relay:write` (`auth.ts:220-224`). Either link works for this.
- In the desktop app: **Settings → Connections → Add environment**, paste the **full** pairing URL
  (`docs/user/remote-access.md:54-57`). A bare host without scheme is normalized to `https://`
  (`apps/web/src/components/settings/ConnectionsSettings.tsx:336-343`), so include `http://`. Loopback
  origins are accepted by the client (`packages/shared/src/remote.ts` `resolveRemotePairingTarget`;
  `packages/client-runtime/src/connection/onboarding.ts:110-150`); `docs/user/remote-access.md:56-57`
  only warns that `127.0.0.1` reaches the device opening the link, which here is the same machine.
  The environment's `serverBrowser` capability is unconditional (`apps/server/src/environment/ServerEnvironment.ts:262`).
- Pairing the same environment again replaces its saved grant and adds a route, not a second
  environment (`docs/user/remote-access.md:65-74`). Re-pair to change scopes.

## 3. Tabs render headless on the server and stream to the desktop

For a server-runtime tab on a server the desktop did not launch:

- `apps/server/src/preview/DesktopBrowserChannel.ts:75-93`: `available: false` unless both
  `config.desktopBrowserFd` and `config.desktopBrowserControlFd` are set. They come only from the
  desktop bootstrap envelope read via `--bootstrap-fd` / `T3CODE_BOOTSTRAP_FD`
  (`apps/server/src/cli/config.ts:281-286, 366-367`; schema `packages/contracts/src/desktopBootstrap.ts:32-34`);
  the only producer is `apps/desktop/src/backend/DesktopBackendConfiguration.ts:578-579`
  (`desktopBrowserFd: 6, desktopBrowserControlFd: 7`), wired when the desktop spawns its own child
  (`DesktopBackendManager.ts:481-491`). A `serve` process has neither.
- `apps/server/src/preview/ServerBrowser.ts:708-720`: `desktopRenders` resolves to `false` when the
  channel is unavailable. `createTab` (`:745-772`) then takes `contexts.contextFor(profileId ?? "default", ...)`,
  `context.newPage()`, runs `prepareContext(context)` (`:767`, the seam for #2) and
  `presentAsChrome(cdp, host)` (`:772`).
- Screencast: `attachViewer` (`ServerBrowser.ts:2179-2343`) sends `Page.startScreencast`
  (`:2214-2220`) and forwards `Page.screencastFrame` as JPEG binary frames over the WebSocket route
  `/api/preview-stream/ws` (`apps/server/src/preview/ServerBrowserStream.ts:28, 67, 146-151`),
  mounted on the normal HTTP server (`apps/server/src/server.ts:706`). The desktop shows these for every
  non-primary environment (`apps/web/src/browser/previewRuntime.ts:56-73` `rendersServerTabNatively`
  is true only for the primary environment).

Chromium: the dependency is `playwright-core` 1.60.0 (`apps/server/package.json:58`), which ships no
browser. `apps/server/src/preview/PreviewBrowser.ts:24-31, 92` pins Chrome for Testing
`chrome-headless-shell` `154.0.8037.92` with per-platform SHA-256, downloaded on first use from
`storage.googleapis.com/chrome-for-testing-public` into `<base-dir>/tools/chrome-headless-shell/<platform>/<version>/`
(`:183-190, 461`). Callers wait 45 s then get "T3 is installing its headless browser"
(`:150-155, 104-117`); the download times out at 15 minutes. It never uses a user-installed Chrome.
`ServerBrowserContexts.ts:96-99` passes this path as `executablePath` to `chromium.launch` /
`launchPersistentContext`. User doc: `docs/user/remote-access.md:193-196` ("about 120 MB, into the T3
home"). With a separate `--base-dir` the fork downloads its own copy.

## 4. Reaching the dapp dev server and Anvil on `localhost`

Yes. The headless browser is a child of the server process on the same machine, and nothing in the
launch or context options constrains the network:

- `apps/server/src/preview/ServerBrowserContexts.ts:95-112` `launchOptions`: `args` are
  `--disable-gpu`, `--force-device-scale-factor=2`, `--disable-blink-features=AutomationControlled`;
  `ignoreDefaultArgs: ["--enable-automation"]`, `headless: true`, `chromiumSandbox` unless disabled.
  No `proxy`, `--proxy-server`, `--host-resolver-rules`. Context options (`:168`) are only viewport
  and `deviceScaleFactor`.
- `presentAsChrome` (`ServerBrowserContexts.ts:73-80`) only overrides the user agent to drop
  `HeadlessChrome`.
- `prepareContext` (`ServerBrowser.ts:1128-1142`) only grants `clipboard-write`, exposes a binding and
  adds the clipboard init script. No `page.route`, `Fetch.*`, `setBlockedURLs` or URL allowlist exists
  in `apps/server/src/preview/`.
- URL validation is `normalizePreviewUrl` (`packages/shared/src/preview.ts:138-166`): rejects non-http(s)
  schemes only; bare `localhost`/`127.0.0.1` input gets `http://`. Discovered ports are deliberately
  opened as `http://localhost:<port>` because "the browser runs inside the environment, so its ports
  are loopback" (`ServerBrowser.ts:1377-1380`, `PortScanner.ts:162, 214, 275`).
- The one indirect way to break this: the launch inherits the server's `env` (`ServerBrowserContexts.ts:97`),
  so an ambient `HTTP_PROXY` without `NO_PROXY=localhost,127.0.0.1` would route loopback through a
  proxy. Nothing in the repo sets one.

Anvil's default `--allow-origin` is `*`, so a page served from the dapp origin can `fetch` `:8545`
directly (decided in #1).

## 5. T3 Connect / tunnel

Transport only: the browser runs on the host regardless, and the screencast rides the same HTTP
router as every other request (`server.ts:706`; `docs/internals/remote.md:3-7`: routes "do not
introduce another execution model"). Not needed for a same-machine setup.

## Checklist

```sh
# in the fork checkout, branch dev-wallet
vp i
node apps/server/src/bin.ts serve --base-dir ~/.t3-dev-wallet      # copy "Pairing URL: http://localhost:<port>/pair#token=..."
# stock desktop: Settings -> Connections -> Add environment -> paste the full URL
node apps/server/src/bin.ts pair --base-dir ~/.t3-dev-wallet       # later re-pairs; standard scopes include preview:operate
# open the dapp URL in the browser panel, then "Open in <env>'s browser" (or let the agent open it)
```

Open points for other tickets: the "move tab" step is a UX cost of the remote-environment decision
(#1); the init-script seam (#2) must live in `prepareContext`, which only server-runtime tabs hit.
