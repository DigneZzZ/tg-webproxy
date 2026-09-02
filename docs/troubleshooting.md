# Диагностика

Установщик после сбоя сам ищет причину в `/var/log/tgwebproxy-install.log` и журнале systemd, печатает её, состояние служб и путь к журналу. Ниже то, что встречается чаще всего.

## Установка

**«Официальный установщик завершился с ошибкой», в журнале `--- FAIL: TestLoad…`.**
Упали unit-тесты апстрима. Один из них зависит от umask и падает на любой свежей установке, скрипт его пропускает через `GOFLAGS`. Если упал другой тест, смотрите строку `_test.go:` в журнале: это регрессия апстрима, закрепите рабочий коммит через `TGWP_REF`.

**`mtproxy.service: status=203/EXEC`.**
systemd не может выполнить `/opt/MTProxy/objs/bin/mtproto-proxy` от пользователя mtproxy: каталог `objs/` собран с правами 0700. Скрипт чинит это сам тремя способами. Вручную:

```bash
chmod -R a+rX /opt/MTProxy && systemctl restart mtproxy && curl -fsS http://127.0.0.1:8081/readyz
```

**`tproxy-server did not become ready`.**
Relay не увидел MTProxy на 127.0.0.1:2398. `journalctl -u mtproxy -n 50`: чаще всего MTProxy не стартует из-за секрета не 32 hex, пустого `-S` или недоступного `core.telegram.org` при загрузке `proxy-secret`.

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
