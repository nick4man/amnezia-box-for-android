# sing-box с AmneziaWG на OPNsense

Пакет ставит `sing-box`, собранный из форка [amnezia-box](https://github.com/hoaxisr/amnezia-box)
с тегом `with_awg`, то есть с поддержкой AmneziaWG (обфусцированный WireGuard),
плюс rc.d-службу, обёртку `sing-boxctl` и набор действий configd.

Проверено на **OPNsense 26.7.2_2-amd64 / FreeBSD 15.1-RELEASE-p2**.

---

## 1. Установка

```sh
# скачать артефакт sing-box-opnsense-amd64-pkg из GitHub Actions и положить в /root
sha256 -c sing-box-*.pkg.sha256 sing-box-*-freebsd-amd64.pkg   # опционально
pkg add -f /root/sing-box-*-freebsd-amd64.pkg
```

Что кладётся:

| Путь | Назначение |
|---|---|
| `/usr/local/bin/sing-box` | сам бинарь (статический, CGO выключен) |
| `/usr/local/sbin/sing-boxctl` | управление службой |
| `/usr/local/etc/rc.d/sing_box` | rc.d-скрипт |
| `/usr/local/etc/sing-box/config.json.sample` | минимальный шаблон (только AWG) |
| `/usr/local/etc/sing-box/config.example.json` | полный рабочий пример (см. раздел 4) |
| `/usr/local/opnsense/service/conf/actions.d/actions_singbox.conf` | действия для `configctl` |
| `/var/db/sing-box/` | рабочий каталог, кэш |

При первой установке `config.json` создаётся из `config.json.sample`, при удалении
пакета — удаляется, только если ты его не правил. Свой конфиг переживает
переустановку и обновление пакета.

Проверить, что AWG действительно внутри:

```sh
sing-box version | grep with_awg
```

## 2. Настройка

```sh
cp /usr/local/etc/sing-box/config.example.json /usr/local/etc/sing-box/config.json
vi /usr/local/etc/sing-box/config.json
sing-boxctl check          # обязательно перед стартом
```

Минимум, что надо заменить в примере: `private_key`, `address`, `peers[].address`,
`peers[].port`, `peers[].public_key`, `peers[].preshared_key`, `allowed_ips`,
адреса и ключи VLESS-серверов, IP LAN-интерфейса в `inbounds`.

Свою пару ключей генерирует сам sing-box:

```sh
sing-box generate wg-keypair
```

## 3. Запуск и автозапуск

```sh
sing-boxctl enable         # автозапуск при загрузке
sing-boxctl start
sing-boxctl status
```

Флаг автозапуска пишется в `/etc/rc.conf.d/sing_box`, а **не** в `/etc/rc.conf` —
OPNsense перегенерирует `rc.conf` при каждой загрузке и затёр бы его.

Все команды:

| Команда | Что делает |
|---|---|
| `sing-boxctl enable` / `disable` | автозапуск вкл/выкл |
| `sing-boxctl start` / `stop` / `restart` | запуск/остановка |
| `sing-boxctl reload` | проверить конфиг и послать SIGHUP — перечитывает конфиг **без разрыва соединений**; если конфиг битый, служба продолжает работать на старом |
| `sing-boxctl status` | автозапуск, pid, версия, пути, хвост лога |
| `sing-boxctl check` | валидация конфига |
| `sing-boxctl logs [-f] [N]` | лог (путь берётся из `log.output`) |
| `sing-boxctl config` | все действующие пути |

То же самое через configd (работает и из API OPNsense):

```sh
configctl singbox status
configctl singbox restart
configctl singbox reload
```

Переменные, которые можно переопределить в `/etc/rc.conf.d/sing_box`:

```sh
sysrc -f /etc/rc.conf.d/sing_box sing_box_config=/usr/local/etc/sing-box/other.json
sysrc -f /etc/rc.conf.d/sing_box sing_box_workdir=/var/db/sing-box
sysrc -f /etc/rc.conf.d/sing_box sing_box_logfile=/var/log/sing-box/sing-box.log
sysrc -f /etc/rc.conf.d/sing_box sing_box_flags=""
```

## 4. Разбор примера конфига

Полный файл — [`config.example.json`](etc/config.example.json), он проходит
`sing-box check` как есть (ключи в нём — сгенерированные пустышки).

### AmneziaWG — это `endpoint`, не outbound

В sing-box 1.14 AWG живёт в секции `endpoints` (в 1.13 был outbound с блоком `awg` —
старые примеры из интернета не подойдут):

```json
"endpoints": [{
  "type": "awg",
  "tag": "wg-ep",
  "useIntegratedTun": false,
  "private_key": "<твой приватный ключ>",
  "address": "10.10.0.150/32",
  "mtu": 1420,
  "jc": 5, "jmin": 50, "jmax": 1000,
  "s1": 28, "s2": 137,
  "h1": "1631980850", "h2": "1967581631", "h3": "1485046168", "h4": "1803539852",
  "peers": [{
    "address": "203.0.113.10",
    "port": 55566,
    "public_key": "<публичный ключ сервера>",
    "preshared_key": "<PSK>",
    "allowed_ips": "10.10.0.0/24",
    "persistent_keepalive_interval": 25
  }]
}]
```

Соответствие параметрам из конфига AmneziaWG: `Jc → jc`, `Jmin → jmin`,
`Jmax → jmax`, `S1 → s1`, `S2 → s2`, `H1..H4 → h1..h4`. **`h1`–`h4` здесь строки**,
а не числа — в кавычках. Есть ещё `i1`–`i5` и `s3`, `s4`, если сервер их использует.

`allowed_ips` — это не «маршрут», а фильтр: пакеты вне этих подсетей туннель просто
не пропустит. Хочешь гнать через AWG весь трафик — ставь `"0.0.0.0/0"`. В примере
туннель используется только для доступа в подсеть `10.10.0.0/24`, и правило
маршрутизации отправляет туда ровно её.

### DNS

Формат серверов новый (`type` + `server`), старый `address: "tcp://1.1.1.1"` больше
не принимается:

```json
{ "type": "udp", "tag": "local-dns", "server": "127.0.0.1", "server_port": 53 },
{ "type": "tcp", "tag": "proxy-dns", "server": "1.1.1.1",
  "detour": "proxy", "domain_resolver": "local-dns" }
```

`local-dns` смотрит в Unbound самого OPNsense, `proxy-dns` ходит наружу через прокси.
`domain_resolver` нужен, чтобы имя самого DNS-сервера было кем-то разрезолвлено.

### Входы

`socks`/`http` на `127.0.0.1` и на IP LAN-интерфейса — sing-box здесь работает
прокси-сервером, а не перехватывает трафик. Это самый спокойный режим для файрвола:
ничего не трогает в маршрутизации, клиенты сами выбирают, идти через прокси или нет.
Не забудь правило firewall, если открываешь порт в LAN.

### Выходы и маршруты

`selector` `proxy` — то, что переключается руками/через Clash API; `urltest` `auto`
внутри него сам выбирает живой VLESS. Порядок правил в `route.rules` важен, сверху вниз:

```
sniff → hijack-dns → 10.10.0.0/24 в AWG → приватные сети напрямую →
.internal/.lan напрямую → реклама в reject → RU-домены и RU-IP напрямую →
всё остальное (final) в proxy
```

`direct` outbound обязателен, если на него ссылаются правила.

### experimental

`cache_file` хранит кэш DNS и **выбранный в селекторе outbound** — он переживает
перезапуск. `clash_api` на `127.0.0.1:9090` позволяет переключать селектор:

```sh
curl -s http://127.0.0.1:9090/proxies/proxy                       # текущий выбор
curl -s -X PUT -H 'Content-Type: application/json' \
  -d '{"name":"auto"}' http://127.0.0.1:9090/proxies/proxy        # переключить
```

## 5. Проверка

```sh
curl -s --socks5-hostname 127.0.0.1:1080 -o /dev/null -w '%{http_code}\n' \
  https://www.gstatic.com/generate_204        # ожидаем 204
curl -s --socks5-hostname 127.0.0.1:1080 https://api.ipify.org; echo   # IP выхода
sing-boxctl logs 50
```

## 6. Типичные грабли

**`panic: invalid memory address` в `direct.(*Outbound).fetchMyAddresses`.**
Апстримный баг 1.14 на FreeBSD: монитора сетевых интерфейсов на этой платформе нет,
а `direct` outbound дёргает его без проверки, поэтому падает любой конфиг с `direct`.
В пакетах из этого репозитория патч уже наложен при сборке — проверь, что стоит
именно этот бинарь (`sing-box version` должен показать `1.14.0-beta.14-awgm.10`
или новее), а не собранный вручную из апстрима.

**`pkg add` завершается с `Segmentation fault`.** Пакет собран более новым pkg,
чем 2.3.1 в OPNsense: формат манифеста разъехался. Пакеты этого репозитория
пересобирают манифест в совместимый вид, так что бери артефакт из Actions,
а не собранный чужим `pkg create`.

**Через прокси ничего не ходит, в логе `dial TCP connection: context deadline exceeded`
на `endpoint/awg`.** Селектор переключён на `wg-ep`, а `allowed_ips` туннеля не
включает интернет. Верни селектор на `auto` (см. Clash API выше) — выбор хранится
в `cache.db` и переживает перезапуск, сам он не сбросится.

**Порт занят / странное поведение после ручного запуска.** Проверь, что нет второго
экземпляра: `sing-boxctl status` покажет `started outside the service`, если процесс
поднят мимо rc.d. Лишний прибей и работай через `sing-boxctl`.

**`Cannot 'start' sing_box. Set sing_box_enable to YES`.** Это `service`, а не
`sing-boxctl` — обёртка сама подставляет `onestart`, когда автозапуск выключен.

## 7. Обновление и удаление

```sh
pkg add -f /root/sing-box-<новая версия>-freebsd-amd64.pkg   # конфиг не тронется
sing-boxctl restart
```

```sh
sing-boxctl stop && sing-boxctl disable
pkg delete -y sing-box
rm -f /etc/rc.conf.d/sing_box
```

## 8. Сборка из исходников

Workflow [`build_opnsense.yaml`](../.github/workflows/build_opnsense.yaml): собирает
FreeBSD-бинарь, накладывает патч `direct`, пакует `.pkg` в VM с FreeBSD 15.1 и там же
прогоняет установку, запуск демона и полный цикл `sing-boxctl`.
