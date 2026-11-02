# Поддержка платформ

## Публичный контракт MVP

- macOS 13 и новее.
- Windows 10 22H2 и Windows 11.
- Функция считается поддержанной после автоматических adapter contract tests и ручного end-to-end прогона на реальной машине каждой поддерживаемой версии.
- MVP распространяется прямыми подписанными установщиками: notarized/signed `.dmg` для macOS и code-signed `.msi` для Windows. App stores не входят в MVP.

## Матрица возможностей

| Возможность | macOS 13+ | Windows 10 22H2/11 |
| --- | --- | --- |
| Фоновое обнаружение изменений | Polling `NSPasteboard.changeCount` с измеренным адаптивным интервалом | `AddClipboardFormatListener` и sequence number |
| Несколько representations | `NSPasteboard` types | Windows clipboard formats |
| Приложение-источник | Foreground app snapshot и стабильный bundle identifier, где доступны | Foreground window/process identity |
| Global shortcut | Runtime registration | Runtime registration |
| Возврат focus | Best effort | Best effort с ограничениями foreground activation |
| Автовставка | Требует Accessibility permission | `SendInput` только в процесс равного или меньшего integrity level |
| App exclusions | По стабильному bundle identifier | По стабильной process identity |
| Безопасный fallback | Clipboard + ручной `Cmd+V` | Clipboard + ручной `Ctrl+V` |
| Хранилище ключа | macOS Keychain | Windows DPAPI/Credential Manager |
| Установщик | Подписанный и notarized `.dmg` | Code-signed `.msi` |

## macOS 13+

- Clipboard adapter строится на [`NSPasteboard`](https://developer.apple.com/documentation/appkit/nspasteboard). Spike измеряет polling interval, idle CPU и потерю краткоживущих owner-provided representations.
- Программная отправка paste требует доверия Accessibility; проверка выполняется через [`AXIsProcessTrustedWithOptions`](https://developer.apple.com/documentation/applicationservices/1459186-axisprocesstrustedwithoptions). Отказ не блокирует историю и copy-only fallback.
- Concealed/transient pasteboard types разных приложений обрабатываются как best effort. Нельзя обещать автоматическое обнаружение любого секрета.
- Release artifact подписывается Developer ID, notarize и проверяется на чистой поддерживаемой системе. Mac App Store требует отдельного ADR и spike по sandbox/entitlements/background behavior.

## Windows 10 22H2 и Windows 11

- Monitor использует clipboard listener и sequence number; официальный обзор listener APIs находится в [Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/dataxchg/using-the-clipboard).
- Adapter уважает Windows formats, запрещающие включение содержимого в историю/monitor processing; точное поведение проверяется fixture-приложением на обеих поддерживаемых линиях Windows.
- [`SendInput`](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-sendinput) подчиняется UIPI и может отправлять input только процессам равного или меньшего integrity level. Менеджер не запускается постоянно от администратора ради обхода; используется copy-only fallback.
- Code-signed `.msi` проверяется на чистых Windows 10 22H2 и Windows 11. Microsoft Store требует отдельного ADR и не является каналом MVP.

## Отложенная платформа

Linux не входит в MVP: проект не реализует Linux adapters, не создаёт Linux packages или CI jobs и не заявляет поддержку X11/Wayland. Возврат к Linux требует нового ADR, исследования актуальных системных API и отдельной release/test matrix; подробности зафиксированы в [ADR-0003](adr/0003-linux-deferred.md).

## Capability state machine

Для каждой системной функции используется состояние `unknown → probing → available | unavailable(reason) | permission_required(action) | degraded(reason)`. Повторный probe запускается после изменения permissions/session и при resume, но не в бесконечном цикле. UI и diagnostics используют этот объект как единственный источник истины.
