# Диагностика

Установщик после сбоя сам ищет причину в `/var/log/tgwebproxy-install.log` и журнале systemd, печатает её, состояние служб и путь к журналу. Ниже то, что встречается чаще всего.

## Установка

**«Этот сервер не подходит: дата-центры Telegram недоступны», код выхода 3.**
Проверка идёт сразу после выбора роли, до вопросов и установки пакетов: TCP на :8888 к пяти адресам дата-центров, куда MTProxy ходит через middle-proxy. Если ни один не отвечает, MTProxy на этом сервере не заработает, что бы ни стояло дальше; типичный случай — хостинг в РФ. Возьмите сервер в другой стране или сети. Такой хост годится только на роль `front` в [split-режиме](split-mode.md), где MTProxy живёт на другом сервере. Проверить вручную:

```bash
timeout 3 bash -c '</dev/tcp/149.154.175.50/8888' && echo ok
```

`TGWP_SKIP_REACH=1` отключает проверку, если вы точно знаете, что делаете.

**«Официальный установщик завершился с ошибкой», в журнале `--- FAIL: TestLoad…`.**
Упали unit-тесты апстрима. Один из них зависит от umask и падает на любой свежей установке, скрипт его пропускает через `GOFLAGS`. Если упал другой тест, смотрите строку `_test.go:` в журнале: это регрессия апстрима, закрепите рабочий коммит через `TGWP_REF`.

**`mtproxy.service: status=203/EXEC`.**
systemd не может выполнить `/opt/MTProxy/objs/bin/mtproto-proxy` от пользователя mtproxy: каталог `objs/` собран с правами 0700. Скрипт чинит это сам тремя способами. Вручную:

```bash
chmod -R a+rX /opt/MTProxy && systemctl restart mtproxy && curl -fsS http://127.0.0.1:8081/readyz
```

**`tproxy-server did not become ready`.**
Relay не увидел MTProxy на 127.0.0.1:2398. `journalctl -u mtproxy -n 50`: чаще всего MTProxy не стартует из-за секрета не 32 hex, пустого `-S` или недоступного `core.telegram.org` при загрузке `proxy-secret`.

**`curl: (22) The requested URL returned error: 404` сразу после сборки MTProxy.**
Сеть сервера блокирует `core.telegram.org` и подставляет 404-заглушку, а апстримный `install-mtproxy.sh` качает оттуда `proxy-secret` и список дата-центров без вариантов. Начиная с 1.3.2 установщик проверяет это первым делом и печатает инструкцию; до сборки дело не доходит. Обход: скачайте оба файла на любой машине с доступом и положите их в `/opt/tgwebproxy/tg/`, либо укажите зеркало `TGWP_TG_MIRROR=https://host/path`, где лежат `getProxySecret` и `getProxyConfig`. Шим `curl` в `PATH` подставит их апстримному скрипту. Проверьте заодно, что дата-центр Telegram доступен: `timeout 3 bash -c '</dev/tcp/149.154.175.50/8888'`. Если и он закрыт, MTProxy на этом сервере работать не будет.

```bash
curl -o proxy-secret https://core.telegram.org/getProxySecret
curl -o proxy-multi.conf https://core.telegram.org/getProxyConfig
scp proxy-secret proxy-multi.conf root@server:/opt/tgwebproxy/tg/
```

Суточный таймер `refresh-mtproxy-config` в такой сети тоже не сможет обновлять конфигурацию, MTProxy продолжит работать со старой.

**Домен не резолвится или указывает на другой IP.**
Проверка идёт при вводе домена. Добавьте A-запись на IP сервера и дождитесь обновления DNS; продолжать без записи можно, но Let's Encrypt не выдаст сертификат, и Caddy будет повторять попытки.

**«Перед доменом обнаружен CDN».**
Cloudflare-прокси и подобные несовместимы: терминируют TLS и видят capability, ломают учёт `X-Forwarded-For`, а операторы РФ режут такой трафик. Выключите проксирование (серое облако) или возьмите другой домен.

**Порты 80/443 заняты не Caddy.**
Установщик предложит отдать порты Caddy. Если там нужный вам веб-сервер, остановите его или используйте ручную интеграцию из README апстрима.

## Работа

**Панель показывает `readyz 503`.**
MTProxy недоступен: `systemctl status mtproxy`, `journalctl -u mtproxy -n 50`. В split-режиме на фронте `readyz` не показателен, смотрите `tgwebproxy backend`.

**Клиент не подключается, хотя всё зелёное.**
Ссылка от профиля, которого MTProxy не знает: `systemctl show -p ExecStart mtproxy.service` должен содержать по одному `-S` на каждый профиль из `tgwebproxy link`. Перезапустите `tgwebproxy mode <режим>`, он пересинхронизирует секреты. В split-режиме обновите секреты на backend.

**Сертификат не выдаётся.**
`journalctl -u caddy -n 50`. Проверьте A-запись, что порт 80 открыт снаружи, и что перед доменом нет CDN. Let's Encrypt проверяет домен с нескольких точек мира, гео-фильтры на 80/443 ломают выпуск.

**Сайт-прикрытие отдаёт 5xx.**
Файлы в `/srv/tproxy-site` должны читаться пользователем `tproxy`: `chmod -R o+r /srv/tproxy-site`, каталогам `o+x`, затем `systemctl restart tproxy-server`.

**Трафик на :443 в панели нулевой.**
Таблица nftables `tgmon` не применена: `systemctl restart tgmon-counters`. Счётчики сбрасываются при перезагрузке и рестарте nftables, история есть у vnstat.

**`tgwebproxy` показывает старую версию после self-update.**
Утилита подменяется через `mv`, запущенный экземпляр дорабатывает старым кодом. Запустите `tgwebproxy` заново.

## Полезные команды

```bash
journalctl -u caddy -u mtproxy -u tproxy-server -f --no-pager
curl -fsS http://127.0.0.1:8081/healthz; curl -fsS http://127.0.0.1:8081/readyz
curl -fsS http://127.0.0.1:8081/metrics | grep ^tproxy_
/usr/local/bin/tproxy-server -config /etc/tproxy-server/config.json -profiles-file /etc/tproxy-server/profiles.json -check
nft list table inet tgmon
```
