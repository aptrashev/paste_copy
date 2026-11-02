# ADR-0001: Tauri 2 + Rust + React/TypeScript

- Статус: принято для MVP
- Дата: 2026-11-02

## Контекст

Приложение постоянно работает в фоне, взаимодействует с clipboard/focus/input API macOS и Windows и одновременно требует общей быстрой UI-панели. Репозиторий пока не содержит кода, поэтому выбор должен уменьшить стоимость platform adapters, не создавая два независимых продукта.

## Решение

- Tauri 2 — desktop shell, окно, tray и lifecycle.
- Rust — domain/application core, platform adapters, storage, crypto и privileged IPC handlers.
- React + TypeScript — quick panel, settings и будущие collections.
- pnpm — frontend package manager; Cargo workspace — Rust packages.
- MVP — один процесс с bounded background workers. Frontend вызывает только versioned typed commands; прямого SQL, shell и произвольного filesystem access нет.

## Альтернативы

- **Electron:** зрелая экосистема и единый web stack, но постоянный Chromium/runtime увеличивает footprint, а системный код всё равно потребует native modules.
- **Flutter:** единый UI и хорошая производительность, но clipboard/focus/input adapters остаются custom plugins; команда получает второй основной язык рядом с native code.
- **Два native UI:** лучший платформенный контроль, но высокая стоимость реализации и риск расхождения поведения/моделей.
- **Rust-native UI toolkit:** меньше web surface, но выше риск зрелости/accessibility и сложнее нанять/поддерживать UI на старте.

## Последствия

Положительные: общий UI, memory profile лучше типичного Electron решения, platform/unsafe код локализован в Rust adapters, домен тестируется без GUI.

Отрицательные: поведение system webview различается; overlay/focus/fullscreen необходимо доказать spike на каждой ОС; Rust/UI boundary требует DTO/versioning; Tauri plugins не отменяют собственные adapters и runtime capability probe.

## Критерий пересмотра

Если фаза 0 покажет, что Tauri window не выполняет latency/focus/accessibility gates на macOS 13+ или Windows 10 22H2/11 и обход требует platform-specific UI, сравнить hybrid native panel или native shell до построения остального продукта.
