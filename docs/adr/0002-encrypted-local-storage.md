# ADR-0002: Зашифрованное локальное хранилище

- Статус: принято для любой сборки с реальными данными
- Дата: 2026-11-02

## Контекст

История clipboard может содержать персональные данные и секреты, даже если приложение фильтрует известные markers. Plain SQLite, FTS index или изображения на диске сделали бы защиту частичной.

## Решение

- SQLite/FTS шифруются SQLCipher.
- Большие payload хранятся content-addressed в отдельном blob store и шифруются XChaCha20-Poly1305 с уникальным nonce. Внутренний адрес строится keyed BLAKE3/HMAC с отдельным производным subkey; filename/ID не содержит unkeyed hash plaintext. Допустим opaque random ID с encrypted mapping в БД.
- Случайный master key хранится в macOS Keychain или Windows DPAPI/Credential Manager; config содержит только key ID/version.
- Запись blob: encrypted temp → fsync → atomic rename → DB transaction. Reconciliation убирает orphaned files.
- При недоступном key store persistent capture выключается. Fallback на plaintext запрещён.
- Упрощённый storage допустим только в изолированном automated-test profile с синтетическими fixtures. Он отсутствует в production/release configuration, отказывает при попытке открыть пользовательский vault или production path и не принимает произвольный clipboard input.
- Payload, query и key material не логируются.

Конкретные crates, параметры KDF/envelope и ротация ключа выбираются после threat model и spike; они не должны менять application contracts.

## Альтернативы

- **Только шифрование диска ОС:** полезно, но не контролируется приложением и не защищает от чтения файлов другим процессом в разблокированной сессии.
- **Plain SQLite + encrypted blobs:** оставляет previews и FTS в открытом виде.
- **Все payload inline в SQLCipher:** проще backup, но плохо для больших images и транзакционного I/O.
- **Шифрование после MVP:** отклонено, потому что создаёт миграцию реальных чувствительных данных. Упрощение допустимо только для автоматических synthetic fixtures в изолированном test profile.

## Последствия

Положительные: единая at-rest модель и отсутствие открытого поискового индекса; blobs можно дедуплицировать внутри одного vault по keyed digest, не публикуя unkeyed plaintext hash в filename.

Отрицательные: key loss может сделать vault невосстановимым; SQLCipher усложняет packaging; encrypted blobs требуют nonce/ref-count/reconciliation; keyed digest всё ещё раскрывает равенство объектов внутри доступного локального vault, поэтому metadata остаётся под защитой SQLCipher и filesystem permissions.

## Критерий пересмотра

Cloud sync, multi-user sharing, app-store sandbox key restrictions или требование восстановить vault без system account потребуют отдельного ADR и иной key hierarchy.
