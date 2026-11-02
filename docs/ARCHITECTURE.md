# Архитектура приложения

## Принципы

- Общая domain/application логика не знает об AppKit или Win32.
- Любой привилегированный доступ находится в Rust; React вызывает только узкие типизированные commands.
- Политика приватности выполняется до persistence и indexing.
- Возможности определяются при запуске, а не выводятся только из имени ОС.
- Деградация явная: автоматическая вставка может стать copy-only, но выбранные данные не теряются.
- В MVP один процесс. Отдельный daemon допустим только после измеренного доказательства необходимости.

## Контекст и компоненты

```text
┌───────────────── macOS / Windows ───────────────────────┐
│ clipboard · foreground app · input · hotkey · secrets   │
└──────────────────────────┬───────────────────────────────┘
                           │ platform ports
                    ┌──────▼───────┐
                    │ OS adapters  │
                    └──────┬───────┘
                           │ bounded event channel
     ┌─────────────────────▼───────────────────────────┐
     │ Rust application core                           │
     │ CapturePipeline · PolicyEngine · Normalizer     │
     │ ClipService · QueryService · PasteOrchestrator  │
     │ RetentionService · CapabilityService            │
     └──────────────┬───────────────────┬───────────────┘
                    │ repositories      │ narrow IPC
          ┌─────────▼──────────┐  ┌────▼─────────────────┐
          │ SQLite + FTS5      │  │ Tauri + React UI     │
          │ encrypted blobs    │  │ panel/settings/tray  │
          └────────────────────┘  └──────────────────────┘
```

### Границы модулей

| Модуль | Ответственность | Не делает |
| --- | --- | --- |
| `domain` | IDs, clip/representation, policy decisions, errors, value objects. | I/O, UI, вызовы ОС. |
| `application` | Use cases, pipeline, orchestration, retention, capability degradation. | Не знает конкретных platform handles и SQL. |
| `ports` | Контракты clipboard, focus/input, repository, crypto, clock, logging. | Не содержит реализации. |
| `platform-*` | AppKit/Win32 и соответствующий system key store. | Не принимает продуктовые решения. |
| `storage` | Миграции, SQLCipher/SQLite, FTS, blob transactions, GC. | Не показывает UI. |
| `desktop-shell` | Tauri lifecycle, tray, window positioning, single instance, autostart/updater. | Не отдаёт frontend shell/fs/sql access. |
| `ui` | Quick panel, search, settings, permissions/capabilities. | Не читает clipboard и БД напрямую. |

Порты остаются абстрактными ради тестируемости и двух текущих OS adapters. Добавление любой новой ОС требует отдельного post-MVP ADR; текущая архитектура не содержит скрытых platform implementations.

## Порты

Конкретные async-типы уточняются при spike, но направление зависимости фиксировано:

```rust
trait ClipboardBackend: Send + Sync {
    fn watch(&self, sink: ClipboardEventSink) -> Result<WatchHandle>;
    fn read_snapshot(&self, change: ChangeId) -> Result<ClipboardSnapshot>;
    fn write_snapshot(
        &self,
        snapshot: &ClipboardSnapshot,
        mode: PasteMode,
    ) -> Result<WriteReceipt>;
}

trait ForegroundAppBackend: Send + Sync {
    fn capture_target(&self) -> Result<Option<WindowTarget>>;
    fn reactivate(&self, target: &WindowTarget) -> Result<ActivationOutcome>;
    fn source_app(&self) -> Result<Option<AppIdentity>>;
}

trait InputBackend: Send + Sync {
    fn inject_paste(&self, target: &WindowTarget) -> Result<PasteOutcome>;
}

trait CapabilityProbe: Send + Sync {
    fn capabilities(&self) -> PlatformCapabilities;
}
```

`PlatformCapabilities` содержит не один флаг, а независимо измеренные `capture`, `source_app`, `global_shortcut`, `reactivation`, `paste_injection`, `secret_store` с причиной недоступности и remediation action.

Frontend API — versioned DTO/commands вроде `query_clips`, `get_clip_preview`, `paste_clip`, `delete_clip`, `set_retention_protected`, `set_capture_paused`, `get_capabilities`. Произвольные SQL, shell, filesystem paths и сырой blob не экспортируются. Tauri capabilities разрешают только нужные commands для конкретных окон; общая модель ограничений описана в [официальной документации Tauri](https://v2.tauri.app/security/capabilities/).

Минимальный mutation-контракт защиты от retention:

```text
v1 set_retention_protected {
  clip_id: ClipId,
  protected: boolean,
  expected_revision: integer
} -> {
  clip_id: ClipId,
  retention_protected: boolean,
  revision: integer
}
```

Команда доступна только trusted quick-panel/settings windows через отдельную Tauri capability. Rust handler проверяет version, UUID, существование clip и `expected_revision`, затем меняет один флаг в транзакции; frontend не получает repository/SQL access и не может передать path или payload. Conflict возвращает typed error и заставляет UI перечитать metadata.

Карточка clip показывает состояние защиты и keyboard-accessible action `Защитить от автоочистки` / `Разрешить автоочистку`. Optimistic UI допустим только до ответа команды; при ошибке состояние откатывается и объявляется через accessible status. Удаление пользователем остаётся отдельным явным действием: `retention_protected` защищает только от автоматического retention/очистки по сроку.

## Capture flow

```text
OS change
  → ClipboardBackend event(change_id)
  → bounded queue / coalescing
  → PreReadGate: pause → available source identity/exclusion
    → capture capability + enumerated types/declared size
    → secret/transient markers visible without reading bytes
  → read all supported representations (timeout + size limits)
  → PostReadGate: payload-only markers/rules → actual size/content rules
  → SearchNormalizer: primary kind, safe preview, searchable text
  → IdentityBuilder: canonical representation-set digest / exact dedupe
  → DB transaction + encrypted temp blob → fsync → atomic rename
  → FTS update
  → UI event(metadata only)
```

- Capture сохраняет несколько representations одного clipboard item.
- `PreReadGate` выполняется до materialization payload. Он останавливает чтение при pause, применяет app exclusion, если adapter уже получил стабильный source identity, проверяет capability, перечисленные без чтения bytes representation types/declared sizes и сразу отклоняет видимые в этой metadata secret/transient markers.
- В `PostReadGate` остаются только markers и правила, которые невозможно определить без materialized representation, а также фактический размер и content rules. Они проверяются до создания preview, persistence, FTS или payload logging. Если source identity недоступна, exclusion не симулируется: действует platform limitation из support matrix.
- Переполненная очередь coalesces последовательные notifications по platform sequence; она не должна неограниченно расходовать память.
- Ошибка чтения одного representation не отменяет остальные, если item остаётся осмысленным.
- HTML не считается доверенным и не вставляется в UI DOM. Превью строится как plain text либо в будущем изолированным renderer.

### Self-write suppression и дедупликация

Search normalization и content identity — разные операции. `SearchNormalizer` может приводить текст к форме для поиска, но никогда не определяет равенство clips. `IdentityBuilder` сериализует канонический ordered set **всех сохраняемых representations**: для каждого элемента учитываются media/platform type и точные сохраняемые bytes, а для file list — канонически упорядоченные URI и сохраняемая metadata. Length-prefixing и фиксированная сортировка исключают неоднозначность. Поэтому одинаковый plain text с разными HTML/RTF имеет разные identity digests.

Два clips объединяются только при совпадении digest, количества, типов и полного канонического набора representations; частично прочитанный item не объединяется с полным. Безопасный merge меняет только `last_copied_at/copy_count`: `retention_protected` и исходные representations/source metadata не перезаписываются повторным capture.

`write_snapshot` возвращает `WriteReceipt` с доступным platform sequence/change ID и `identity_digest`. Monitor подавляет только совпадающую собственную запись по receipt + digest. Timeout ограничивает жизнь receipt, но не является единственным критерием.

## Paste flow

```text
hotkey → capture current WindowTarget → show panel
select clip → validate representations → hide panel
→ write clipboard (rich or plain) → receive WriteReceipt
→ reactivate target → inject platform paste shortcut
→ success
   └─ failure/unsupported: keep clipboard value + show copy-only instruction
```

Target захватывается до передачи focus панели. Paste pipeline не восстанавливает предыдущий clipboard в MVP: автоматическое восстановление создаёт гонку с действиями пользователя и требует отдельного ADR/исследования.

## Хранение данных

### Логическая схема

| Таблица | Основные поля |
| --- | --- |
| `clips` | `id`, `created_at`, `last_copied_at`, `source_app_id`, `primary_kind`, `identity_digest`, `preview`, `size_bytes`, `sensitivity`, `expires_at`, `retention_protected`, `copy_count` |
| `representations` | `id`, `clip_id`, `media_type`, `platform_type`, `storage_kind`, `inline_ciphertext`/`blob_id`, `size_bytes` |
| `blobs` | opaque `id`, keyed `address_digest`, `encrypted_path`, `size_bytes`, `ref_count`, `created_at` |
| `clip_search` | FTS5 external-content index разрешённого текста и metadata |
| `collections` | `id`, `name`, `created_at`, `updated_at` (P1) |
| `collection_items` | `collection_id`, `clip_id`, `position` (P1) |
| `stack_items` | `stack_id`, `clip_id`, `position`, `state` (P1) |
| `schema_migrations` | `version`, `checksum`, `applied_at` |

Мелкие representations хранятся inline, крупные — content-addressed blobs. Адрес внутри vault — keyed BLAKE3/HMAC digest от содержимого с отдельным subkey, производным от master key; filename не содержит unkeyed plaintext hash. Альтернативная допустимая реализация — opaque random ID с зашифрованным mapping в БД. Порог является измеряемой настройкой сборки, не пользовательской опцией. File list хранит URI/путь и metadata; исходный файл не копируется.

### Транзакции и миграции

1. Blob шифруется во временный файл в каталоге vault.
2. Выполняются `fsync` и atomic rename на том же filesystem.
3. DB transaction добавляет clip/representation/ref count и FTS.
4. При rollback временный/orphaned blob удаляется reconciliation-задачей.

Миграции только вперёд, versioned и с checksum. Перед миграцией создаётся согласованный encrypted backup/копия, если хватает места. Downgrade, изменяющий схему назад, не поддерживается; старый binary должен отказаться открывать более новую схему без записи. Retention удаляет только clips без `retention_protected`, декрементирует ref count и отдельной idempotent GC удаляет blobs с нулевой ссылкой. Полноценные favorites/collections остаются P1 и не меняют смысл этого MVP-флага.

## Безопасность и приватность

- База, включая FTS, шифруется SQLCipher; внешние blobs — XChaCha20-Poly1305 с уникальным nonce.
- Имена/IDs blob-файлов не раскрывают unkeyed hash plaintext: content addressing использует отдельный keyed digest subkey либо opaque ID с encrypted mapping.
- Случайный master key хранится в macOS Keychain или Windows DPAPI/Credential Manager. Config хранит только key identifier и версию envelope.
- Если secret store/vault недоступен, persistent capture выключен. Незаметного перехода на plaintext нет.
- Pause, доступные source exclusions и metadata-visible secret/transient markers применяются в `PreReadGate` до чтения bytes. Только payload-dependent markers и content rules выполняются после materialization, но до persistence/index/preview. Эвристики токенов опциональны.
- Любая конфигурация, способная принять реальные пользовательские данные, использует SQLCipher и encrypted blobs. Упрощённый storage компилируется/включается только в изолированном automated-test profile, принимает только синтетические fixtures и отказывает при попытке открыть пользовательский vault или production path.
- Production logging содержит event IDs, durations, sizes/buckets и error classes, но не payload, query, key material, preview и полный path.
- IPC DTO не содержит ключей и лишних representations. CSP запрещает remote content; link preview не загружает сеть в MVP.
- Release artifacts и updater подписываются; direct installers — notarized/signed `.dmg` и code-signed `.msi`. Tauri updater требует подпись обновлений; см. [официальное руководство](https://v2.tauri.app/plugin/updater/).
- Threat model перед beta отдельно проверяет malicious clipboard payload, HTML/script preview, symlink/path traversal в blob store, IPC privilege escalation, key loss и forensic leftovers.

Подробное решение: [ADR-0002](adr/0002-encrypted-local-storage.md).

## Lifecycle

1. Single-instance guard создаётся до инициализации БД; второй запуск активирует панель существующего процесса. Официальный Tauri plugin описан [здесь](https://v2.tauri.app/plugin/single-instance/).
2. Открываются config и secret store, проверяется schema, затем запускаются workers.
3. Capability probe регистрирует monitor/hotkey и публикует UI состояние. Для Tauri global shortcut используется [официальный plugin](https://v2.tauri.app/plugin/global-shortcut/); каждый macOS/Windows adapter всё равно подтверждает возможность runtime probe.
4. Закрытие окна скрывает панель; Quit из tray завершает monitor, дренирует допустимые транзакции и закрывает vault.
5. Autostart только opt-in через системный механизм/официальный [Tauri plugin](https://v2.tauri.app/plugin/autostart/). Обновление не должно самовольно менять выбор.

## Отказы и поведение

| Отказ | Поведение |
| --- | --- |
| Clipboard временно занят/owner исчез | Ограниченный retry с jitter; item пропускается, monitor продолжает работу. |
| Payload превышает лимит | Metadata-only отказ с локализованным статусом; содержимое не пишется частично. |
| Vault/secret store заблокирован | Capture off, история закрыта, remediation в настройках. |
| БД повреждена/миграция не прошла | Read-only/recovery mode, сохранить исходный файл, без destructive reset. |
| Hotkey конфликтует | Панель настроек предлагает другой shortcut; tray остаётся доступен. |
| Paste injection запрещена | Clipboard заполнен, copy-only notification; история не теряется. |
| Исходный file URI исчез | Карточка остаётся с пометкой; попытка paste объясняет отсутствие. |
| Blob отсутствует/не проходит AEAD | Clip помечается damaged, payload не отдаётся; diagnostics без данных. |
| Очередь перегружена | Coalescing/backpressure и счётчик diagnostics; UI остаётся responsive. |

Платформенные детали и источники находятся в [PLATFORM_SUPPORT.md](PLATFORM_SUPPORT.md).
