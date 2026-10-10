# Where the init-script seam goes in ServerBrowser

Resolves Wayfinder ticket [#2](https://github.com/FSM1/t3code-w3/issues/2) for the dev-wallet
map ([#1](https://github.com/FSM1/t3code-w3/issues/1)). Read-only research against
`fork/dev-wallet` at `5f7294d452`; no servers were run.

Line numbers below are from that commit. The ticket's approximate numbers (`prepareContext`
"near 1104", `createTab` "near 742") have drifted: they are at
[`ServerBrowser.ts:1128`](../../apps/server/src/preview/ServerBrowser.ts#L1128) and
[`ServerBrowser.ts:745`](../../apps/server/src/preview/ServerBrowser.ts#L745).

## Short answer

Add the per-profile script inside `prepareContext`, directly after
`await context.addInitScript(CLIPBOARD_SCRIPT)` at `ServerBrowser.ts:1141`, and widen the
function's signature so the call site at `:767` passes the raw `snapshot.profileId` and the
`isolatedContext` flag. That slot already runs exactly once per `BrowserContext`, after the
context exists and before its first `newPage()`, and every headless tab (human, agent,
incognito, popup) flows through it. Nothing else in the file needs to move.

## 1. Which contexts exist per profile, and when they are created

`ServerBrowserContexts.contextFor(profileId, isolationKey?)`
([`ServerBrowserContexts.ts:140-165`](../../apps/server/src/preview/ServerBrowserContexts.ts#L140-L165))
caches one context per key `JSON.stringify([profileId, isolationKey ?? null])` (`:142`).
`createContext` (`:167-180`) picks one of two shapes:

| Shape | Condition (`:169`) | How it is made | Lifetime |
| --- | --- | --- | --- |
| Persistent | no `isolationKey` and `profileId !== "incognito"` | `chromium.launchPersistentContext(<profilesDir>/<encoded profileId>)` (`:173-179`): its own Chromium process with on-disk storage | until `clearProfile` (`:215-233`) or `close` (`:235`) |
| Isolated | `isolationKey` set, or `profileId === "incognito"` | `sharedBrowser().newContext(...)` (`:170-171`): one shared `chromium.launch` browser (`:122-138`) hosts every isolated context | closed by the tab that owns it (`ServerBrowser.ts:699`, `:904`) |

Who asks for which, from `createTab`
([`ServerBrowser.ts:745-767`](../../apps/server/src/preview/ServerBrowser.ts#L745-L767)), the
only caller of `contextFor` in the server:

- Human tab with a profile: `contextFor(snapshot.profileId)` -> persistent, keyed
  `[profileId, null]`. Every tab of that profile shares the one context.
- Human tab with no profile (`snapshot.profileId === undefined`, no `automationOwner`):
  `contextFor("default")` -> the persistent `default` profile (`:762`).
- Incognito (`profileId === "incognito"`): `isolatedContext` is true (`:753-757`), so the
  call is `contextFor("incognito", tabKey)` -> a fresh isolated context per tab.
- Agent tab without a profile (`automationOwner` set, `profileId` undefined): also
  `isolatedContext` (`:756`), call is `contextFor("default", tabKey)` -> a fresh isolated
  context per tab. Note the mismatch: the pool sees `"default"`, but the tab records
  `profileId: snapshot.profileId` which is `undefined` (`:792`).
- Desktop-rendered tab: no server context at all. `connectDesktop` (`:723-742`) attaches
  over CDP to the page Electron renders (`ServerBrowserContexts.ts:203-212`).
- Popups: reuse the opener's context object, see section 2.

Creation time relative to the first navigation. Inside `createTab` the order is fixed:

1. `contexts.contextFor(...)` creates the context (`:759-765`). `contextFor` closes any page
   the launch opened before returning it (`ServerBrowserContexts.ts:153`), so a fresh
   context has zero pages here.
2. `prepareContext(context)` (`:767`).
3. `context.newPage()` (`:768`), then `newCDPSession` (`:769`) and `presentAsChrome` (`:772`).
4. The first navigation: `page.goto(snapshot.navStatus.url, { waitUntil: "commit" })` at
   `:912-918`, only when the session was opened with a URL (`navStatus._tag === "Loading"`,
   set by `Manager.open`, [`Manager.ts:216-218`](../../apps/server/src/preview/Manager.ts#L216-L218)).
   Idle tabs navigate later through `navigate` (`:1383`) or a viewer's `goto` (`:2150`).

So for a context's first tab the context exists, and `prepareContext` has run, strictly before
the page exists and before any URL is loaded. For later tabs in a persistent profile the
context is reused from the cache; `prepareContext` is a no-op (section 2) and the new page
inherits the context's scripts.

## 2. Do popups and profile-less agent tabs pass through `prepareContext`?

Both do, in different senses.

Popups ([`adoptPopup`, `ServerBrowser.ts:1144-1182`](../../apps/server/src/preview/ServerBrowser.ts#L1144-L1182)):

1. `page.on("popup")` on every tab calls `adoptPopup(tab, popup)` (`:897`).
2. `adoptPopup` opens a session with the opener's `profileId` (`:1161`) and, in
   `beforePublish`, stores the popup `Page` in `adoptedPages` under the new tab key
   (`:1166-1170`).
3. The manager's `opened` event reaches `mirrorManagerEvent` (`:1996-1999`) ->
   `ensureTab` -> `createTab`, which finds the adopted page (`:746`).
4. With `adopted` set, `desktop` is skipped (`:748-751`), `isolatedContext` is false
   (`:754`), and the context is `adopted.page.context()` (`:760`). Playwright returns the
   same `BrowserContext` object the opener holds
   ([`client/page.ts:253-255`](https://github.com/microsoft/playwright/blob/v1.60.0/packages/playwright-core/src/client/page.ts#L253-L255)).
5. `if (!desktop) await prepareContext(context)` (`:767`) runs, but the `WeakSet` guard
   (`:1127-1130`) returns immediately because the opener's context was prepared when its own
   tab was created.

The popup still gets every context init script. Chromium auto-attaches new targets with
`waitForDebuggerOnStart: true`
([`crPage.ts:512`](https://github.com/microsoft/playwright/blob/v1.60.0/packages/playwright-core/src/server/chromium/crPage.ts#L512)),
Playwright installs all context init scripts into the new target with `runImmediately`
(`crPage.ts:544-545`) and only then sends `Runtime.runIfWaitingForDebugger` (`:547`). The
comment at `ServerBrowser.ts:770-771` ("a popup's first request went out before it became a
tab") is about the user-agent override and network listeners, not init scripts. No popup
seam is needed.

Profile-less agent tabs: `isolatedContext` is true (`:753-757`), so `contextFor("default",
tabKey)` returns a brand-new isolated context, `prepareContext` sees an unprepared context and
installs the clipboard bridge. These tabs do pass through the seam, and the hook must decide
on purpose whether "no profile" means "the default profile's script" or "no script"; the
`"default"` passed to the pool is a storage key, not the user's choice.

Two paths bypass the seam, both correctly:

- Desktop tabs skip `prepareContext` (`:767`) because Electron owns the page and its
  session partition. A desktop wallet would be a separate seam in `apps/desktop`; out of
  scope for this ticket but worth recording on the map as an uncovered surface.
- `scratchPage` (`ServerBrowserContexts.ts:191-197`) makes a throwaway context without
  `contextFor`, for work that must not touch tab storage. It never gets tab scripts, which is
  what you want.

## 3. Does `context.addInitScript` apply to pages already open?

Not to their current documents. It applies to pages created afterwards and to any page's next
navigation.

The `playwright-core` version pinned by the server is `1.60.0`
([`apps/server/package.json:58`](../../apps/server/package.json#L58)). `node_modules` is not
installed in this worktree or the main checkout, so the vendored `types/types.d.ts` could not
be opened; the same file was read from the matching upstream tag,
[`packages/playwright-core/types/types.d.ts@v1.60.0`](https://github.com/microsoft/playwright/blob/v1.60.0/packages/playwright-core/types/types.d.ts),
and a local 1.63.0 copy of the package carries the identical paragraph. The
`BrowserContext.addInitScript` doc at lines 8190-8195 reads:

> Adds a script which would be evaluated in one of the following scenarios:
> - Whenever a page is created in the browser context or is navigated.
> - Whenever a child frame is attached or navigated in any page in the browser context. In this case, the script is evaluated in the context of the newly attached frame.
>
> The script is evaluated after the document was created but before any of its scripts were run. This is useful to amend the JavaScript environment, e.g. to seed `Math.random`.

The implementation confirms the "or is navigated" reading. `BrowserContext.addInitScript`
([`server/browserContext.ts:484-497`](https://github.com/microsoft/playwright/blob/v1.60.0/packages/playwright-core/src/server/browserContext.ts#L484-L497))
calls `doAddInitScript`, which for Chromium loops over the context's existing pages
([`crBrowser.ts:519-522`](https://github.com/microsoft/playwright/blob/v1.60.0/packages/playwright-core/src/server/chromium/crBrowser.ts#L519-L522))
and sends `Page.addScriptToEvaluateOnNewDocument` without `runImmediately`
([`crPage.ts:227-229`](https://github.com/microsoft/playwright/blob/v1.60.0/packages/playwright-core/src/server/chromium/crPage.ts#L227-L229),
[`:1016-1020`](https://github.com/microsoft/playwright/blob/v1.60.0/packages/playwright-core/src/server/chromium/crPage.ts#L1016-L1020)).
The already-loaded document is untouched; the script runs when that page next commits a
document. Contrast `exposeBinding`, which additionally evaluates its stub in all open frames
right away (`browserContext.ts:371`), so a binding is callable immediately while an init
script is not.

Two more lines from the same doc block matter for the design:

- "The order of evaluation of multiple scripts installed via `browserContext.addInitScript`
  and `page.addInitScript` is not defined." (`types.d.ts:8214-8217`). If the wallet script
  calls an exposed binding, call it defensively the way `CLIPBOARD_SCRIPT` does
  (`globalThis.${CLIPBOARD_BINDING}?.(...)`, `ServerBrowser.ts:445`).
- `addInitScript` returns a `Disposable` (`types.d.ts:8224`); `dispose()` removes the init
  script (`:19378-19387`, backed by `Page.removeScriptToEvaluateOnNewDocument`,
  `crPage.ts:1030`). Removal has the same "next navigation" semantics as installation.

For this seam the caveat is moot on the happy path: `prepareContext` runs before
`context.newPage()`, and `contextFor` has already closed the launch-time page, so there are
no pre-existing pages when the script is installed.

## 4. What the hook needs to know, and where `profileId` lives

`prepareContext` is `(context: BrowserContext) => Promise<void>` (`ServerBrowser.ts:1128`).
It does not receive, and cannot recover, the profile: the `BrowserContext` carries no
profile metadata, and `ServerBrowserContexts.contexts` is private.

The profile is in scope one frame up, in `createTab`:

- `snapshot.profileId: string | undefined` (`PreviewSessionSnapshot`,
  [`packages/contracts/src/preview.ts:214`](../../packages/contracts/src/preview.ts#L214)),
  recorded on the tab at `:792`.
- `isolatedContext: boolean` (`:753-757`), which distinguishes "incognito or profile-less
  agent tab" from "persistent human profile".
- The resolved pool key `snapshot.profileId ?? "default"` (`:762`).

Profile ids themselves come from the client. `BrowserProfileReporter`
([`apps/web/src/browser/BrowserProfileReporter.tsx`](../../apps/web/src/browser/BrowserProfileReporter.tsx))
sends the client's `browserProfiles` and default over `preview.reportProfiles`; the server
keeps only the latest report in memory (`reportedProfiles`, `ServerBrowser.ts:1270-1284`) and
uses it to resolve agent `open` requests (`resolveOpenProfile`, `:1299-1313`). Built-ins are
`default` and `incognito` ([`browserProfile.ts:51-61`](../../packages/contracts/src/browserProfile.ts#L51-L61));
user profiles are persistent by construction (`resolveBrowserProfiles`, `:81-92`). The hook
therefore keys on a plain string id that is stable across clients only insofar as the
clients agree; ticket #4 owns where the id-to-script mapping is stored.

Smallest change that makes script selection possible: widen the signature and pass what
`createTab` already has.

```ts
// ServerBrowser.ts:767
if (!desktop) await prepareContext(context, { profileId: snapshot.profileId, isolated: isolatedContext });

// ServerBrowser.ts:1128
const prepareContext = async (
  context: BrowserContext,
  tab: { readonly profileId: string | undefined; readonly isolated: boolean },
) => { ... };
```

Pass the raw `snapshot.profileId`, not `?? "default"`, so the policy for profile-less agent
tabs (ticket #6) is decided in the hook rather than by the storage key.

Why not hook `ServerBrowserContexts.createContext` instead (it has `profileId` and
`isolationKey`, and an `onContextCreate` option would mirror `onContextClose`):

- `createTab` is the sole `contextFor` caller, so the ordering guarantee is identical.
- The pool is a Playwright resource owner "outside the Effect runtime"
  (`ServerBrowserContexts.ts:1`). The reported profile list and any script configuration
  live in `ServerBrowser`, next to the clipboard bridge, which also needs server state
  (`tabs`, viewers) at `:1132-1140`. Keeping both bridges in one function keeps one seam.
- The test harness already fakes `addInitScript` on the context
  ([`ServerBrowser.test.ts:136`](../../apps/server/src/preview/ServerBrowser.test.ts#L136))
  and records every `contextFor` call with its profile (`:42`, asserted at `:1519-1523`), so
  a focused test for "profile X installs script X, incognito installs nothing" is cheap.

The one case where the `WeakSet` could in principle mis-select is "two profiles share a
context". It cannot happen: the cache key includes the profile id, and popups inherit the
opener's profile id (`:1161`), so each context maps to exactly one profile.

## 5. Recommended insertion point and ordering constraints

Insert at [`ServerBrowser.ts:1141`](../../apps/server/src/preview/ServerBrowser.ts#L1141),
immediately after `await context.addInitScript(CLIPBOARD_SCRIPT);`, inside the
`preparedContexts` guard of `prepareContext`. Call site unchanged in position (`:767`), only
widened to pass the profile.

Constraints the slot already satisfies, and must keep satisfying:

- Before `context.newPage()` (`:768`). The script must be registered before the context's
  first page so the first document gets it; later pages and popups inherit it from the
  context (sections 1 and 2). Nothing may move `prepareContext` below `:768`.
- Once per context. The `WeakSet` (`:1127-1130`) is the only thing preventing a second
  `exposeBinding` with the same name, which throws
  ([`browserContext.ts:359-360`](https://github.com/microsoft/playwright/blob/v1.60.0/packages/playwright-core/src/server/browserContext.ts#L359-L360)).
  If the wallet needs a server round-trip (an EIP-1193 request proxy), expose its binding
  here, before its init script, mirroring `:1132` then `:1141`.
- Independent of `presentAsChrome` (`:772`). That is a per-page CDP
  `Emulation.setUserAgentOverride` (`ServerBrowserContexts.ts:73-80`) issued after
  `newPage`; init scripts are context-level. Both land before the first `goto` (`:913`), so a
  wallet script reading `navigator.userAgent` sees the Chrome identity. There is no reason to
  interleave them.
- Independent of `grantPermissions` (`:1131`); keep the clipboard sequence as is and append.
- Skipped for desktop tabs by the existing `if (!desktop)`. Keep that: Electron's page is not
  a server context and the desktop app would need its own injection.
- Reload semantics. A context's scripts are fixed for its life. If the mapping changes while a
  persistent profile is open, the choices are: keep the `Disposable`, `dispose()` and re-add
  (takes effect on each page's next navigation, section 3), or close that profile's context
  so the next tab relaunches it. `clearProfile` (`ServerBrowserContexts.ts:215-233`) is not a
  reload: it deletes the profile's storage. Simplest first version: install once, document
  that a changed script applies to new contexts.
- Inherited fragility. `preparedContexts.add(context)` happens before the awaits (`:1130`),
  so if any step throws, the context is marked prepared with nothing installed and the tab
  creation fails; the next tab in that profile silently gets no bridges. The wallet step
  inherits this. Not this ticket's bug, but if the script fetch can fail (file read, config
  parse) consider moving the `add` after the awaits or resolving the script before touching
  the context.

## Sources

- [`apps/server/src/preview/ServerBrowser.ts`](../../apps/server/src/preview/ServerBrowser.ts)
  at `5f7294d452`: `createTab` `:745-920`, `prepareContext` `:1127-1142`, `adoptPopup`
  `:1144-1182`, `reportProfiles` `:1276-1284`, `resolveOpenProfile` `:1299-1313`,
  `mirrorManagerEvent` `:1996-1999`.
- [`apps/server/src/preview/ServerBrowserContexts.ts`](../../apps/server/src/preview/ServerBrowserContexts.ts):
  `contextFor` `:140-165`, `createContext` `:167-180`, `scratchPage` `:191-197`,
  `connectDesktopPage` `:203-212`, `clearProfile` `:215-233`.
- [`apps/server/src/preview/Manager.ts:207-230`](../../apps/server/src/preview/Manager.ts#L207-L230)
  (`open` builds the snapshot's `navStatus` and `profileId`).
- [`packages/contracts/src/browserProfile.ts`](../../packages/contracts/src/browserProfile.ts),
  [`packages/contracts/src/preview.ts:214`](../../packages/contracts/src/preview.ts#L214),
  [`apps/web/src/browser/BrowserProfileReporter.tsx`](../../apps/web/src/browser/BrowserProfileReporter.tsx).
- Playwright 1.60.0, upstream tag matching `apps/server/package.json:58`:
  [`types/types.d.ts`](https://github.com/microsoft/playwright/blob/v1.60.0/packages/playwright-core/types/types.d.ts)
  `:8190-8224`, `:19378-19387`;
  [`src/server/browserContext.ts`](https://github.com/microsoft/playwright/blob/v1.60.0/packages/playwright-core/src/server/browserContext.ts)
  `:358-376`, `:484-497`;
  [`src/server/chromium/crBrowser.ts`](https://github.com/microsoft/playwright/blob/v1.60.0/packages/playwright-core/src/server/chromium/crBrowser.ts)
  `:519-527`;
  [`src/server/chromium/crPage.ts`](https://github.com/microsoft/playwright/blob/v1.60.0/packages/playwright-core/src/server/chromium/crPage.ts)
  `:227-229`, `:512`, `:544-547`, `:1016-1031`;
  [`src/server/page.ts`](https://github.com/microsoft/playwright/blob/v1.60.0/packages/playwright-core/src/server/page.ts)
  `:900-903`;
  [`src/client/page.ts`](https://github.com/microsoft/playwright/blob/v1.60.0/packages/playwright-core/src/client/page.ts)
  `:253-255`.
