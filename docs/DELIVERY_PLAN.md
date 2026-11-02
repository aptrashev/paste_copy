# План разработки и проверки

## Этапы

| Этап | Результат | Exit criteria |
| --- | --- | --- |
| 0. Platform spikes | Измеренные clipboard/paste/window прототипы на macOS 13+, Windows 10 22H2 и Windows 11. | Round-trip text/HTML/RTF/image/file fixtures; зафиксированы capabilities, permissions, payload limits, signing/notarization path и open blockers. |
| 1. Foundation | Cargo workspace, Tauri shell, React/TS, pnpm lockfile, миграции, platform ports, CI matrix. | Воспроизводимые install/lint/typecheck/test/build; пустое приложение single-instance; macOS/Windows adapters заменяемы fakes; test-only storage изолирован и принимает только synthetic fixtures. |
| 2. Vertical slice | Text capture → encrypted storage → history/search → hotkey → paste/fallback. | `FR-002/003/005/006/007/011/012` для text проходят на обеих ОС; latency baseline записан; real-data path не запускается без encryption. |
| 3. Multi-format | HTML/RTF/images/URLs/file lists, safe previews, plain-text mode. | `FR-001/004/007`; fixture round-trip без исполнения HTML; большие payload не блокируют UI. |
| 4. Privacy and durability | Pre/post-read gates, pause, exclusions, secret markers, versioned `set_retention_protected` IPC + UI action, retention/GC, recovery, encryption/key lifecycle. | `FR-002/008/009/010`, `NFR-003/004/007`; set/unset проходит component, integration и E2E; security review без P0/P1. |
| 5. Product polish | Tray, settings, permissions UX, autostart, accessibility, diagnostics, signed `.dmg`/`.msi` и updater. | Все MVP requirements, real-device matrix и upgrade test; copy-only fallback понятен; installers подписаны и проверены на чистых системах. |
| 6. P1 | Favorites, collections, stack. | Отдельные acceptance criteria после стабильного MVP; не задерживает MVP. |

Новые OS targets, cloud sync, mobile, OCR, sharing и MCP/AI начинаются только с отдельного product/architecture decision после MVP.

## Трассировка MVP

| Requirement | Компоненты | Основная проверка |
| --- | --- | --- |
| FR-001 | `ClipboardBackend`, `CapturePipeline`, `RepresentationRepository` | Adapter contract + multi-format round-trip fixtures |
| FR-002 | `PreReadGate`, `PostReadGate`, `CapturePipeline`, storage/logging | Metadata-visible markers rejected before byte read; payload-only rules rejected before preview/DB/FTS/logs |
| FR-003 | `IdentityBuilder`, dedupe repository, `WriteReceipt` registry | Full representation-set fixtures, different-HTML case, recopy + self-write race |
| FR-004 | `ClipService`, query DTO, React history | Integration + UI states for missing source file |
| FR-005 | hotkey adapter, window/focus shell, React panel | Keyboard-only E2E on macOS 13+ and Windows 10 22H2/11, including multi-monitor |
| FR-006 | `QueryService`, FTS5, filters | Relevance fixtures + 50k performance benchmark |
| FR-007 | `PasteOrchestrator`, clipboard/focus/input adapters | Rich/plain round-trip; macOS permission, Windows UIPI and copy-only cases |
| FR-008 | `RetentionService`, versioned `set_retention_protected` handler, card action/state, repositories, blob GC | Component set/unset/error rollback; IPC auth/revision integration; E2E set → survive retention → unset → expire; crash/restart idempotency |
| FR-009 | `PolicyEngine`, monitor state, tray/UI | Pause-before-read integration test |
| FR-010 | source identity adapter, exclusion rules | Excluded helper app on both OS families + unsupported-capability UX |
| FR-011 | `CapabilityProbe`, settings/remediation UI | macOS Accessibility/key-store and Windows UIPI/key-store denial/recovery matrix |
| FR-012 | Tauri lifecycle, tray, single-instance, autostart | Second-launch and login/reboot manual automation |
| NFR-001 | UI shell, `QueryService`, benchmark dataset | p95 panel/search benchmark with recorded hardware |
| NFR-002 | bounded channels, workers, blob/storage adapters | Oversize/load test + UI responsiveness trace |
| NFR-003 | crypto/key adapters, SQLCipher, blob store, isolated test profile | At-rest inspection; locked key store; key rotation; release-config bypass rejection; non-fixture rejection in test profile |
| NFR-004 | structured logging/crash config | Canary secrets scan across logs/diagnostics/crash payload |
| NFR-005 | React UI/window shell | Keyboard, focus, screen reader, contrast/reduced motion audit |
| NFR-006 | supervisor/application services | Fault injection per adapter; monitor remains live |
| NFR-007 | migration/recovery service | Upgrade fixtures and failed-migration preservation test |
| NFR-008 | CI/release/updater | `.dmg`/`.msi` signing verification and tampered-update rejection |

## Стратегия тестирования

### Быстрый слой

- Rust unit/property tests: search normalization, canonical representation-set identity, exact dedupe, pre/post-read policy precedence, protected retention, migration transforms, search ranking.
- TypeScript component tests: focus model, keyboard navigation, capability/error states, retention action labels/state и rollback при typed IPC error.
- Общие contract suites запускаются против fake adapter и реализаций macOS/Windows.
- Build/config tests доказывают, что test-only storage отсутствует в production/release и отказывает всем данным, кроме synthetic fixtures.

### Интеграционный слой

Fixture helper app на каждой поддерживаемой ОС публикует text, одинаковый text с разными HTML/RTF, PNG, file list, delayed/owner-provided data и secret markers, затем проверяет capture/round-trip. Отдельные сценарии: pause/source exclusion и metadata-visible marker rejection до byte read; payload-only marker rejection до preview/persistence; authorized set/unset, stale revision и unauthorized IPC; E2E set → retention → unset → expiration; self-write race, clipboard owner death, payload limit, locked vault, missing file, corrupted blob, hotkey conflict и queue overload.

### End-to-end и ручная матрица

- Реальные macOS 13+ машины и чистая установка подписанного/notarized `.dmg`.
- Реальные Windows 10 22H2 и Windows 11 машины и чистая установка code-signed `.msi`.
- Multi-monitor, fullscreen, sleep/resume, login autostart, elevated Windows target, denied macOS Accessibility и locked/unavailable system key store.
- Upgrade с последнего публичного schema/release; clean uninstall проверяет только документированную политику сохранения данных.

## Performance и reliability gates

- Набор 50 000 записей разных типов и размеров, фиксированный seed.
- Panel visible p95 ≤ 200 ms; query p95 ≤ 100 ms на зафиксированном эталонном железе.
- UI thread не выполняет payload I/O/crypto/FTS; oversize fixture не вызывает frame stall или unbounded RSS.
- 8-часовой soak с частыми clipboard changes не теряет monitor и не растит очередь/handles без границ.
- Crash points вокруг temp write, rename, DB commit и GC доказывают recoverability.

Числовые budgets idle CPU, memory, disk quota и maximum item size утверждаются после фазы 0 и затем становятся release gates.

## Риски

| Риск | Вероятность/влияние | Снижение | Триггер пересмотра |
| --- | --- | --- | --- |
| Потеря focus или запрещённая input injection | Средняя/высокое | Захват target до панели, platform outcomes, copy-only | Частота fallback делает core job неприемлемым |
| Секрет попадает в историю | Средняя/критическое | Pre-persistence policy, system markers, exclusions, no payload logs, security tests | Любая подтверждённая утечка — stop-ship |
| Повреждение/потеря ключа закрывает историю | Низкая/высокое | System key store, recovery UX, backup strategy decision | Key-store instability на поддерживаемой ОС |
| Multi-format semantics отличаются между приложениями | Высокая/среднее | Fixture helper + real app compatibility matrix | Новый format требует domain change |
| Windows 10 22H2 требует отдельной legacy-совместимости | Средняя/среднее | Отдельная real-device/installer matrix, запрет зависимости только от Windows 11 APIs | Стоимость поддержки превышает подтверждённую аудиторию; изменение требует ADR |
| Signing/notarization или installer reputation блокируют поставку | Средняя/высокое | Ранний signing spike, CI verification, clean-machine install | Установщик не проходит OS trust flow |
| Webview/Tauri окно плохо ведёт себя поверх fullscreen | Средняя/среднее | Ранний window/focus spike на обеих ОС | Нельзя выполнить p95/core flow |
| Storage растёт без границ | Средняя/среднее | Retention, quota, ref-count GC, metrics without payload | Soak превышает утверждённый disk budget |

## Definition of Done для MVP

- Все `FR-*` и `NFR-*` из [PRODUCT.md](PRODUCT.md) имеют evidence в трассировке и пройденный gate либо явно утверждённое platform limitation.
- Нет известных P0/P1; focused tests, полный relevant suite, lint, typecheck и release build проходят.
- Реальные матрицы macOS 13+, Windows 10 22H2 и Windows 11 пройдены.
- Security review подтверждает pre-persistence filtering, encrypted at rest, отсутствие payload в diagnostics и невозможность test-only bypass в release configuration.
- Подписанный/notarized `.dmg` и code-signed `.msi` проходят clean-machine install и upgrade; повреждённое обновление отклоняется.
- Миграция с предыдущего release и recovery от прерванной записи проверены.
- Документация, permissions UX, support matrix и fallback соответствуют фактической сборке.
- В release artifacts нет secrets, debug entitlements, неподписанных binaries и случайных dev paths.

## Правила изменения архитектуры

Изменение stack, процесса/daemon boundary, encryption model, target OS или delivery channel требует нового ADR. Изменение одного adapter без влияния на public contract может проходить обычный review с обновлением platform tests.
