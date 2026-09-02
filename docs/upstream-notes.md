# Апстрим: что он делает и что мы обходим

Установщик не переписывает [telegramdesktop/tproxy-server](https://github.com/telegramdesktop/tproxy-server), а запускает его `deploy/install.sh` как есть. Ниже то, что полезно знать, читая код обёртки.

## Что делает `deploy/install.sh`

1. Проверяет root, x86_64, домен, почту, секрет (32 hex или `dd`+32 hex). Секрет читает из stdin, если не передан флагом; обёртка всегда передаёт его через stdin, чтобы он не светился в `ps`.
2. Ставит `ca-certificates curl nftables`, скачивает Caddy с проверкой SHA-512 в `/usr/local/bin/caddy`.
3. Запускает `deploy/install-mtproxy.sh`: `build-essential`, сборка MTProxy закреплённого коммита под пользователем `mtproxy`, загрузка `proxy-secret` и `proxy-multi.conf` с `core.telegram.org`.
4. Скачивает Go, гоняет `go test ./...`, собирает relay в `/usr/local/bin/tproxy-server`.
5. Копирует сайт в `/srv/tproxy-site`, если там ещё нет `index.html`; иначе сохраняет существующий.
6. Пишет `config.json`, `profiles.json` (0400, читается relay через `LoadCredential`), `mtproxy.env`, Caddyfile с бэкапом прежнего, юниты systemd, включает всё и ждёт `readyz` 20 секунд.

`deploy/update-relay.sh` собирает новый relay, проверяет его `-check`, подменяет бинарник и откатывается, если healthz не отвечает. `refresh-mtproxy-config` раз в сутки обновляет `proxy-multi.conf` и делает `try-restart mtproxy`; `mtproxy.env` он не трогает.

## Баги апстрима, которые обходит обёртка

**Тест, зависящий от umask.** `install.sh` ставит `umask 077` и затем гоняет тесты. `TestLoadAcceptsSystemdCredentialReadPermissions` создаёт файл с правами 0444, под этой umask получает 0400, проверка «лишних прав» не срабатывает, тест падает на любой свежей установке коммита 52a5feb. Обёртка пропускает ровно этот тест через `GOFLAGS=-skip=…`; `go build` неизвестные флаги игнорирует. Правильный фикс на стороне апстрима: явный `os.Chmod` после `WriteFile`.

**Бинарник MTProxy без прав на выполнение.** `install-mtproxy.sh` собирает MTProxy через `runuser -u mtproxy -- make`, пока родитель держит `umask 077`. Каталог `objs/` выходит 0700, затем `chown -R root:root`, и пользователь службы не может даже войти в каталог: systemd отвечает `203/EXEC`. Повторная установка не помогает, потому что для root бинарник выглядит исполняемым и сборка пропускается. Обёртка подставляет в `PATH` шим `runuser` с `umask 022` на время сборки, чинит права перед переустановкой и, если служба всё же упала с 203, правит права и перезапускает её вместо выхода.

**Хрупкость при переустановке.** Апстрим переписывает `mtproxy.env`, оставляя один `MTPROXY_SECRET`, и перезапускает MTProxy. Если наш drop-in ещё ссылается на `${MTPROXY_SECRET2}` или `${MTPROXY_TAG}`, systemd подставляет пустые аргументы, MTProxy выходит на `-S ""`, readyz не поднимается. Обёртка снимает drop-in до запуска апстрима и пишет заново после.

## Ограничения, которые обёртка соблюдает намеренно

- **Backend только loopback.** Проверка конфигурации relay требует loopback-адрес, юнит закрыт `IPAddressAllow=localhost`. Split-режим не патчит это, а ставит `systemd-socket-proxyd` на loopback.
- **Один сертификат на домен, никакого CDN.** Relay принимает `X-Forwarded-For` только с ровно одним адресом; CDN терминирует TLS и видит capability; несколько доменов в одном сертификате связываются через Certificate Transparency.
- **Caddyfile без `file_server`, `redir`, `respond`, path-scoped правил, `request_body max_size` и h3.** Любой из них даёт второй набор заголовков или границу, по которой активный пробинг отличает прокси от сайта. Таймауты `read_body 60s` и `response_header_timeout 40s` рассчитаны на long-poll relay.
- **Логи Caddy.** В Caddyfile апстрима нет `log`, но `http.log.error` активен всегда и при каждом рестарте relay пишет запрос с URI, где лежит capability. Обёртка добавляет глобальный `log default` с фильтром `request>uri delete`, `request>headers delete`, `request>remote_ip ip_mask`.
- **Секрет `dd`.** Префикс клиентский: в `profiles.json` он остаётся, в MTProxy передаётся секрет без него. Префикс `ee` для WEB-прокси не поддерживается.
- **Несколько профилей на один MTProxy.** Апстрим требует уникальные имя и capability, но не backend. Отдельный MTProxy на профиль нужен только ради раздельных квот.
