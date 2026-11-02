# ADR-0003: Linux отложен после MVP

- Статус: принято для MVP
- Дата: 2026-11-02

## Контекст

Clipboard manager требует фонового наблюдения, global shortcut, определения source/target, возврата focus и input injection. Добавление Linux сейчас расширило бы scope отдельными X11/Wayland backends, packaging и release/test matrix до того, как основной сценарий доказан на macOS и Windows.

Пользователь решил выпустить MVP только для macOS 13+ и Windows 10 22H2/11, а к Linux вернуться после стабилизации продукта.

## Решение

- Linux полностью исключён из MVP и не заявляется как поддерживаемая платформа.
- В MVP нет Linux adapters, условно включаемых Linux implementations, packages, CI jobs, release artifacts или support matrix.
- Общие platform ports сохраняются ради тестируемости и двух текущих macOS/Windows adapters; они не являются обещанием или скрытой реализацией Linux.
- Документация может упоминать Linux только как явно отложенную post-MVP платформу.
- Возврат к Linux требует нового ADR, актуальных platform spikes, отдельной threat/privacy оценки и release/test matrix. Старые предположения о X11 или Wayland не переносятся автоматически в новое решение.

## Альтернативы

- **Включить Linux/X11 в MVP:** отклонено из-за дополнительного backend, packaging и real-device matrix.
- **Сделать ограниченный Wayland Tier:** отклонено для MVP, потому что capability-dependent продукт усложняет support contract до проверки core flow на двух основных ОС.
- **Публиковать неподдерживаемую community build:** отклонено, потому что создаёт неявные ожидания по приватности, обновлениям и совместимости.

## Последствия

Положительные: меньше платформенных рисков, быстрее проверяется основной сценарий, signing/release усилия сосредоточены на двух системах.

Отрицательные: Linux-пользователи не получают MVP; будущая поддержка потребует нового исследования и может повлиять на UI, permissions и delivery architecture.

## Критерий пересмотра

Linux можно вернуть в roadmap только после отдельного решения о целевых distro/session, исследования актуальных clipboard/input APIs, работающих platform prototypes и оценки стоимости самостоятельной CI/release/support matrix.
