# GUI QML PR #1142 — End Model & Step-by-Step Refactor Plan

**Baseline:** [bitcoin-core/gui-qml#1142](https://github.com/bitcoin-core/gui-qml/pull/1142) · head `bc79778bb5661ea6fee9ae084801919772328532` · base `db9aa8429ac228ebf51a17804ca3da200f4f7fc0` · dependency tip `038668df86e764cf18ae5502fc7f8e3aff67072e` (51 dependency commits + 50 PR-owned commits) · Core `44901d8f64cb617714707bd025ecd83f108353c2`.  
**Interactive diagram:** [https://johnny9.github.io/gui-qml-object-model/](https://johnny9.github.io/gui-qml-object-model/) ([index.html](index.html)).

---

# Part I — The End Model

## 1. Ownership Tree

Every object has one clear owner. Orderly teardown is strictly reverse construction order (bottom to top):

```text
BitcoinQmlApplication::run              composition & app lifecycle (Controller)
├─ NativeApplicationInterfaces (Core)   Init, Node, Chain — created first, destroyed last in a final std::thread
├─ execution::Executor (Executor)       owns Core / Priority / Rpc / Utility lanes; outlives all models
├─ App Models & Controllers             NodeModel, SettingsModel, WalletController, ShutdownCoordinator, ...
│   ├─ each async resource owner holds 1 SerialQueue
│   └─ WalletModel (one per wallet)     UI facade, drafts, data models (no native Wallet/Handler ownership)
│       └─ walletopening::Session       single owner of unique_ptr<interfaces::Wallet>, handlers & 1 Core SerialQueue
└─ QQmlApplicationEngine (Views)        destroyed first on exit
    └─ Page (one per wallet for wallet pages)
        └─ Page Controller              owns UI state, prompts & presentation operation handles
```

## 2. Roles, Scopes & Approved Dependency Paths

**Role** (6 app/infra roles + Core) describes responsibility; **Scope** (`page`, `wallet`, `app`) describes lifetime.

| Role | Responsibility | Examples (target names) |
|---|---|---|
| **View** (QML) | Layout, bindings, focus, dialogs, navigation. No request tickets, schedulers, or native handles. | `MainWindow.qml`, `WalletView.qml`, `SendCreate.qml`, `SendForm.qml`, `SendTransactionReview.qml` |
| **Controller** (C++) | Use-case state, prompts, active operation handles. `BitcoinQmlApplication::run` owns composition/lifecycle. | `SendDraftController`, `TransactionReviewController`, `WalletController`, `ShutdownCoordinator` |
| **Interface** (C++ abstract) | Narrow capability contract (`*Interface`). Operation contracts use `*Operation`. Concrete adapters drop `*Interface`. | `WalletAvailabilityInterface`, `SendDraftInterface`, `TransactionReviewInterface`, `SettingsEditorFactoryInterface` |
| **Model** (C++) | Observable domain data, drafts, rows, projections, self-contained editors, or backend sessions. | `WalletModel`, `SettingsModel`, `NodeModel`, `TransactionReviewModel`, `SettingsEditorModel`, `walletopening::Session` |
| **Operation** (C++) | One async request with its own terminal state, result, and error, bound to its original target. | `SendOperation`, `WalletSendHandle`, `walletsend::SendTask`, `PreparationOperation`, `FileSaveOperation` |
| **Executor** (C++ internal) | Bounded worker lanes and lightweight FIFO `SerialQueue`s. Knows nothing about wallets, settings, Qt/QML, or Core roots. | `execution::Executor`, `execution::SerialQueue` |
| **Core** (Bitcoin Core) | Native C++ backend interfaces (`bitcoin/src/interfaces/`). Confined to explicit backend owners. | `NativeApplicationInterfaces`, `interfaces::Node`, `interfaces::Wallet` |

**Three approved dependency paths** (no forwarding controllers for symmetry):
1. **Workflow coordination:** `View → Controller → narrow contract → Model/Session` (abstract `*Interface` where an implementation/test seam helps, or a coherent facade like `WalletReceiveModel`).
2. **Presentation:** `View` binds observable `Model` and `Controller` properties directly.
3. **Self-contained editors:** `View` uses `SettingsEditorModel` or `PaymentRequestEditorModel` directly.

## 3. Generic Executor & SerialQueue Contract

```cpp
namespace execution {

enum class Lane {
    Core,     // Bitcoin Core calls (serialized one task at a time per SerialQueue; starts with 1 worker)
    Priority, // Dedicated fast thread for startShutdown(), shutdownRequested() polling, RPC stop, urgent close signals
    Rpc,      // Dedicated worker for ordinary RPC console commands so long RPCs never stall Core
    Utility,  // Files, QR images, external signer CLIs, off-UI cleanup (isolated Signer worker + IO/cleanup worker)
};

enum class UtilityWorker { IO, Signer };

class SerialQueue {
public:
    SerialQueue(SerialQueue&&) noexcept;
    SerialQueue& operator=(SerialQueue&&) noexcept;
    SerialQueue(const SerialQueue&) = delete;

    bool post(std::function<void()> task);        // FIFO; yields back to the lane worker after each task; returns false once closed
    void close();                                 // Stops new posts to this queue only; idempotent; non-blocking
    void whenDrained(std::function<void()> done); // Queued once after close() + empty & idle (supports multiple & late listeners)
private:
    friend class ValueCleanupQueue;
    bool reserveCleanup();                        // Keeps accepted value/handle/outcome cleanup deliverable during stopAndJoin()
    bool postReservedCleanup(std::function<void()> cleanup);
};

class Executor {
public:
    using Dispatch = std::function<bool(Lane, UtilityWorker, std::function<void()>)>;

    explicit Executor(Config config);     // Production: real Core / Priority / Rpc / Utility threads
    explicit Executor(Dispatch dispatch); // Tests: manual stepping, inline, or rejection lambda using real SerialQueue
    SerialQueue makeQueue(Lane lane, UtilityWorker utility = UtilityWorker::IO); // Lightweight strand; creates no thread
    void stopAndJoin();                   // Closes queues, drains remaining tasks (when real workers exist), physically joins all threads
};

} // namespace execution
```

* Closing one `SerialQueue` never closes `Executor` (which stays open until `stopAndJoin()`). `Session::stop(done)` posts cleanup onto an open `Utility` (or `Core`) queue.
* Two single-purpose owners each keep **1 internal thread** (with zero `BackendExecutor`, `TaskQueue`, or extra abstractions) and expose the same `stop(done)` contract:
  * **`QmlInitExecutor`**: 1 internal thread runs `appInitMain()`, `retireSubscription()`, and `appShutdown()` sequentially (preserving Core-thread cleanup affinity and preventing `appInitMain()` from blocking `beforeInterrupt` `SettingsModel` work on the shared `Core` lane), while `startShutdown()` posts on shared `Lane::Priority`.
  * **`desktopintent::transport::Service`**: 1 internal `QThread` runs the live `QLocalServer`/`QLocalSocket` Qt event loop and releases sockets/lock inside `Service::stop(done)` (its request queue already uses shared `Lane::Utility`).
* Task bodies, capture destructors, and `whenDrained` callbacks always execute outside queue locks; task captures are destroyed before yielding or reporting idle/drained.
* Each `SerialQueue` yields back to its lane's worker after executing one task, giving fair round-robin interleaving across sibling queues without feature-specific yield budgets or batch caps (and handling inline test dispatchers without growing the call stack).
* `ValueCleanupQueue` uses private `SerialQueue::reserveCleanup()` / `postReservedCleanup()` so already-accepted retained values, native wallet handles, and Core outcomes still retire on `Utility` during `Executor::stopAndJoin()` without re-introducing `TaskQueue` or `ResourceLease`. Keep this friendship strictly internal to `ValueCleanupQueue`.

## 4. Ownership, Lifecycle & Shutdown Contracts

1. **Native handles stay strictly inside backend owners:** Models, QML, and rejected `post()` lambdas never hold `interfaces::Wallet` or `interfaces::Handler`. Only `walletopening::Session` (or node backend) holds them via `std::unique_ptr` and frees them on a worker inside `stop(done)`. Plain values/snapshots use ordinary value ownership; Qt objects use `QObject` parent ownership.
2. **`SerialQueue::whenDrained()` vs. `Feature::stop(done)`:** `whenDrained()` covers tasks on that queue. `Session::stop(done)` closes and drains its `Core` queue, then completes native cleanup (disconnect handlers, optional `wallet->remove()`, `wallet.reset()`) on a worker before calling `done()`. `ShutdownCoordinator` waits on `stop(done)`.
3. **Core-thread cleanup affinity where required:** `QmlInitExecutor` subscription cleanup runs on `Core` (`nativeCleanupAffinityRetiresOnCoreWhileControlRemainsIndependent`). Other off-UI cleanup runs on `Utility`.
4. **Passive `close()` vs. explicit `cancel()`:** `SerialQueue::close()` rejects new posts without withdrawing already-supplied consent (`passive-close-preserves-consent`). Hiding/closing the review page ends prompts and withdraws unclaimed send consent; once `consent->claim()` succeeds, the single committed `Core` task finishes and records its actual outcome.
5. **App-wide shared authorities stay shared:** Cross-wallet backup destination/file-identity installation checks, wallet registry reopen/handoff barriers, and transient secret-safe RPC handling remain shared.
6. **Orderly 5-step shutdown (normal quit, early startup exit, recoverable error):**
   1. **Cancel waiting prompts & stop pre-interrupt features:** Immediately disable UI input, stop accepting new requests, and cancel any waiting prompts (so a prompt blocked on `Core` cannot deadlock a queued `SettingsModel` write behind it), then run `beforeInterrupt` `stop(done)` hooks.
   2. **Interrupt Core:** Post `node.startShutdown()` on `Lane::Priority`.
   3. **Stop remaining features:** Run `afterInterrupt` `stop(done)` hooks (wallets, node models, services); each drains its `SerialQueue` and completes worker cleanup before calling `done()`.
   4. **Shut down Core:** Run `node.appShutdown()` on `QmlInitExecutor`'s lifecycle thread (retiring subscriptions on that same thread) and join it.
   5. **Unwind in reverse ownership order:** Exit Qt loop → destroy Views (`QQmlApplicationEngine`) and Models/Controllers → flush queued `DeferredDelete` (`QCoreApplication::sendPostedEvents(nullptr, QEvent::DeferredDelete)`) while `QCoreApplication` still exists → `executor.stopAndJoin()` (physical joins including thread-local cleanup after `QThread::finished`) → move `Chain`/`Node`/`Init` into a final `std::thread`, destroy them in its body, then join it.
7. **Separate native-fatal unsafe exit:** Preserves existing fast-fail diagnostic/exit policy (`fatalExceptionExitsWithoutDraining`, `nativeFatalNeverStartsUnsafeTeardown`, `fatalInitializationRejectsPendingAndLaterShutdown`) — never runs unsafe `appShutdown()` or hangs waiting on a stalled worker.

## 5. Accepted Class Renames (20 Total)

| Current name | Target name | Role |
|---|---|---|
| `WalletQmlController` | `WalletController` | Controller |
| `RequestViewController` | `PaymentRequestViewController` | Controller |
| `RequestDeleteController` | `PaymentRequestDeleteController` | Controller |
| `QmlShutdownCoordinator` | `ShutdownCoordinator` | Controller |
| `WalletInterface` | `WalletAvailabilityInterface` | Interface |
| `SettingsEditorSource` | `SettingsEditorFactoryInterface` | Interface |
| `PeerCommandTarget` | `PeerActionsInterface` | Interface |
| `PeerCommandPort` | `PeerCommandInterface` | Interface |
| `WalletQmlModel` | `WalletModel` | Model |
| `WalletQmlModelTransaction` | `TransactionReviewModel` | Model |
| `ReceiveReservationModel` | `AddressReservationModel` | Model |
| `OptionsQmlModel` | `SettingsModel` | Model |
| `WalletBackups` | `WalletBackupModel` | Model |
| `FeeBumps` | `FeeBumpModel` | Model |
| `PeerListSortProxy` | `PeerListProxyModel` | Model |
| `WalletSendDraftInterface` | `WalletSendDraft` | Model |
| `WalletTransactionReviewInterface` | `WalletTransactionReview` | Model |
| `RequestOpenOperation` | `PaymentRequestOpenOperation` | Operation |
| `WalletSendOperation` | `WalletSendHandle` | Operation |
| `walletsend::Operation` | `walletsend::SendTask` | Operation |

---

# Part II — Step-by-Step Refactor Plan

### Delivery Strategy (Pick Before Phase 1)
* **Option A (recommended, lowest risk):** Finish **Phase 0** to merge PR #1142 green, then land Phases 1–5 as 3–4 small follow-up PRs on a green `qt6` base.
* **Option B (inside PR #1142):** Finish **Phase 0**, fold behavioral work into the 8 PR-owned area commits (`3da6df679ec8`, `cfea414652a0`, `fcb13840924e`, `53d3fab03864`, `fb47ba5dd57a`, `69a618a86ca3`, `261adf218605`, `08b740b69f34`) without touching the 51 dependency commits, and keep Phase 5's mechanical renames as **standalone trailing commits** (never rebasing 20 global renames back across commits 66–100).

---

### Phase 0 — Establish a Green Baseline & Close Coverage Gaps

1. **0.1 Archive & branch cleanly:** Create a local backup ref at `bc79778bb5661ea6fee9ae084801919772328532` and a clean refactor worktree/branch.
2. **0.2 Diagnose & fix the 5 failing CI jobs:** Inspect full logs for Ubuntu 22.04 (Qt 6.2), Ubuntu 26.04 (Qt 6.10), depends Linux x86_64, depends macOS native, and `qml-functional-tests`. Fix each root cause with its regression test.
3. **0.3 Fix the cancelled sanitizer run:** Diagnose why the sanitizer job hit the 1h30m limit and shard or trim slow runs so ASan/LSan/UBSan completes green.
4. **0.4 Close the 3 discovered baseline gaps:**
   * Wire `test/functional/interface_qml_console.py`'s 2 console-privacy scenarios into the regular functional workflow.
   * Wire `test/qml_api_schema.py` into the Qt 6.10+ lint build (`-DENABLE_QML_LINT=ON`).
   * Reconcile `#846`'s missing selector (`StartupProfileTests::originalFailureShowsActualErrorAndExitsCleanly` — restore coverage or update the PR claim).
   * Fix `doc/integration-testing.md` ("nine CTest entries" → 10 CTest entries across 9 stages due to wallet sending shards).
* **Phase 0 Gate:** Full CI matrix green on baseline before changing architecture.

---

### Phase 1 — Add `execution::Executor` & `execution::SerialQueue`

1. **1.1 Implement `Executor` and `SerialQueue`** (`qml/execution.h`, `qml/execution.cpp`, or new `qml/executor.h`/`qml/executor.cpp` alongside legacy code during migration):
   * `Lane::Core` (1 worker thread), `Lane::Priority` (1 dedicated thread), `Lane::Rpc` (1 dedicated console RPC thread), `Lane::Utility` (`UtilityWorker::Signer` external-process worker + `UtilityWorker::IO` I/O/cleanup worker so signer CLIs cannot starve cleanup).
   * `Executor(Config)` for production and `Executor(Dispatch)` for deterministic manual tests (sharing the exact same `SerialQueue` implementation; `stopAndJoin()` skips `waitIdle()` for injected dispatchers since the test owns pending execution).
   * `SerialQueue`: FIFO deque, per-task yielding back to the lane worker (stack-safe on inline dispatchers), `post()`, idempotent non-blocking `close()`, queued replayable `whenDrained()` supporting multiple and late listeners, private `ValueCleanupQueue` reservation support, executing tasks/destructors/callbacks outside locks.
   * `Executor::stopAndJoin()`: closes queues, drains remaining tasks, physically joins all worker threads (waiting through `thread_local` cleanup after `QThread::finished`).
2. **1.2 Wire one `Executor` in `BitcoinQmlApplication::run`** (`qml/bitcoin.cpp`) and remove the empty-wallet sentinel's unconditional 5-thread `BackendExecutor` (`qml/walletqmlcontroller.cpp`).
3. **1.3 Write deterministic + real-thread execution tests** (`test/test_execution.cpp`, `test/test_backendexecutor.cpp`):
   * *Manual `Executor(Dispatch)`:* FIFO ordering; Queue A closes/drains while Queue B runs; close while idle; rejected `post()` drops no native handles; multiple & late `whenDrained()` listeners; reentrant callback/capture destruction outside locks.
   * *Real threads:* `Priority` executes while `Core` blocks on a barrier; concurrent `post()`/`close()`; `stopAndJoin()` blocks until delayed `thread_local` destruction completes; steady-state thread count stays constant with 10+ `SerialQueue`s.
* **Phase 1 Gate:** All new deterministic + real-thread execution tests pass (with red/green negative controls) and existing suites remain green.

---

### Phase 2 — Migrate `walletopening::Session` & Wallet Features

1. **2.1 Make `walletopening::Session` the single wallet owner** (`qml/walletopening.h`, `qml/walletopening.cpp`, `qml/models/walletqmlmodel.h`, `qml/models/walletqmlmodel.cpp`):
   * Move exclusive `std::unique_ptr<interfaces::Wallet>` and `interfaces::Handler` ownership into `Session`. Remove direct native wallet handles from `WalletQmlModel` and sub-models.
   * Give `Session` one `Core` `SerialQueue` (and access to `Utility` cleanup queue). Remove per-wallet `BackendExecutor` (`m_backend_executor`) and replace `WalletQmlModel`'s 5-flag `checkShutdownFinished()` barrier with `Session::stop(done)`.
   * Implement `Session::stop(done)`: stop timers/ingress → `core_queue.close()` → on `core_queue.whenDrained()`, run native `cleanup()` (disconnect handlers, optional `wallet->remove()`, `wallet.reset()`) on a worker → call `done()`.
2. **2.2 Migrate `WalletRegistry` & `WalletQmlController`** (`qml/walletregistry.cpp`, `qml/walletqmlcontroller.cpp`):
   * Route wallet close/unload through `Session::stop(done)` before removing a wallet; preserve same-name reopen isolation and late-load retirement on a worker if the destination closed.
   * Remove `WalletQmlController`'s separate `m_wallet_command_executor`.
3. **2.3 Migrate wallet feature backends onto `Session`'s `SerialQueue`:**
   * **Send / preparation / fees / coins** (`qml/walletsend.cpp`, `qml/walletprepare.cpp`, `qml/models/walletsendoperation.cpp`, `qml/models/walletbalancereader.cpp`, `qml/models/feeestimatereader.cpp`): committed send stays one `Core` task; preserve `consent->claim()` boundary and `passive-close-preserves-consent`.
   * **PSBT / external signers / fee bumps** (`qml/walletreview.cpp`, `qml/walletbump.cpp`, `qml/models/psbtqmlmodel.cpp`, `qml/models/feebumps.cpp`): preserve authenticated fee proof, signer substitution checks, and wallet-owned unresolved fee-bump tracking.
   * **Payment/PSBT import, Activity, Receive, Backup** (`qml/walletrequests.cpp`, `qml/walletreservations.cpp`, `qml/walletreceipts.cpp`, `qml/models/activitystore.cpp`, `qml/models/backuppolicyio.cpp`): preserve shared cross-wallet backup installation lock and editor-write persistence.
* **Phase 2 Gate:** All 6 wallet boundary tests pass (idle wallet destroys `interfaces::Wallet` on a worker; rejected `post()` frees no native handle; holding `Utility` cleanup keeps `Session::stop(done)` pending; Wallet A stops while Wallet B stays busy; send page hide before vs. after `consent->claim()`; late wallet load after close) + all wallet unit, QML, integration (both sending shards), and functional tests pass.

---

### Phase 3 — Migrate App Services, Remaining `BackendExecutor` Users & Orderly Shutdown

1. **3.1 Fold `NodeModel::TrackedScheduler` directly into `SerialQueue`:**
   * Replace `NodeModel::TrackedScheduler` and `NodeModel::WorkLedger` (`qml/models/nodemodel.cpp`) with `NodeModel`'s own `SerialQueue`s on the shared `Executor` (`Core` for node queries/peer commands, `Priority` for `shutdownRequested()` polling, `Utility` for diagnostics/retirements).
   * `NodeModel::stop(done)` closes its queues, waits for `whenDrained()`, retires `m_subscriptions` on a worker, and calls `done()`. Delete `TrackedScheduler` and `WorkLedger` completely.
2. **3.2 Fold `QmlInitExecutor` / `nodelifecycle::Driver` without `BackendExecutor` or `TaskQueue`:**
   * Replace its 3-thread `BackendExecutor`, `TaskQueue` action scopes, and resource keys with **1 internal lifecycle thread** owned directly by `QmlInitExecutor` (running `appInitMain()`, `retireSubscription()`, and `appShutdown()` sequentially on the same thread — preserving `Core`-thread cleanup affinity and preventing a long `appInitMain()` from blocking `beforeInterrupt` `SettingsModel` work on the shared `Core` lane) plus a shared **`Lane::Priority` `SerialQueue`** for `node->startShutdown()`.
   * Remove `diagnostic_scope` / `fatalDiagnosticScheduler()` completely: `PrepareRunawayException` already bounds the error text on the worker, and `StartupDiagnosticsModel` can format on its own `Lane::Utility` `SerialQueue` (since `Executor` stays open until `stopAndJoin()`).
   * Emit `shutdownResult()` once `appShutdown()` completes and the 1 lifecycle thread joins inside `QmlInitExecutor::stop(done)`. Keep the separate native-fatal fast-fail path intact.
3. **3.3 Fold `desktopintent::transport::Service` without `BackendExecutor` or `TaskQueue`:**
   * Replace its 5-thread `BackendExecutor`, `openActionScope()`, `borrow()`, `ResourceLease`, and `reserveCleanup()` with **1 internal `QThread`** owned directly by `Service` (needed solely because `QLocalServer` and `QLocalSocket` require a live Qt event loop for incoming payment links; its request queue already runs on shared `Lane::Utility`).
   * Keep `QLocalServer`, client `QLocalSocket`s, and `QLockFile` exclusively inside `State::Native` on that 1 thread. `Service::stop(done)` stops accepting, calls `owner->stop()`, posts `native->retire()` + `thread.quit()` onto that thread, physically joins it, and calls `done()`. Register `Service::stop(done)` as a `beforeInterrupt` participant on `ShutdownCoordinator` (and in startup's early-exit guard).
4. **3.4 Migrate remaining app services onto shared `Executor` `SerialQueue`s:**
   * Move `OptionsQmlModel`, `RpcConsoleModel` (keeping secret redaction and `Priority` RPC `stop`), `PeerListModel`, `BanListModel`, `DebugLogModel`, `NetworkTrafficTower`, `DesktopIntentModel`, `UiPreferences`, QR image provider, and `SignerDiscoveryModel` (using `Utility`'s isolated signer worker) onto `SerialQueue`s.
   * Remove all remaining standalone `BackendExecutor` instances (`startup_executor`, `qr_executor`, `NodeModel`, `QmlInitExecutor`, `OptionsQmlModel`, `NetworkTrafficTower`, `SignerDiscoveryModel`).
5. **3.5 Switch orderly shutdown & remove `BackendExecutor::shutdownAll()`:**
   * Step 1 immediately disables UI input, stops accepting new requests, and **cancels waiting prompts** (`RuntimePromptService::close()`) *before* running `beforeInterrupt` `stop(done)` hooks (Settings, `desktopintent::transport::Service`, startup, preferences).
   * Step 2 posts `node.startShutdown()` on `Lane::Priority`.
   * Step 3 awaits `afterInterrupt` `stop(done)` hooks (`NodeModel`, wallets, services).
   * Step 4 runs `node.appShutdown()` on `QmlInitExecutor`'s lifecycle thread and joins it.
   * Step 5 exits Qt loop → destroys `QQmlApplicationEngine` and Models/Controllers → flushes `QCoreApplication::sendPostedEvents(nullptr, QEvent::DeferredDelete)` while `QCoreApplication` still exists → calls `executor.stopAndJoin()` → moves `Chain`/`Node`/`Init` into a final `std::thread`, destroys them in its body, then joins it.
   * Delete `BackendExecutor::shutdownAll()` and `startup::DrainWorkers()` / `DrainThenRetire`'s temporary cleanup executor once zero `BackendExecutor`s remain.
   * Ensure early startup returns and recoverable errors enforce the same precondition (cancel prompts + stop started features + flush `DeferredDelete` before `stopAndJoin()`). Keep native-fatal unsafe exit (`fatalExceptionExitsWithoutDraining`, `nativeFatalNeverStartsUnsafeTeardown`) on its separate fast-fail path.
* **Phase 3 Gate:** Combined **Core blocked on prompt + queued Settings write + shutdown** test passes; quit during `appInitMain()` never deadlocks `beforeInterrupt` Settings; payment-link forwarding receipts and listener resource cleanup complete before `Service::stop(done)` fires; normal quit, onboarding close/quit, wallet-loading quit/SIGINT, fatal child-process exit, delayed `thread_local` join, and `ENABLE_WALLET=ON/OFF` builds all pass.

---

### Phase 4 — Complete UI Boundaries & Extract `SendForm`

1. **4.1 Fix `WalletPageController`** (`qml/models/walletpagecontroller.h/.cpp`, `test/test_walletpagecontrollers.cpp`):
   * Add an irreversible `bool m_bound{false}` so destroying the first wallet cannot allow a second wallet to bind.
   * Keep `TransactionReviewController` explicit on `active` vs. `wallet->available()` (don't force it into `WalletPageController`'s single `stop()`).
2. **4.2 Complete `TransactionReviewInterface`** (`qml/models/transactionreviewinterface.h`, `qml/models/transactionreviewcontroller.h/.cpp`, `qml/components/ExternalSignerReviewActions.qml`, `qml/pages/wallet/SendTransactionReview.qml`):
   * Route external-signer approval, review capabilities (`currentTransactionIsImportedPsbt`, `currentTransactionCanSend`, `currentTransactionCanBroadcast`, `currentTransactionCanSignOnly`, `currentTransactionNeedsRevalidation`, `transactionFeePending`, `currentTransactionReviewMessage`), review file-export (`fileExportOperation`), and review error state through `TransactionReviewInterface` + `TransactionReviewController`. Keep separate sweep confirmations for preparation (`SendDraftController`) and broadcast (`SendTransactionReview.qml`).
   * Shrink `WalletQmlModel`'s exposed surface and tighten `test/test_walletqmlmodelsurface.cpp`.
3. **4.3 Narrow `SendDraftInterface` & extract `SendForm.qml`** (`qml/models/senddraftinterface.h`, `qml/models/senddraftcontroller.h/.cpp`, `qml/pages/wallet/SendCreate.qml`, `qml/components/SendForm.qml`):
   * Add `virtual bool draftReady() const = 0` (function), `void draftReadyChanged()` (signal), and `virtual void resetDraft() = 0` (rejected while preparation/import is active; immediate clear otherwise) to `SendDraftInterface`, plus `canPrepare` property on `SendDraftController`.
   * Move the preparation sweep-consent decision behind `SendDraftController` so `prepareRequested` routes through one controller action without bypassing `askSweepConsent()`.
   * Extract `SendForm.qml` (and decouple `SendFeeSelection.qml`, `SendRecipientCard.qml`, `SendRecipientEditor.qml`) with explicit typed inputs (`available`, `estimatedFeeSatoshi`, `estimatedFeeRate`, `feeEstimatePending`, `estimatedInputCount`, `sendTotalSatoshi`, `recipients`, `fees`, `coins`, `preparing`, `canPrepare`, `errorText`) and intent signals (`useMaximumRequested`, `prepareRequested`, `resetRequested`), keeping layout/focus/popups/imports in `SendCreate.qml`.
   * Retain `WalletReceiveModel` facade and direct `SettingsEditorModel`/`PaymentRequestEditorModel` contracts.
* **Phase 4 Gate:** Real-controller + fake-interface QML tests pass; `WalletQmlModelSurfaceTests` shrinks; `qml_lint` and `test/qml_api_schema.py` pass with zero new warnings; compact/regular layout, keyboard/Escape, and macOS native checks pass.

---

### Phase 5 — Delete Legacy Plumbing, Apply Renames & Sync Docs

1. **5.1 Delete legacy execution plumbing:** Remove `qml/backendexecutor.h/.cpp`, `TaskQueue` lease/cleanup/scope ledgers (`borrow`, `ResourceLease`, `reserveCleanup`, `activateCleanup`), `NodeModel::TrackedScheduler`, `BackendExecutor::shutdownAll()`, and `startup::DrainThenRetire`'s temporary cleanup executor. Retire obsolete mechanism-only tests (keeping all behavioral invariant tests).
2. **5.2 Apply the 20 accepted renames (Section 5)** as clean, standalone mechanical/scripted-diff commits at the end of the branch, updating headers, includes, `CMakeLists.txt`, `RegisterQmlTypes`, `cmake/qmltypes.h`, wallet-disabled stubs, `.qrc`, `test/qml_api_policy.json`, and tests together.
3. **5.3 Update docs & regenerate object model:** Update `doc/architecture.md`, `doc/integration-testing.md`, the PR description (honest fixed vs. partial claims), and regenerate the interactive object-model HTML from the final implemented code.
* **Phase 5 Gate:** Full CI matrix green on the exact final head.

---

# Part III — Verification Standard & Core Commands

### Pragmatic Proof Rules
* **Existing unmodified tests (~3,700+ C++/QML/integration/functional):** Full green CI pass across all supported configurations + diff review proving no weakened assertions, dropped rows, or widened suppressions.
* **New or rewritten tests (Phases 1–4):** Meaningful red/green verification (each fails for the expected defect without the fix and passes with it).
* **Explicitly preserve all 7 posted PR review findings:**
  1. Single shipped application executable (`wallet::ListDatabases` direct, no helper binary)
  2. Backup refuses loaded wallet databases/aliases under shared installation authority
  3. Pending send dismissal guard (`canDismiss`) and unchanged-draft clearing
  4. `QStringBuilder` temporaries materialized in fee-bump integration lambdas under sanitizers
  5. Qt 6.2 completion routing through stable `transactionReviewPopup` context
  6. QML mock numeric `balanceSatoshi` matches formatted balance (Qt 6.4)
  7. Popup focus does not jump back when user tabs during enter transition (`DialogPopup`, Qt 6.10)
* **Honest issue claims:** Keep the 10 partial public claims marked partial; never silently promote them to fixed.

### Primary Build & Test Commands

```sh
# 1. Full Release test build (C++ unit, bootstrap, QML, integration, daemon, automation app)
cmake -S . -B build -G Ninja \
  -DCMAKE_BUILD_TYPE=Release -DBUILD_APP_TESTS=ON -DBUILD_DAEMON=ON \
  -DENABLE_TEST_AUTOMATION=ON -DENABLE_IPC=OFF -DAPP_UNIT_TEST_CLASSES=
cmake --build build --target bitcoin-core-app bitcoind bitcoinqml_unit_tests bitcoinqml_bootstrap_tests bitcoinqml_qmltests bitcoinqml_integration

# 2. Run non-integration and integration suites (verify both wallet sending shards + Cairo DST QML rerun)
ctest --test-dir build --output-on-failure -LE '^integration$'
ctest --test-dir build --output-on-failure -L '^integration$'

# 3. Qt 6.10+ QML lint & moc API schema gate
cmake -S . -B build-lint -G Ninja \
  -DCMAKE_BUILD_TYPE=Release -DBUILD_APP_TESTS=ON -DENABLE_IPC=OFF \
  -DENABLE_QML_LINT=ON -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
cmake --build build-lint --target bitcoin-core-app qml_lint
python3 test/qml_api_schema.py --source-root . --build-dir build-lint --output reports/qml-api-schema.json
```

Also run: all 13 functional scripts (79+ scenarios including `interface_qml_console.py`), sanitizers (ASan/LSan/UBSan), clang-tidy, `ENABLE_WALLET=OFF` build, Qt 6.2 / 6.4 / 6.10, and Linux + native macOS checks.
