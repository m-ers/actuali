# Enable Banking bank sync in Actuali (MattFaz/actuali #747)

## Context

Issue #747 on upstream `MattFaz/actuali` asks for Enable Banking support ("help wanted", no maintainer guidance). In Actual Budget the sync-server only *proxies* Enable Banking; the **client** triggers each download and reconciles the transactions. Actuali can't do that today, so an Enable Banking account linked in the web app never refreshes from the phone.

Upstream Actual (`actualbudget/actual` @ `4ff4aac`, 2026-11-25) has Enable Banking merged and its server explicitly supports native clients. Actuali already has the same shape for SimpleFIN (`/simplefin/*` through `ActualServerBankSyncProvider`), and its reconciler, approval queue, link/unlink writes and link sheet are already provider-neutral.

**Outcome:** Actuali syncs Enable Banking accounts, and can authorize a bank, link accounts and re-authorize (consent lasts ≤ 90 days) without the web app. The one-time server setup (Application ID + `.pem`) stays in the web app.

**Agreed with the user:** scope = sync + linking; delivered as upstream PRs with tests; executed on the user's Mac with Xcode, work handed to smaller-model worker agents.

## How bank sync works (who does what)

The server fetches from the bank, but only when a client asks, and only as a pass-through. It has no schedule and never writes to the budget (it can't read an encrypted one).

1. **Client → Actual server:** "transactions for account X since date D."
2. **Server → Enable Banking → bank**, using the key only the server holds.
3. **Server → client:** the transactions, unsaved.
4. **Client:** match against the budget, apply rules, write the new ones.
5. **Client → server:** the ordinary budget sync, so other devices see them.

Steps 2, 3 and 5 exist already. **PR 1 adds steps 1 and 4** for Enable Banking. **PR 2 adds granting the server access to a bank** (expires after ≤ 90 days), which a client also starts. Actuali never talks to Enable Banking or the bank directly, and stores no bank credential.

Enable Banking stays behind the sync button, as SimpleFIN does in Actuali today. Automatic syncing on app open is not part of this plan.

## Decisions

| Topic | Decision | Why |
|---|---|---|
| Return from the bank | **No URL scheme registered.** Open the bank page in an in-app `ASWebAuthenticationSession`, send `returnToApp: false`, treat the server's `poll-auth` result as the only source of truth, and dismiss the sheet when it arrives. | Actuali stays in the foreground for the web part, so its waiting request is live when the bank finishes. Nothing depends on `actualbudget://` (which the official app registers in `packages/mobile-client/ios/App/App/Info.plist`), so no clash is possible. |
| Banking app hands off to Safari | Hold a background-task assertion while waiting; retry the poll on transport failure. Confirmed by **Gate G1** on a real bank before PR 2 opens. | The server keeps a finished result 30 s (single read). The official app survives this with one long request + its scheme; G1 shows whether Actuali needs more. |
| First import window | Enable Banking first sync starts at `max(importStart, today − 89 d)`. | Upstream's `getAccountSyncStartDate` default; banks may reject older `date_from`. |
| `importPending` | Not sent. | Server default returns everything; `importBankSync` already drops pending per the synced preference. |
| Server credential setup | Out of scope (web app). | Once per server, admin-only. |

## Wire protocol (all the workers need)

POST JSON, header `X-ACTUAL-TOKEN`. Envelope `{status:"ok", data}`; app errors are **HTTP 200 with the error inside `data`**. Upstream refs: `packages/sync-server/src/app-enablebanking/app-enablebanking.ts`, `…/services/enablebanking-service.ts`, `packages/loot-core/src/server/accounts/{sync,app}.ts`.

| Route | Request body | `data` |
|---|---|---|
| `/enablebanking/status` | `{}` | `{configured: Bool}` |
| `/enablebanking/aspsps` | `{country}` (upper-case ISO) | `[bank]` **or** `{aspsps:[bank]}`; error `{error}`. bank = `{name, country, psu_types?, beta?, maximum_consent_validity?}` (seconds) |
| `/enablebanking/start-auth` | `{aspsp:{name,country}, redirectUrl, psuType:"personal"\|"business", maxConsentValidity?, returnToApp:false}` | `{url, state}`; error `{error}` or `{error_code, error_type}` |
| `/enablebanking/poll-auth` | `{state}` — long-poll, server cap 300 s | `{session_id, accounts:[{account_id, name, institution, balance}], aspsp}` or `{error: String}` (`"Polling timed out"`, …) |
| `/enablebanking/transactions` | `{accountId, startDate:"YYYY-MM-DD", aspspName?}` — one account per request | `{transactions:{all:[tx]}, balances, startingBalance: Int\|null}` or `{error_type, error_code}` |

- `redirectUrl` = `<configured primary server URL>/enablebanking/auth_callback`; must be `https` and equal the URL whitelisted in the Enable Banking application (same one the web app uses).
- `tx` = `{transactionId, date, payeeName, notes?, booked, transactionAmount:{amount:"-25.99"}}` plus raw bank fields. This is exactly what `ServerBankSyncTransaction` + `BankSyncCandidate.init(serverBankSync:)` already decode. `startingBalance` is the current balance in cents.
- Error codes: `ITEM_LOGIN_REQUIRED` (consent expired), `RATE_LIMIT_EXCEEDED`, `TIMED_OUT`, `NOT_CONFIGURED`, `INVALID_INPUT`, `NOT_FOUND`, `INTERNAL_ERROR`. No `reason` field.
- Link writes (upstream `enablebanking-accounts-link`): `accounts.account_id = uid`, `account_sync_source = "enableBanking"`, `accounts.bank → banks{bank_id: uid, name: institution}`. Re-authorizing issues **new uids**, so each new remote account is re-linked onto the existing local account.

## PR 1 — Sync Enable Banking accounts (`feat/enable-banking-sync`)

Standalone value: accounts linked in the web app refresh from Actuali.

### WP1 · Server client: transactions endpoint
`Actuali/Actuali/Services/Network/ActualServerClient.swift`
- `postBankSync`: add `timeout: TimeInterval? = nil` (sets `request.timeoutInterval`); reword its doc comment to "bank-sync routes".
- `enum EnableBankingDownload: Decodable, Sendable { case account(ServerBankSyncAccountDownload), failure(ServerBankSyncError) }` — decode error-first (failure only when `errorCode != nil`), the pattern `ServerBankSyncDownloads` uses.
- `func enableBankingTransactions(accountId:startDate:aspspName:) async throws -> EnableBankingDownload?` — `fileId: nil`, `timeout: 60` (upstream's value); `nil` = route not served.
- `ServerBankSyncError.bankSyncStatus`: add `"NOT_CONFIGURED" → "not-configured"` (upstream `getBankSyncStatusFromError`).

Tests → `ActualiTests/Services/BankSync/ServerBankSyncTests.swift`: decode a success payload (raw fields spread in, `startingBalance: null`); error payload → `.failure`; add `NOT_CONFIGURED` to `mapsErrorCodesOntoActualsStatusVocabulary`; candidate from an Enable Banking tx (debit sign, `booked:false`, empty `payeeName` → notes).

### WP2 · Source, provider, routing
- `Services/BankSync/BankSyncAccount.swift`: `case enableBanking` on `BankSyncSource`; `var institutionName: String? = nil` (last property) on `BankSyncAccount`; `static func attentionMessage(status:source:)` backing the existing property. **Keep the SimpleFIN strings/keys untouched**; add Enable Banking variants for `reauth-required` and `not-configured`.
- `Services/BankSync/BankSyncProvider.swift`: `BankSyncTarget.institutionName: String? = nil`; `struct EnableBankingProvider { let client: ActualServerClient; func download(_:) async -> BankSyncDownloadSet }` — sequential per-account requests, never throws, **every target gets a `byAccount` entry** (candidates + `currentBalanceCents`, or `problem` + `status`). Problem text from `attentionMessage(status:source:)`, never the raw error code. Does not conform to `BankSyncProvider` (no accounts endpoint), same as `AppleWalletProvider`.
- `Services/Database/BudgetDatabase.swift` `fetchBankSyncAccounts()`: add the bank's name via a subquery on `banks`, guarded by `db.tableExists("banks")` (test schemas often lack it; a failed query would silently empty the list).
- `Services/BudgetStore.swift` `syncBankAccounts` — edit points:
  1. `enableBankingTargets` beside `simpleFinTargets`; include in the empty guard, `bankSyncingAccountIds`, and `targets`.
  2. `downloadTargets`: pass `institutionName`; first-import branch uses `max(importStart, lookbackFloor)` for `.enableBanking`.
  3. SimpleFIN `catch` guard and Wallet `catch` guard: only rethrow when **both** other groups are empty.
  4. New download block calling `EnableBankingProvider(client: serverClient)`.
  5. Missing-download branch: make `sourceHasProblems` and the `simpleFinFailureIsDeviceLocal` check apply to `.simpleFin` targets only (today "not FinanceKit" is read as SimpleFIN).
  6. `unsupportedSourceMessage` comment (and the test comment at `BudgetStoreBankSyncTests.swift:49`): Enable Banking is no longer "unsupported".

Tests → new `ActualiTests/Services/BankSync/BudgetStoreEnableBankingSyncTests.swift` (handler-based `StubTransport.session`, read `request.url.path` / `request.bodyData`; create `banks` inline as `BudgetStoreBankSyncTests.swift:179` does):
syncs through `/enablebanking/transactions` (asserts body incl. `aspspName`, no `importPending`; imported ids, cleared flag, `bank_sync_status = ok`, `last_sync`) · first sync opening balance + 89-day cap · `ITEM_LOGIN_REQUIRED` → `reauth-required`, localized problem, nothing imported · SimpleFIN failing doesn't stop Enable Banking and vice-versa · 404 route · `attentionMessage` per source · `fetchBankSyncAccounts` with and without a `banks` table.

### WP3 · Strings (PR 1)
Add every new key to `Actuali/Actuali/Localizable.xcstrings` in all 7 locales (en, fr, es, pt-BR, de, it, nl); run `python3 dev/scripts/validate-localization.py`. Service-layer text goes through `ReportStrings.text` / `String(localized:)` — don't copy the unlocalized literals that exist in `syncBankAccounts`.

## PR 2 — Authorize, link and re-authorize from Actuali (`feat/enable-banking-link`, stacked on PR 1)

### WP4 · Server client: status, banks, authorization
- New `Actuali/Actuali/Services/BankSync/EnableBanking.swift`: `EnableBankingBank`, `EnableBankingAuthorization {url, state}`, `EnableBankingRemoteAccount` with `var remoteAccount: BankSyncRemoteAccount` (`id` and `institutionId` = `account_id`, `institutionName` = `institution`, `balanceCents` = `balance`, `source: .enableBanking`), and `enum EnableBankingAuthError: LocalizedError { case timedOut, failed(String), insecureServer }`. (File-system-synchronized groups: no `pbxproj` edit.)
- `ActualServerClient`: `enableBankingStatus() -> Bool?`; `enableBankingBanks(country:)` (accept both shapes); `enableBankingRedirectURL()` from `configuredPrimaryURL` (the fallback route would not match the whitelist); `enableBankingStartAuthorization(bank:psuType:)` (rejects non-`https`; sends `maxConsentValidity` only when a positive integer ≤ 315 360 000); `enableBankingPollAuthorization(state:)`.
- The poll calls `session.data(for:)` directly — **not** `send` — with `timeoutInterval = 280` (the session's resource cap is 300 s), keeping `makeRequest`'s custom headers and the token. `send` would treat a poll that died during suspension as an unreachable server and swap to the fallback URL.

Tests → new `ActualiTests/Services/Network/EnableBankingClientTests.swift`: status true/false/404; both bank-list shapes + error; exact start-auth body and redirect URL; poll success, `"Polling timed out"` → `.timedOut`, other error → `.failed`; poll carries token and custom headers.

### WP5 · Store + browser session
- `Services/BudgetStore.swift` (beside the other bank-sync methods): thin wrappers, plus `awaitEnableBankingAuthorization(state:) -> [BankSyncRemoteAccount]` — loops the poll, retrying after a 1 s pause on any non-cancelled `URLError` until a 10-minute deadline, then `.timedOut`. Task cancellation ends it.
- `Services/Network/OpenIDAuthenticator.swift`: extract `open(_ url:, callbackScheme:) async throws -> URL` (existing `authenticate` becomes a caller) and `cancel()` (no-op without a session; clear `session` on completion). Keep `callbackScheme` / `returnURL` so `loginWithOpenID` is untouched.
- No change to `linkBankAccount`: re-linking an already-linked account passes the existing compare-and-swap and overwrites `account_id` / `bank`.

Tests (same suite as WP2 or a sibling): poll retry after a handler-thrown `URLError(.networkConnectionLost)`; link writes the three columns + bank row; re-link with a new uid replaces `account_id` and keeps `last_sync`.

### WP6 · UI
- New `Actuali/Actuali/Views/Accounts/EnableBankingSyncView.swift`:
  - **State:** route missing → "update your server"; not configured → "set it up once in the Actual web app" + the redirect URL to whitelist; ready → sections below.
  - **Linked accounts** (source `.enableBanking`): name, last sync, attention message, swipe to unlink, "Sync now".
  - **Connect a bank:** country (`AT BE BG HR CY CZ DK EE FI FR DE GR HU IS IE IT LV LI LT LU MT NL NO PL PT RO SK SI ES SE GB`, names from `Locale`, default `Locale.current.region`), searchable bank list, usage type only when the bank offers both.
  - **Authorize:** start → present sheet via `OpenIDAuthenticator.open` in a child task → `awaitEnableBankingAuthorization` → `cancel()` the sheet → show returned accounts as `BankRemoteAccountRow`s opening the existing `BankAccountLinkView`. Closing the sheet early keeps a "Waiting for your bank…" state with Cancel. Wrap the wait in `UIApplication.shared.beginBackgroundTask`. The flow task is cancelled on Cancel and `onDisappear`.
  - **After the link sheet closes:** sync Enable Banking accounts that have never synced or need attention (`syncBankAccounts(accountIds:)`, result in a local alert). Upstream syncs on link; this also clears a stale `reauth-required` after a re-link.
- `Views/Accounts/BankSyncSetupView.swift`: third provider row (`bankSync.provider.enableBanking`); `BankAccountLinkView.linkableAccounts` also offers accounts currently linked via Enable Banking when the remote is Enable Banking (needs-attention first), with a footer explaining it replaces the old link.
- Pure logic as `nonisolated static` helpers (AGENTS.md concurrency rule 3) so tests need no `@MainActor`. Identifiers: `enableBanking.country`, `.bank`, `.usageType`, `.authorize`, `.cancel`, `.syncNow`, `.remoteAccount.<id>`, `.linked.<accountId>`.

Tests → new `ActualiTests/Views/Accounts/EnableBankingSyncViewTests.swift` for the static helpers (readiness mapping, linkable accounts, accounts needing a sync, default country).

### Gate G1 · Real-bank test on a device (before WP7 and before opening PR 2)
Authorize the user's own bank from a physical iPhone and record where the redirect lands.

| Observation | Action |
|---|---|
| Finishes inside the sheet; accounts appear | Done. |
| Banking app → Safari; accounts appear after switching back | Done. Note it in the PR. |
| Result lost after returning from Safari | Stop and decide with the user: (a) propose a custom return URL to `actualbudget/actual` (as OpenID's `returnUrl` already allows), or (b) register `actualbudget://` with `returnToApp: true`. Do not pick silently. |
| First sync of a new account fails | Check the `startDate` sent; the 89-day cap should prevent it. |

### WP7 · Strings (PR 2) — as WP3.

## PR 3 — Website copy (`docs/enable-banking-website`)
### WP8
`website/src/pages/index.astro` (≈126–127, 203) and `privacy.astro` (≈25–26, 49, 72–77): mention Enable Banking next to SimpleFIN; privacy: it runs through the user's Actual server, the app opens the bank's page in a browser sheet, no credential is stored on the device. Flag in the PR that it should merge with the release.

## Handing the work to agents (on the Mac)

1. `git remote add upstream https://github.com/MattFaz/actuali && git fetch upstream`; cut each feature branch from `upstream/main` (the fork's `main` carries a fork-only commit). Optional reference clone: `git clone --depth 1 https://github.com/actualbudget/actual actual` (gitignored).
2. **One worker per WP, in order** (WP1 → 2 → 3, then 4 → 5 → 6 → G1 → 7; WP8 any time). Run them on the smaller model (`model: "sonnet"` in the Agent tool). Sequential on purpose: WP2/5/6 share `BudgetStore.swift` and the string catalog, and each must compile on the last.
3. **Brief per worker** — keep it to: "Read AGENTS.md. Read `dev/plans/enable-banking-747.md` sections *Decisions*, *Wire protocol* and *WP n*. Implement WP n only; touch only the files it names; write the listed tests; add English strings in code only (the strings WP owns the catalog). Run `swiftformat Actuali`, build, and `dev/scripts/sim-test.sh any -only-testing:ActualiTests/<Suite>`. Report: files changed, test results verbatim, anything in the plan that didn't match the code." Don't paste code into the brief.
4. **Orchestrator** (main session) reviews each `git diff` against the WP before starting the next, and runs `/code-review` on the full branch before each PR. WP6 and G1 need a person at the simulator/phone.
5. PRs: `gh pr create --repo MattFaz/actuali --head m-ers:<branch>`; brief and casual, name the upstream files above so the parity can be checked, say what was run. PR 1 "Part of #747", PR 2 "Closes #747", opened once PR 1 merges. No version or build bumps.

## Verification

- Build: `xcodebuild -project Actuali/Actuali.xcodeproj -scheme Actuali -sdk iphonesimulator build 2>&1 | grep -E 'error:|warning:|\*\* BUILD'` (timeout 600000 ms).
- Per WP: `dev/scripts/sim-test.sh any -only-testing:ActualiTests/<Suite>`.
- Before each PR: `dev/scripts/sim-test.sh any` **and** `dev/scripts/sim-test.sh ios18` (CI runs both), `swiftformat --lint --strict Actuali`, `python3 dev/scripts/validate-localization.py`. Existing sync fixture tests must stay green untouched.
- PR 1 end-to-end: with an account linked through the web app, "Sync from Bank" in Actuali imports transactions; the web app shows the same `last_sync` and status.
- PR 2 end-to-end: Gate G1, then link a new account, re-link an existing one, and confirm the web app can still sync both.

## Delivery from this session

The user asked to start the work here, with proper branch names, and will take over on the Mac within the hour. This container has no Xcode, so **everything committed here is uncompiled and untested**; SwiftFormat has not been run either.

- Code: `feat/enable-banking-sync` (PR 1) and, if time allows, `feat/enable-banking-link` (PR 2, stacked on it). Both are cut from upstream `main` (`MattFaz/actuali`, currently ahead of the fork's `main`), one commit per work package, so they can become upstream PRs as they are.
- This plan: its own commit on `claude/happy-faraday-9y1w3i` only. It never enters the feature branches.
- Order here: WP1 + WP2 + WP3 in parallel (disjoint files, interfaces fixed up front), then WP4 + WP5. WP6 (UI) and Gate G1 need Xcode and a phone and stay for the Mac.
- A *Status* section at the end of the committed plan records what is done and what to run first.

## Status at hand-off (2026-11-25)

**Nothing below has been compiled, run, or passed through SwiftFormat.** It was written and then reviewed by reading only. Expect compile errors; the first job on the Mac is to build.

| Branch | Holds | State |
|---|---|---|
| `feat/enable-banking-sync` | WP1, WP2, WP3 (PR 1) | Complete. Reviewed by reading; three test fixes applied. |
| `feat/enable-banking-link` | PR 1 + WP4, WP5, WP6, WP7 (PR 2) | Plumbing (WP4, WP5) reviewed by reading; one test fix applied. The screen (WP6) is a first draft that was only skimmed. Contains one merge commit from the sync branch. |
| `docs/enable-banking-website` | WP8 (PR 3) | Complete. |
| `claude/happy-faraday-9y1w3i` | This plan | Not part of any PR. |

All three feature branches are cut from upstream `main` at `ebdbb5d`.

### First steps on the Mac

1. `git fetch origin && git switch feat/enable-banking-sync`
2. `swiftformat Actuali`, then build (command under *Verification*). Fix what the compiler reports and commit the fixes on this branch.
3. `dev/scripts/sim-test.sh any -only-testing:ActualiTests/EnableBankingDownloadTests -only-testing:ActualiTests/BudgetStoreEnableBankingSyncTests`, then the existing `BudgetStoreBankSyncTests` and `ServerBankSyncDecodingTests` to confirm nothing regressed.
4. End-to-end check for PR 1: an account linked through the web app syncs from "Sync from Bank".
5. `git switch feat/enable-banking-link && git merge feat/enable-banking-sync`, format, build.
6. `dev/scripts/sim-test.sh any -only-testing:ActualiTests/EnableBankingClientTests -only-testing:ActualiTests/BudgetStoreEnableBankingLinkTests -only-testing:ActualiTests/EnableBankingSyncViewTests -only-testing:ActualiTests/OpenIDAuthTests`
7. Walk the screen in the simulator (Settings → Bank Sync → Enable Banking), then **Gate G1** on a phone with a real bank. G1 is still open and decides whether the no-URL-scheme approach is enough.
8. Before each PR: full unit suite on `any` and `ios18`, `/code-review`, PR text.

### Where to look first if it doesn't compile

- `EnableBankingSyncView.swift`: the background-task expiry closure captures `self` and calls a view method (`endBackgroundTask`); the `Picker` with an optional `String` selection in `.navigationLink` style; `.task(id:)` on the country picker.
- `BankSyncAccount.attentionMessage(status:source:)`: a `return switch` with `where` clauses on string patterns of a `String?`.
- `EnableBankingProvider.download`: `switch` over an optional enum using `.account` / `.failure` / `nil` cases.
- `EnableBankingClientTests`: `request.timeoutInterval == 280` assumes the stub sees the per-request timeout; if it fails, only that assertion is wrong.
- Test fixtures build `EnableBankingBank`, `Account` and `BankSyncAccount` through memberwise initialisers.

### Small departures from the plan

- `not-configured` has one message for every source; the Enable Banking `reauth-required` message does not point at a screen, so it is true for PR 1 alone.
- `BankRemoteAccountRow` lost its `private` so the new screen can reuse it.
- `EnableBankingSyncView.psuType(for:choice:)` was added: a business-only bank would otherwise be sent `personal`.
- `enableBankingStatus()` reuses the private `SimpleFINStatusResponse` (same shape).
- A failed Enable Banking download still counts in `accountsSynced`, exactly as a failed SimpleFIN account already does. Left alone.

### Known, not caused by this work

- `python3 dev/scripts/validate-localization.py` fails on upstream `main` itself with five "missing catalog key for interpolated" errors. This work adds none.
- New catalog entries carry `"extractionState" : "stale"`, copied from their neighbours, and sit at approximate sort positions. Xcode rewrites both on its next catalog save.
- The existing `Bank` key is untranslated ("Bank") in es, fr, it and pt-BR; the new screen uses it as the bank picker's label.
