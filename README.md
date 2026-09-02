# tg-webproxy

[![Lint](https://github.com/DigneZzZ/tg-webproxy/actions/workflows/shellcheck.yml/badge.svg)](https://github.com/DigneZzZ/tg-webproxy/actions/workflows/shellcheck.yml)
[![License: GPL-3.0](https://img.shields.io/badge/license-GPL--3.0-blue.svg)](LICENSE)
![bash](https://img.shields.io/badge/bash-5%2B-4EAA25?logo=gnubash&logoColor=white)
![platform](https://img.shields.io/badge/Ubuntu%2022.04%2B%20%7C%20Debian%2012%2B-x86__64-E95420?logo=ubuntu&logoColor=white)

Установщик и утилита управления для **нового Telegram WEB-прокси**: MTProto-трафик прячется внутри обычного HTTPS к вашему домену, снаружи это просто сайт. Оборачивает официальный [telegramdesktop/tproxy-server](https://github.com/telegramdesktop/tproxy-server), добавляя проверки, генерацию секретов, сайт-прикрытие, живую панель, самообновление и режим разнесения relay и MTProxy по разным хостам.

> **TL;DR (English).** One-command installer for the new Telegram WEB proxy (MTProto over plain HTTPS/WebSocket to a real domain). Wraps the official `tproxy-server` with pre-flight checks, secret generation, a randomized cover site, a live TUI dashboard (`tgwebproxy`), self-update, and an optional split mode (Caddy + relay on one host, MTProxy on another over NetBird/WireGuard). Russian UI.

## Установка

```bash
bash <(wget -qO- https://raw.githubusercontent.com/DigneZzZ/tg-webproxy/main/tg-webproxy.sh)
```

Зеркало на GitHub Pages, тот же скрипт:

```bash
bash <(wget -qO- https://dignezzz.github.io/server/tg-webproxy.sh)
```

Что нужно заранее:

- VPS на **x86_64** с Ubuntu 22.04+ или Debian 12+, systemd, root;
- публичный IPv4 и свободные порты **80 и 443**;
- **отдельный домен** с A-записью на этот сервер. Домен остаётся обычным сайтом, никакого CDN и Cloudflare-проксирования перед ним.

Установщик спрашивает домен, почту для Let's Encrypt, секрет, тип подключения и тег спонсорского канала, всё с разумными значениями по Enter. Домен сверяется с IP сервера сразу при вводе. Сборка идёт в фоне с одной живой строкой прогресса, полный вывод пишется в `/var/log/tgwebproxy-install.log`. В конце печатаются ссылки вида `https://t.me/webproxy?server=…&secret=…`.

Все варианты, включая полностью неинтерактивную установку через переменные окружения, свой сайт-прикрытие и закрепление коммита апстрима: [docs/install-variants.md](docs/install-variants.md).

## Как это устроено

```
Интернет ──:443/:80──▶ Caddy (TLS, Let's Encrypt) ──▶ tproxy-server (relay, 127.0.0.1:8080)
                                                              │
                                                              ▼
                                                     MTProxy (127.0.0.1:2398) ──▶ Telegram DC
```

Публично открыты только 80 и 443. Relay ничего не расшифровывает: он превращает HTTPS-запросы или WebSocket клиента обратно в MTProto-поток и отдаёт его штатному MTProxy. Все три компонента ставятся апстримным `deploy/install.sh`, этот скрипт оборачивает его и дописывает то, чего апстрим не делает.

Что добавляет `tg-webproxy.sh` сверх апстрима:

- проверки перед установкой: root, архитектура, DNS, занятость портов, CDN перед доменом;
- уникальный сайт-заглушка на каждую установку, чтобы две инсталляции не совпадали по отпечатку;
- поддержка всех carrier-режимов (`https`, `https-lanes`, `websocket`, `websocket-lanes`, `all`) и синхронизация секретов с MTProxy;
- AD_TAG от @MTProxybot с готовой подсказкой, что ввести боту;
- фильтр логов Caddy, чтобы capability-токены не утекали в journald;
- обход двух багов апстрима, из-за которых свежая установка падает (подробности в [docs/upstream-notes.md](docs/upstream-notes.md));
- утилита `tgwebproxy` с живой панелью, самообновлением и split-режимом.

## Управление

После установки команда `tgwebproxy` без аргументов открывает меню с панелью состояния, которая обновляется сама:

```
╭────────────────────────────────────────────────────────────────────────────╮
│ TELEGRAM WEB PROXY                                        tgwebproxy 1.3.0 │
│ proxy.example.com                                               ● работает │
╰────────────────────────────────────────────────────────────────────────────╯

  СЛУЖБЫ                                ПОДКЛЮЧЕНИЯ
  ● caddy   ● relay   ● mtproxy         клиенты → caddy      12
  ● firewall   ● refresh                caddy → relay        12
  relay: healthz ok  ·  readyz ready    relay → mtproxy      12
                                        сессий 12  ·  стримов 31  ·  отказов 0

  ТРАФИК                                НАСТРОЙКИ
  relay     ↑ 254MB   ↓ 3GB             транспорт   all  (4 проф.)
  :443      вход 300MB   выход 3GB      AD_TAG      e3045596…0562
  ens3      сегодня 3 GiB / 300 MiB     relay       52a5feb  от 2026-09-03
            месяц   41 GiB / 4 GiB

  ПОДКЛЮЧЕНИЕ               ОБСЛУЖИВАНИЕ              УТИЛИТА
  1  Ссылки                 4  Перезапуск служб       7  Обновить утилиту
  2  Тип подключения        5  Журналы                8  Удалить
  3  AD_TAG                 6  Обновить relay         0  Выход

  s  Подробный статус       w  Живой монитор
```

Те же действия доступны командами: `tgwebproxy status`, `link`, `mode`, `adtag`, `logs`, `restart`, `update`, `version`, `self-update`, `uninstall`. Полный список и описание каждого экрана: [docs/cli.md](docs/cli.md).

## Обновления

- `tgwebproxy update` обновляет relay из репозитория апстрима с автоматическим откатом при неудаче.
- `tgwebproxy self-update` обновляет саму утилиту из опубликованного скрипта. Версия проверяется раз в сутки в фоне по трём источникам (raw GitHub, jsDelivr, GitHub Pages), берётся максимальная, о новой версии сообщает панель.
- Переустановка той же командой безопасна: домен, почта, секрет, режим и AD_TAG подставляются из прошлой установки, сайт в `/srv/tproxy-site` сохраняется.

## Split-режим: relay и MTProxy на разных хостах

По умолчанию всё живёт на одном сервере, и для одного VPS это лучший вариант. Если хочется держать несколько фронтов с разными доменами на один MTProxy, или менять фронт при блокировке, не трогая MTProxy, установщик умеет две роли поверх NetBird или WireGuard:

- **front**: Caddy и relay. На `127.0.0.1:2398` встаёт `systemd-socket-proxyd`, который уводит поток в туннель;
- **backend**: только MTProxy, порт открыт исключительно с адресов туннеля.

Как поднять, что печатает установщик и чем это отличается от одного хоста: [docs/split-mode.md](docs/split-mode.md).

## Если что-то пошло не так

Установщик сам разбирает причину падения: упавшие тесты апстрима, неготовый relay, ошибки apt, права на бинарник MTProxy, и печатает состояние служб и путь к журналу. Типовые ситуации и что с ними делать: [docs/troubleshooting.md](docs/troubleshooting.md).

## Безопасность и маскировка

- Секрет передаётся апстримному установщику через stdin и не попадает в список процессов.
- `info.env` с секретами лежит в `/opt/tgwebproxy` с правами 0600, `profiles.json` читается relay через systemd `LoadCredential`.
- Логи Caddy фильтруются: URI с capability, заголовки и полный IP клиента не пишутся.
- Не ставьте CDN перед доменом, не добавляйте `file_server`, `redir`, `respond` или path-scoped правила в Caddyfile, не включайте h3 и не объединяйте несколько таких доменов в один сертификат. Причины разобраны в [docs/upstream-notes.md](docs/upstream-notes.md).

## Удаление

```bash
tgwebproxy uninstall            # службы, юниты, конфиги, relay и MTProxy; сайт и сертификаты остаются
tgwebproxy uninstall --purge    # плюс сайт, сертификаты, Go и системные пользователи
```

Если до установки на сервере уже был Caddy, его Caddyfile и юнит восстанавливаются из бэкапа.

## Лицензия

[GPL-3.0](LICENSE). Апстримный `tproxy-server` и MTProxy распространяются по своим лицензиям.
