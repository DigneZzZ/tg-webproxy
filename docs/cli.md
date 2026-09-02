# Утилита tgwebproxy

Ставится в `/usr/local/bin/tgwebproxy` вместе с прокси. Нужен root.

## Меню

`tgwebproxy` без аргументов в терминале открывает панель, которая перерисовывается каждые 5 секунд, и меню в одну клавишу. Ctrl-C внутри любого действия возвращает в меню, ошибка действия тоже. Без терминала печатается панель и выход.

| Клавиша | Экран |
|---|---|
| `1` | Ссылки подключения для каждого профиля, инструкция по ручному вводу, в split-режиме команда для backend |
| `2` | Тип подключения: `https`, `https-lanes`, `websocket`, `websocket-lanes`, `all`. Профили проверяются `tproxy-server -check`, при ошибке откат |
| `3` | AD_TAG: показать, сменить, убрать. Подсказка, что вводить боту |
| `4` | Перезапуск служб |
| `5` | Журналы relay, MTProxy и Caddy в режиме follow |
| `6` | Обновить relay из апстрима с автооткатом |
| `7` | Версия и обновление утилиты |
| `8` | Удаление |
| `b` | Backend-хост: показать, направить, вернуть локально |
| `s` | Подробный статус: все метрики relay и помесячная таблица vnstat |
| `w` | Живой монитор: панель каждые 2 секунды, `q` для выхода |
| `0`, `q` | Выход |

## Панель

Заголовок: домен, версия утилиты, доступное обновление, вердикт `● работает` или `○ проблемы: …` со списком того, что не так.

| Раздел | Что показывает |
|---|---|
| Службы | caddy, relay, mtproxy (или backend-сокет в split), firewall, таймер обновления конфигурации MTProxy; healthz и readyz relay |
| Подключения | цепочка клиенты → caddy → relay → mtproxy по установленным TCP-сессиям; сессии, стримы и отказы relay; пользователи MTProxy |
| Трафик | счётчики relay, счётчики nftables на :443, vnstat по внешнему интерфейсу за сегодня и месяц |
| Настройки | тип подключения и число профилей, AD_TAG, коммит relay и дата установки, адрес и доступность backend |

Счётчики nftables сбрасываются при перезагрузке и при перезапуске nftables, vnstat хранит историю.

## Команды

```
tgwebproxy                      меню
tgwebproxy status [--full]      панель, --full добавляет метрики и vnstat
tgwebproxy watch                живая панель
tgwebproxy link                 ссылки подключения
tgwebproxy mode [режим]         тип подключения
tgwebproxy adtag [32hex|off]    спонсорский канал
tgwebproxy logs                 журналы
tgwebproxy restart              перезапуск служб
tgwebproxy update               обновить relay (или MTProxy на backend-хосте)
tgwebproxy version              версия и проверка обновлений
tgwebproxy self-update [--force] обновить утилиту
tgwebproxy uninstall [--purge]  удалить
tgwebproxy backend [show|set <ip[:port]>|local]
tgwebproxy secrets [set '<s1 s2>']       backend
tgwebproxy allow [<ip/cidr,...>]         backend
tgwebproxy help
```

Каждую команду можно вызвать и через установщик: `bash <(wget -qO- …/tg-webproxy.sh) status`.

## Обновление утилиты

Версия скрипта хранится в константе `TGWP_VERSION` и попадает в утилиту при генерации. Раз в сутки в фоне читаются первые 4 КБ скрипта из трёх источников: raw GitHub, jsDelivr и GitHub Pages, берётся максимальная версия, результат кэшируется в `/opt/tgwebproxy/version-check`. Панель показывает `→ x.y.z`, если есть новее.

`tgwebproxy self-update` скачивает скрипт из источника с максимальной версией, проверяет `bash -n` и наличие версии, затем перегенерирует утилиту командой `install-cli`. Файл пишется во временный и подменяется через `mv`, потому что bash читает скрипт по ходу выполнения и перезапись работающей утилиты на месте сломала бы её.

Отключить проверки: `TGWP_NO_UPDATE_CHECK=1`.

## Файлы

| Путь | Что там |
|---|---|
| `/opt/tgwebproxy/info.env` | домен, секрет, режим, профили, AD_TAG, роль; права 0600 |
| `/opt/tgwebproxy/site` | сгенерированная заглушка до копирования в `/srv/tproxy-site` |
| `/opt/tgwebproxy/shim/runuser` | обёртка для сборки MTProxy с нормальной umask |
| `/var/log/tgwebproxy-install.log` | полный вывод установки |
| `/etc/systemd/system/mtproxy.service.d/tgwp.conf` | единственный drop-in с секретами и AD_TAG для MTProxy |
| `/etc/tproxy-server/tgmon.nft`, `tgmon-counters.service` | счётчики трафика на :443 |
| `/etc/systemd/system/tgwp-backend.socket`, `.service` | split-режим, front |
| `/etc/tproxy-server/firewall.nft` | split-режим, backend: кому открыт MTProxy |
