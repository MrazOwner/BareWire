# Barewire

**VPN-клиент для Linux, который показывает, что происходит на самом деле.**

Barewire — текстовый интерфейс к привилегированному хосту, который держит
[sing-box](https://github.com/SagerNet/sing-box) внутри себя как библиотеку. Ни
демона, ни обёртки над чужим CLI, ни скрытого состояния: интерфейс говорит с
хостом по unix-сокету, хост собирает конфигурацию ядра из твоих леверов, и
каждый экран показывает факты, а не успокоение.

> **Статус: production-ready на моём опыте.** Это мой основной клиент, проверенный
> ежедневной работой на Arch Linux. Проект не заброшен и находится в стабильной
> разработке: планы ниже — это очередь задач, а не пожелания. Честные ограничения
> перечислены отдельно и не спрятаны — если что-то не работает, в логе есть строка,
> которая объясняет почему.

**Русский** · [English](#english)

```
BAREWIRE                                               connected · sing-box 1.15.0-alpha.13
1 Status   2 Settings   3 Logs   4 Subscriptions   5 Profiles
────────────────────────────────────────────────────────────────────────────────────────────────────
╭─ STATUS ────────────────────────────────────────────────────────────────────────────────────────╮
│State               connected                                                           since 0:01│
│Server              de1.example.net:443                                                      vless│
│Outbound            proxy[vless]                                                        rule final│
│TUN stack           sing-tun                                                       mtu 1420 (auto)│
│Multiplexing        off                                                                           │
│TCP Brutal          off                                                                           │
│Congestion          bbr · fq                                                                kernel│
│DNS                 fakeip                                                        1.1.1.1, 1.0.0.1│
│Kill switch         on · keep / flush                           ruleset v3 · crash keep · net keep│
│Reconnects          0                                                            no disconnect yet│
│Throughput          ↑ 281.3 KiB/s  ↓ 1.4 MiB/s                           total 476.0 KiB / 3.0 MiB│
│Connections         21                                                                            │
│Resources           41.0 MiB                                                              cpu 0.7%│
╰──────────────────────────────────────────────────────────────────────────────────────────────────╯
```

---

## Русский

### Содержание

- [Зачем ещё один клиент](#зачем-ещё-один-клиент)
- [Возможности](#возможности)
- [Профили и пресеты](#профили-и-пресеты)
- [Что проверено](#что-проверено)
- [Требования](#требования)
- [Сборка и установка](#сборка-и-установка)
- [Клавиши](#клавиши)
- [Справка по настройкам](#справка-по-настройкам)
- [Как это работает](#как-это-работает)
- [Файлы](#файлы)
- [Диагностика](#диагностика)
- [Если что-то не работает](#если-что-то-не-работает)
- [Ограничения и недочёты](#ограничения-и-недочёты)
- [Разработка](#разработка)
- [Планы](#планы)
- [Благодарности](#благодарности)

### Зачем ещё один клиент

Большинство клиентов для Linux — это либо тонкая обёртка над файлом
конфигурации, либо чёрный ящик с одной большой кнопкой. Barewire построен вокруг
трёх правил:

1. **Без вранья.** Левер, который ничего не может, говорит об этом вместо
   молчания. Ядро, переставшее отвечать, показывается как переставшее отвечать, а
   не как «connected». Конфигурация, которую отвергает настоящий декодер sing-box,
   до ядра не доходит.
2. **Ничего в фоне.** Хост поднимается, когда ты подключаешься, и завершается
   вместе с интерфейсом. После закрытия окна не остаётся ни процесса, ни правил.
3. **Никаких молчаливых изменений системы.** Firewall, kernel-леверы и DNS-guard
   пишутся только по твоей команде, каждый хранит бэкап, а снятие левера
   возвращает машину в исходное состояние.

### Возможности

**Транспорт**

- Выбор TUN-стека: собственный стек sing-tun (sing-box 1.15), `mixed` (TCP ядра +
  gvisor UDP), `gvisor`, `system`; multi-queue предлагается только там, где он
  существует.
- MTU: вручную или измеренный — по MSS, который согласует сервер, и по MTU
  маршрута для транспортов, говорящих только по UDP. Значение, которое путь не
  несёт, понижается, и этот факт попадает в лог и в статус.
- Мультиплексирование (`smux`, `yamux`, `h2mux`), padding и TCP Brutal с
  измеренной скоростью — каждое пишется только туда, где протокол и ядро это
  действительно умеют.

**DNS**

- Режимы `fakeip`, `direct`, `disabled`, готовые наборы резолверов (Cloudflare,
  Google, Quad9, AdGuard, Mullvad, OpenDNS) и переключатель `Include IPv6`,
  который управляет резолверами, IPv6-адресом TUN и ответами `AAAA`.
- Запросы резолверов **всегда идут внутри туннеля**: сеть, режущая порт 53, не
  ломает разрешение имён, а резолверы не видят твой настоящий адрес. Единственный
  запрос наружу — имя самого сервера, когда эндпоинт задан доменом: его нужно
  разрешить до того, как туннель появится.
- `Auto redirect`: nftables-правила, которые заставляют TUN забрать трафик,
  проходящий мимо него. Это единственный способ поймать резолвер на loopback
  (systemd-resolved слушает `127.0.0.53`, и такие пакеты в TUN не попадают) — то,
  что тест утечек показывает как резолверы провайдера.
- **Проверено:** после этих двух исправлений `dnsleaktest` и `ipleak.net` не
  показывают ни одного адреса провайдера или чужого резолвера — только DNS
  туннеля.
- **IPv6 DNS guard:** при выключенном IPv6 туннель его не несёт, и запрос к
  v6-резолверу ушёл бы напрямую. Guard закрывает ровно это — порт 53 и 853 по
  IPv6 — и не трогает локальный IPv6.

**Маршрутизация**

- Список «напрямую» — IP-префиксы, сайты (`example.com`, `*keyword*`), процессы
  (`steam` покрывает и `steamwebhelper`) — идёт мимо туннеля. Списка запрета нет: в
  туннеле, который и так несёт всё, он повторял бы дефолт.

**Firewall**

- Kill switch как таблица nftables в `output` с разрешёнными адресами сервера в
  обоих семействах, версией набора правил и явными политиками на краш и на смену
  сети.
- Набор, который не разрешает ни одного адреса сервера, не устанавливается: он
  заблокировал бы сам туннель.
- Упавший хост не оставляет машину без сети: следующий старт снимает его правила и
  TUN, а второй хост не отбирает живой сокет.
- В набор правил входят DHCP и NDP: без них аренда истекает и сеть умирает через
  час, при «живом» туннеле.

**Ядро**

- Congestion control из всего каталога Linux — `bbr`, `cubic`, `dctcp`, `cdg`,
  `westwood`, `vegas`, `illinois`, `yeah`, `htcp`, `scalable`, `hybla`, `veno`,
  `nv`, `lp`, `brutal`, `bic`, `highspeed`, `reno`; для значений-модулей сначала
  выполняется `modprobe`, потом чтение обратно — ошибка видна, а не проглочена.
- Queue discipline (`fq`, `fq_codel`, `cake`), `tcp_mtu_probing` и
  `tcp_slow_start_after_idle` — одним `sysctl.d` drop-in с бэкапом.
- **Прогон алгоритмов:** замеряет каждый алгоритм через живой туннель (полоса и
  p95 джиттера), считает оценку и запоминает победителя для конкретного сервера.
- QUIC congestion (`cubic`, `new_reno`, `bbr`) для TUIC, где TCP ядра нет вообще.

**Всё остальное**

- Подписки: Clash YAML, V2Ray base64 и отдельные ссылки `vless`, `vmess`,
  `trojan`, `ss`, `hysteria2`, `tuic`.
- Профили с привязкой к серверу или сети, пять встроенных пресетов и сохранение
  каждой правки в активный профиль.
- Логи ядра без цветов и без времени (время ставит интерфейс), с фильтром и
  отдельной панелью; всё дополнительно пишется в `host.log`, который переживает
  падение.
- **Watchdog:** ядро, которое перестало отвечать своему же API, перезапускается
  (три попытки, вторая — со снятым alpha-стеком и multi-queue), после чего линк
  честно завершается с причиной. Хост, который умер, интерфейс поднимает сам и
  показывает последние строки его лога.
- **Проверка мультиплексирования:** сразу после подключения через туннель уходит
  один запрос. Ответа нет — левер выключается на сессию с причиной в логе.
- Замер скорости без `curl`: `barewire-ctl speed`.

### Профили и пресеты

На чистой установке список профилей заполняется пресетами; их можно менять как
угодно, и правка леверов на экране Settings автоматически пишется в активный
профиль — активация того же профиля больше не откатывает работу.

| Профиль | Что внутри | Для чего |
|---|---|---|
| `balanced` | sing-tun, MTU auto, fakeip | дефолт: быстрый и без лишних движений |
| `compatible` | `mixed` стек, MTU 1380, mtu probing on | ссылка поднимается и встаёт, большие пакеты не проходят |
| `quic` | `mixed`, MTU 1370, QUIC congestion `bbr` | hysteria2 и TUIC |
| `fast` | `mixed`, MTU 1420, DNS `direct` (1.1.1.1) | без fakeip, настоящие ответы DNS |
| `locked` | kill switch on, auto redirect on | утечек нет даже при падении ядра |

На экране Profiles: `enter` — активировать, `s` — сохранить текущие леверы в
выбранный профиль, `n` — новый из текущих, `r` — переименовать, `d` — удалить,
`b` — привязать сеть, `S` — привязать сервер.

### Что проверено

- `make check` — gofmt, go vet и весь набор тестов: **279 тестов**, включая
  настоящий декодер sing-box 1.15 на сгенерированных конфигурациях и `nft -c` на
  тексте правил.
- `go test -race` на пакетах с конкурентностью (host, socket, core, tui) — чисто.
- Утечки DNS: `dnsleaktest` и `ipleak.net` не показывают адресов провайдера.
- Полоса: `barewire-ctl speed` даёт воспроизводимые числа, и разница между
  «туннель выключен» и «включён» — это цена туннеля, а не догадка.
- Отказы: ядро, которое не может создать TUN, заканчивается состоянием
  «disconnected» с причиной, а не «connected».

### Требования

- Linux с TUN и nftables (Arch, Debian/Ubuntu, Fedora — без разницы, важны ядро и
  `nft`). X11/Wayland не нужны: это TUI.
- `root` для хоста (`CAP_NET_ADMIN` + `CAP_NET_RAW`): TUN, nftables, sysctl.
- `iproute2` для наблюдения за маршрутом; `nftables` для kill switch, DNS-guard и
  `auto redirect`.
- Go 1.25+ для сборки. sing-box не нужно ставить отдельно — он вкомпилирован.

### Сборка и установка

```sh
make build                                   # три бинарника в ./bin, от твоего пользователя
make install PREFIX=$HOME/.local             # в ~/.local/bin, без sudo
sudo make install-only                       # в /usr/local/bin, когда bin/ уже собран
```

Собирать под root Makefile отказывается: `bin/` стал бы root-owned.

Запуск: `barewire` (или `~/.local/bin/barewire`, если каталога нет в `PATH`).
Первый `Connect` поднимает `barewire-host` через **pkexec**; если polkit в системе
нет, используется `sudo -n`, то есть нужен заранее обновлённый sudo-тикет
(`sudo -v` в соседнем терминале). Закрытие интерфейса останавливает хост.

```sh
make check      # что должно проходить перед коммитом
make lint       # golangci-lint, если установлен
make cover      # покрытие
make preview TAB=2 W=120 H=40   # отрисовать экран в stdout, без root и сети
make mock       # интерфейс против смоделированного хоста
```

### Клавиши

| Клавиша | Что делает |
|---|---|
| `1`–`5` | Status, Settings, Logs, Subscriptions, Profiles |
| `tab` / `shift+tab` | следующая / предыдущая вкладка |
| `enter` | подключиться на Status; активировать левер на остальных |
| `h` `l` / `←` `→` | шаг по выбору, числу или переключателю |
| `enter` ещё раз | ввести точное значение; `esc` отменяет |
| `q` | выход (хост останавливается вместе с интерфейсом) |

### Справка по настройкам

| Группа | Левер | Что пишет |
|---|---|---|
| Network | TUN stack | `stack` в tun-inbound; собственный стек sing-tun — это *отсутствие* поля |
| | MTU | `inbounds[].mtu`; `auto` меряет путь и вычитает оверхед транспорта, левер идёт по размерам 1280…1452 и принимает точное значение, а то, что путь не несёт, понижается с записью в лог |
| | Multi-queue | `inbounds[].multi_queue`, только на собственном стеке |
| | Auto redirect | `inbounds[].auto_redirect`; nftables-правила, забирающие в TUN трафик, который иначе прошёл бы мимо, включая резолвер на loopback |
| DNS | Mode | `fakeip`, секция `direct` или отсутствие DNS вообще |
| | Preset / Servers | резолверы: из набора или введённые руками |
| | Include IPv6 | IPv6-резолверы, IPv6-адрес TUN, ответы `AAAA` |
| | On network change | перезапустить ядро, оставить или отключиться |
| Direct | IPs / Sites / Processes | `ip_cidr`, `domain_suffix`+`domain_keyword`, `process_path_regex` → `direct` |
| Multiplexing | Enabled / Protocol / Padding | `outbounds[].multiplex`, только для `vless`, `vmess`, `trojan`, `shadowsocks` и только без `flow` |
| TCP Brutal | Enabled / Up / Down / Auto-detect | `outbounds[].multiplex.brutal`, только если ядро предлагает `brutal` |
| Kernel | Algorithm | `net.ipv4.tcp_congestion_control`; предлагается весь каталог, модуль загружается при применении и проверяется чтением |
| | Queue discipline | `net.core.default_qdisc` |
| | MTU probing | `net.ipv4.tcp_mtu_probing = 1` |
| | Slow start after idle | `net.ipv4.tcp_slow_start_after_idle = 0` |
| | QUIC congestion | `outbounds[].congestion_control` для TUIC |
| | Test algorithms | `kernel-test.json`, результат на каждый сервер |
| Kill switch | Enabled | nftables `inet barewire`, policy drop на output |
| | On core crash | `keep` оставляет машину заблокированной, `flush` открывает |
| | On network change | оставить или снять набор правил; перезапустить, оставить или остановить ядро |
| | Flush ruleset now | снять таблицы по требованию |
| Theme | Accent / Accent alt | два цвета интерфейса |

### Как это работает

```
barewire (TUI)  ──unix socket, JSON построчно──  barewire-host (root)
   settings.json                                  ├─ sing-box как библиотека
   profiles.json                                  ├─ nftables: kill switch + DNS guard
   subscriptions.json                             ├─ sysctl.d: kernel-леверы
                                                  └─ TUN, пробы MTU и скорости
```

- **Конфигурация собирается, а не шаблонится.** `internal/core/config` — чистая
  функция из набора леверов в JSON-документ. Тесты скармливают результат
  настоящему декодеру sing-box, поэтому документ, который ядро отвергло бы, до
  запущенного ядра не доходит.
- **Один хост, один сокет.** Сокет с правами `0600`, uid собеседника проверяется на
  каждом соединении, второй хост не отбирает сокет, который отвечает.
- **За туннелем смотрят, а не верят ему.** Ядро, чей API замолчал, перезапускается
  через полминуты тишины. Вторая попытка снимает наименее проверенное — alpha-стек
  `sing-tun` и multi-queue — на эту сессию и говорит об этом.
- **Перезапуск — крайняя мера.** Ядро перезапускает только левер, который до него
  доходит; цвет, политика kill switch и правило смены сети применяются на месте.
- **Факты, а не пожелания.** Статус показывает то, что в силе: алгоритм, о котором
  сообщает ядро, MTU, который получило ядро sing-box, состояние mux, которое оно
  получило, версию набора правил — а не то, что написано в файле настроек.

### Файлы

| Путь | Что внутри |
|---|---|
| `$XDG_CONFIG_HOME/barewire/settings.json` | набор леверов, режим 0600 |
| `$XDG_CONFIG_HOME/barewire/profiles.json` | именованные наборы и их привязки |
| `$XDG_CONFIG_HOME/barewire/subscriptions.json` | ссылки подписок и разобранные узлы |
| `$XDG_CONFIG_HOME/barewire/cache.db` | хранилище fakeip, появляется при первом подключении |
| `$XDG_CONFIG_HOME/barewire/kernel-backup.json` | значения ядра до применения левера |
| `$XDG_CONFIG_HOME/barewire/kernel-test.json` | последний прогон алгоритмов, по серверам |
| `$XDG_RUNTIME_DIR/barewire/host.sock` | сокет хоста, 0600, uid проверяется |
| `$XDG_RUNTIME_DIR/barewire/host.log` | все строки лога хоста, ротация на 4 МиБ; файл, который переживает падение |
| `/etc/sysctl.d/99-barewire-kernel.conf` | пишется, только если kernel-левер о чём-то просит |

Ни один файл не создаётся заранее.

### Диагностика

```sh
barewire-ctl status                      # те же факты в JSON
barewire-ctl connections                 # таблица соединений
barewire-ctl subfetch https://… -ua …    # что на самом деле ответил провайдер
barewire-ctl nodecheck de1               # разобрать и проверить один узел
barewire-ctl addlink 'vless://…'         # импортировать одну ссылку
barewire-ctl speed                       # скачать и залить через тот маршрут, что есть у процесса
sudo barewire-host -cleanup              # правила, TUN и fakeip-кэш
barewire-host -version                   # чем собран бинарник: старый виден сразу
```

Лог хоста — первое место, куда смотреть:

```sh
grep -E 'core:|kernel:|mux|kill switch|dns|mtu' \
  ${XDG_RUNTIME_DIR:-/run/user/$(id -u)}/barewire/host.log | tail -30
```

`speed` не требует ни хоста, ни `curl`: качает 10 МиБ с эндпоинта Cloudflare,
пропускает первую четверть как прогрев (цифра, включающая slow start, — это не
канал) и печатает оба направления в Мбит/с и МиБ/с.

### Если что-то не работает

| Симптом | Что это |
|---|---|
| Линк поднят, ничего не грузится | Включён mux, а сервер не говорит на sing-mux. Клиент проверяет это сам: сразу после подключения через туннель уходит запрос, и если ответа нет — левер выключается на сессию с причиной в логе. Сервер на Xray sing-mux не умеет в принципе. |
| Видео встаёт каждые несколько минут | Сначала замер: `barewire-ctl speed` с туннелем и без. Если канал в порядке — дело не в туннеле. Если нет — смотри в логе `mtu:` и `core:`, поставь `MTU probing = on` и попробуй стек `mixed`. |
| После падения нет сети | Набор правил kill switch пережил туннель. Следующий запуск хоста его снимает; `barewire-host -cleanup` делает то же руками. |
| Тест утечек показывает провайдера | `cat /etc/resolv.conf`: `nameserver 127.0.0.53` — это systemd-resolved, его пакеты в TUN не попадают. Включи `Auto redirect`; в логе при подключении будет строка `dns:`. |
| Левер ничего не меняет | Прочитай примечание под левером и строку в логе после него. Каждый левер, который не может сработать, говорит почему. |

### Ограничения и недочёты

- **Только Linux.** TUN, nftables и `sysctl` — это весь замысел.
- **Собственный стек `sing-tun` в 1.15 — alpha.** Он выбран по умолчанию, потому
  что он быстрее и это будущее sing-box, но именно поэтому watchdog умеет
  откатываться на `mixed`.
- **Mux и TCP Brutal требуют поддержки на сервере.** Клиентской хитрости не
  существует; mux хотя бы проверяется запросом, а Brutal требует модуль
  `tcp_brutal` в ядре.
- **BBR доступен не везде.** Если `tcp_bbr` нет ни в ядре, ни модулем, остаются
  `cubic` и другие; интерфейс честно пишет, что загрузит при применении.
- **DoH на 443 не перехватывается** — иначе пришлось бы ломать TLS.
- **Резолвер на loopback** ловится только через `auto redirect`, которому нужен
  nftables.
- **Нет цикла переподключений на видимых ошибках:** неудачный старт — это
  disconnected с причиной, а зависший туннель заменяет watchdog, а не бесконечный
  retry.
- **Нет файла лицензии.** Выбрать осознанно до публикации.
- **Нет GUI и мобильных клиентов** — и не планируется: это TUI по замыслу.

### Разработка

```sh
make check                      # gofmt, go vet, весь набор тестов
make lint                       # golangci-lint, когда установлен
go test -race ./internal/...    # конкурентные пути: host, socket, core, tui
```

Набор тестов покрывает настоящий декодер sing-box на сгенерированных конфигурациях,
текст правил nftables (проверяется `nft -c`, когда утилита есть), протокол сокета на
живом unix-сокете, бэкап и восстановление sysctl, парсеры подписок, кольцевой буфер
логов, геометрию текста и все экраны, оценку прогона алгоритмов и пути отказа:
ядро, которое не может открыть TUN, обязано оказаться disconnected с причиной.

```
cmd/barewire/          интерфейс
cmd/barewire-host/     привилегированный хост: sing-box, TUN, nftables, sysctl
cmd/barewire-ctl/      леверы из шелла через тот же сокет, JSON на выход
cmd/tuipreview/        рендер одного экрана в stdout
internal/config/       настройки, профили, подписки, валидация
internal/core/         встроенное ядро: decode, start, stop, статистика
internal/core/config/  сборщик конфигурации (чистые функции, под тестами)
internal/host/         привилегированная половина: жизненный цикл, леверы, ядро
internal/hostsock/     протокол сокета
internal/system/       nftables, sysctl, пробы, наблюдение за маршрутом
internal/tui/          экраны, ввод, раскладка
internal/subscription/ разбор Clash YAML и ссылок
```

Правила, которые стоит сохранять: сборщик конфигурации остаётся чистым и без option-структур,
интерфейс ничего не решает сам, каждая ошибка сообщается, а не «повторяется тихо», и
левер, который не может сработать, говорит об этом.

### Планы

Проект жив и развивается; ближайшая очередь:

1. Разобраться с зависаниями alpha-стека `sing-tun` и, если причина в библиотеке,
   отправить отчёт наверх.
2. Цель `make race` в Makefile и прогон `-race` для всего репозитория.
3. Лицензия, `git`-история и первые релизы.
4. Больше пресетов под конкретные сети (мобильный оператор, Wi-Fi с captive portal).
5. Детект DoH внутри браузера и подсказка, как его выключить.

### Благодарности

Barewire — это фронтенд. Туннель, TUN-стек и протоколы — это
[sing-box](https://github.com/SagerNet/sing-box) и
[sing-tun](https://github.com/SagerNet/sing-tun). Интерфейс использует
[Bubble Tea](https://github.com/charmbracelet/bubbletea).

---

## English

[Русский](#русский) · **English**

### Contents

- [Why another client](#why-another-client)
- [Features](#features)
- [Profiles and presets](#profiles-and-presets)
- [What has been verified](#what-has-been-verified)
- [Requirements](#requirements)
- [Build and install](#build-and-install)
- [Keys](#keys)
- [Settings reference](#settings-reference)
- [How it works](#how-it-works)
- [Files](#files)
- [Diagnostics](#diagnostics)
- [When something does not work](#when-something-does-not-work)
- [Limitations and rough edges](#limitations-and-rough-edges)
- [Development](#development)
- [Plans](#plans)
- [Acknowledgements](#acknowledgements-1)

### Why another client

Most Linux clients are either a thin wrapper around a configuration file or a black
box with one big button. Barewire is built around three rules:

1. **No lies.** A lever that cannot do anything says so instead of doing nothing. A
   core that stopped answering is shown as such, not as "connected". A
   configuration the real sing-box decoder rejects never reaches the core.
2. **No background anything.** The host starts when you connect and exits when the
   interface exits. Nothing survives the window — no process, no ruleset.
3. **No silent network changes.** The firewall, the kernel levers and the DNS guard
   are written only when you ask for them, each keeps a backup, and clearing a
   lever restores the machine exactly as it was.

### Features

**Transport**

- TUN stack selector: sing-tun's own stack (sing-box 1.15), `mixed` (kernel TCP +
  gvisor UDP), `gvisor`, `system`; multi-queue is offered only where it exists.
- MTU: manual, or measured — from the MSS the server negotiates over TCP, and from
  the kernel's route MTU for the transports that speak UDP only. A value the path
  cannot carry is lowered, and the fact is written to the log and shown in status.
- Multiplexing (`smux`, `yamux`, `h2mux`), padding, and TCP Brutal with a measured
  rate — each written only where the protocol and the kernel can actually carry it.

**DNS**

- `fakeip`, `direct` and `disabled` modes, the well-known resolver presets
  (Cloudflare, Google, Quad9, AdGuard, Mullvad, OpenDNS) and an `Include IPv6`
  toggle that governs the resolvers, the IPv6 TUN address and `AAAA` answers.
- Resolver queries **always travel inside the tunnel**: a network that blocks port
  53 cannot break name resolution, and no resolver sees your real address. The only
  lookup that goes out directly is the server's own name when the endpoint is a
  domain — it has to be resolved before the tunnel exists.
- `Auto redirect`: nftables rules that make the TUN claim traffic which would
  otherwise bypass it. This is the only way to catch a resolver daemon on the
  loopback (systemd-resolved answers on `127.0.0.53`, and those packets never enter
  the TUN) — what a leak test shows as the ISP's resolvers.
- **Verified:** after these two fixes, `dnsleaktest` and `ipleak.net` show no ISP
  address and no foreign resolver at all — only the tunnel's DNS.
- **IPv6 DNS guard:** with IPv6 off the tunnel carries none, so a query to a v6
  resolver would leave the machine directly. The guard closes exactly that — port
  53 and 853 over IPv6 — and leaves local IPv6 alone.

**Routing**

- A direct list — IP prefixes, sites (`example.com`, `*keyword*`), processes
  (`steam` also covers `steamwebhelper`) — that goes around the tunnel. There is no
  deny list: inside a tunnel that already carries everything it would only repeat
  the default.

**Firewall**

- Kill switch as an nftables table in `output`, with the server's addresses allowed
  in both families, a ruleset version and explicit crash and network-change
  policies.
- A ruleset that would allow no server address at all is refused instead of
  installed: it would block the tunnel itself.
- A crashed host cannot leave the machine offline: the next start clears its ruleset
  and TUN link, and a second host refuses to take over a live socket.
- The ruleset includes DHCP and NDP: without them the lease expires and the network
  dies an hour into a session with the tunnel looking healthy.

**Kernel**

- Congestion control from the whole Linux catalogue — `bbr`, `cubic`, `dctcp`,
  `cdg`, `westwood`, `vegas`, `illinois`, `yeah`, `htcp`, `scalable`, `hybla`,
  `veno`, `nv`, `lp`, `brutal`, `bic`, `highspeed`, `reno`; for values that ship as
  modules `modprobe` runs first and the result is read back, so a failure is
  visible instead of swallowed.
- Queue discipline (`fq`, `fq_codel`, `cake`), `tcp_mtu_probing` and
  `tcp_slow_start_after_idle` through one `sysctl.d` drop-in with a backup.
- **Trial run:** measures every algorithm through the live tunnel (throughput and
  p95 jitter), scores them and remembers the winner for that server.
- QUIC congestion (`cubic`, `new_reno`, `bbr`) for TUIC, where there is no kernel
  TCP in the path at all.

**Everything else**

- Subscriptions: Clash YAML, V2Ray base64 and individual `vless`, `vmess`, `trojan`,
  `ss`, `hysteria2`, `tuic` links.
- Profiles bound to a server or a network, five built-in presets, and every edit
  saved into the active profile.
- Core logs without colours and without timestamps (the interface stamps them),
  filterable and in their own panel; everything is also appended to `host.log`,
  which survives a crash.
- **Watchdog:** a core that stopped answering its own API is restarted (three
  attempts, the second with the alpha stack and multi-queue dropped), and after that
  the link ends with the reason. A host that dies is relaunched by the interface,
  which also prints the last lines of its log.
- **Multiplexing is verified:** one request goes through the tunnel right after
  connecting. No answer means the lever is turned off for the session, with the
  reason in the log.
- A speed measurement without `curl`: `barewire-ctl speed`.

### Profiles and presets

On a fresh install the profile list is filled with the presets; they can be edited
like any other, and a lever change on the Settings screen is written into the active
profile automatically — activating the same profile again no longer undoes the work.

| Profile | What is inside | What it is for |
|---|---|---|
| `balanced` | sing-tun, MTU auto, fakeip | the default: fast, no extra moving parts |
| `compatible` | `mixed` stack, MTU 1380, mtu probing on | the link comes up and then stalls, large packets do not pass |
| `quic` | `mixed`, MTU 1370, QUIC congestion `bbr` | hysteria2 and TUIC |
| `fast` | `mixed`, MTU 1420, DNS `direct` (1.1.1.1) | no fakeip, real DNS answers |
| `locked` | kill switch on, auto redirect on | no leaks even when the core dies |

On the Profiles screen: `enter` activates, `s` saves the current levers into the
selected profile, `n` creates one from the current levers, `r` renames, `d` removes,
`b` binds the network and `S` the server.

### What has been verified

- `make check` — gofmt, go vet and the whole suite: **279 tests**, including the real
  sing-box 1.15 decoder on generated configurations and `nft -c` on the ruleset text.
- `go test -race` on the packages with concurrency (host, socket, core, tui) — clean.
- DNS leaks: `dnsleaktest` and `ipleak.net` show no ISP address.
- Throughput: `barewire-ctl speed` gives reproducible numbers, and the difference
  between "tunnel down" and "tunnel up" is the tunnel's cost, not a guess.
- Failures: a core that cannot create the TUN device ends up `disconnected` with a
  reason, not `connected`.

### Requirements

- Linux with TUN and nftables (Arch, Debian/Ubuntu, Fedora — it does not matter;
  the kernel and `nft` do). No X11/Wayland: this is a TUI.
- `root` for the host (`CAP_NET_ADMIN` + `CAP_NET_RAW`): TUN, nftables, sysctl.
- `iproute2` for route watching; `nftables` for the kill switch, the DNS guard and
  `auto redirect`.
- Go 1.25+ to build. sing-box does not need to be installed — it is compiled in.

### Build and install

```sh
make build                                   # three binaries in ./bin, as your user
make install PREFIX=$HOME/.local             # into ~/.local/bin, no sudo
sudo make install-only                       # into /usr/local/bin, once bin/ is built
```

The Makefile refuses to build as root: `bin/` would become root-owned.

Run `barewire` (or `~/.local/bin/barewire` when the directory is not in `PATH`). The
first `Connect` starts `barewire-host` through **pkexec**; without polkit it uses
`sudo -n`, so a sudo ticket has to be fresh (`sudo -v` in another terminal).
Closing the interface stops the host.

```sh
make check      # what has to pass before a commit
make lint       # golangci-lint, when installed
make cover      # coverage summary
make preview TAB=2 W=120 H=40   # render a screen to stdout, no root and no network
make mock       # the interface against a simulated host
```

### Keys

| Key | What it does |
|---|---|
| `1`–`5` | Status, Settings, Logs, Subscriptions, Profiles |
| `tab` / `shift+tab` | next / previous tab |
| `enter` | connect on Status; activate a lever elsewhere |
| `h` `l` / `←` `→` | step a choice, a number or a toggle |
| `enter` again | type an exact value; `esc` cancels |
| `q` | quit (the host stops with the interface) |

### Settings reference

| Group | Lever | What it writes |
|---|---|---|
| Network | TUN stack | `stack` in the tun inbound; sing-tun's own stack is the *absence* of the field |
| | MTU | `inbounds[].mtu`; `auto` measures the path and subtracts the transport's overhead, the lever walks 1280…1452 and accepts a typed value, and what the path cannot carry is lowered with a line in the log |
| | Multi-queue | `inbounds[].multi_queue`, only on the own stack |
| | Auto redirect | `inbounds[].auto_redirect`; nftables rules that steer traffic which would bypass the TUN into it, including a resolver on the loopback |
| DNS | Mode | `fakeip`, a `direct` section, or no DNS section at all |
| | Preset / Servers | the resolvers, from a named pair or typed by hand |
| | Include IPv6 | IPv6 resolvers, IPv6 TUN address, `AAAA` answers |
| | On network change | restart the core, keep it, or disconnect |
| Direct | IPs / Sites / Processes | `ip_cidr`, `domain_suffix`+`domain_keyword`, `process_path_regex` → `direct` |
| Multiplexing | Enabled / Protocol / Padding | `outbounds[].multiplex`, only for `vless`, `vmess`, `trojan`, `shadowsocks`, and only without a `flow` |
| TCP Brutal | Enabled / Up / Down / Auto-detect | `outbounds[].multiplex.brutal`, only when the kernel offers `brutal` |
| Kernel | Algorithm | `net.ipv4.tcp_congestion_control`; the whole catalogue is offered, the module is loaded on apply and verified by reading it back |
| | Queue discipline | `net.core.default_qdisc` |
| | MTU probing | `net.ipv4.tcp_mtu_probing = 1` |
| | Slow start after idle | `net.ipv4.tcp_slow_start_after_idle = 0` |
| | QUIC congestion | `outbounds[].congestion_control` for TUIC |
| | Test algorithms | `kernel-test.json`, one result per server |
| Kill switch | Enabled | nftables `inet barewire`, policy drop on output |
| | On core crash | `keep` leaves the machine blocked, `flush` opens it |
| | On network change | keep or flush the ruleset; restart, keep or stop the core |
| | Flush ruleset now | remove the tables on demand |
| Theme | Accent / Accent alt | the two colours the interface uses |

### How it works

```
barewire (TUI)  ──unix socket, newline JSON──  barewire-host (root)
   settings.json                                  ├─ sing-box as a library
   profiles.json                                  ├─ nftables: kill switch + DNS guard
   subscriptions.json                             ├─ sysctl.d: kernel levers
                                                  └─ TUN, MTU and throughput probes
```

- **The configuration is built, not templated.** `internal/core/config` is a pure
  function from the lever set to a JSON document. Tests feed the result to the real
  sing-box decoder, so a document the core would reject never reaches a running
  core.
- **One host, one socket.** The socket is mode `0600`, the peer uid is checked on
  every connection, and a second host refuses to take over a socket that answers.
- **The tunnel is watched, not trusted.** A core whose API went silent is restarted
  after half a minute of silence. The second attempt drops the least proven parts —
  the alpha `sing-tun` stack and multi-queue — for that session, and says so.
- **A restart is a last resort.** Only a lever that reaches the core restarts it; a
  colour, a kill-switch policy and a network-change rule are applied in place.
- **Facts, not wishes.** The status shows what is in force: the algorithm the kernel
  reports, the MTU the core received, the mux state the core received, the ruleset
  version — not what the settings file would like.

### Files

| Path | Content |
|---|---|
| `$XDG_CONFIG_HOME/barewire/settings.json` | the lever set, mode 0600 |
| `$XDG_CONFIG_HOME/barewire/profiles.json` | named lever sets and their bindings |
| `$XDG_CONFIG_HOME/barewire/subscriptions.json` | subscription URLs and parsed nodes |
| `$XDG_CONFIG_HOME/barewire/cache.db` | the fakeip store, created on the first connect |
| `$XDG_CONFIG_HOME/barewire/kernel-backup.json` | kernel values saved before the kernel lever was applied |
| `$XDG_CONFIG_HOME/barewire/kernel-test.json` | the last congestion-control trial, per server |
| `$XDG_RUNTIME_DIR/barewire/host.sock` | the host socket, mode 0600, peer uid checked |
| `$XDG_RUNTIME_DIR/barewire/host.log` | every log line the host produced, rotated at 4 MiB; the file that survives a crash |
| `/etc/sysctl.d/99-barewire-kernel.conf` | written only when the kernel lever asks for something |

No file is created ahead of time.

### Diagnostics

```sh
barewire-ctl status                      # the same facts as JSON
barewire-ctl connections                 # the connection table
barewire-ctl subfetch https://… -ua …    # what a provider actually answered
barewire-ctl nodecheck de1               # parse and check one node
barewire-ctl addlink 'vless://…'         # import a single link
barewire-ctl speed                       # download and upload through whatever route the process has
sudo barewire-host -cleanup              # ruleset, TUN link and fakeip cache
barewire-host -version                   # what the binary was built from: a stale one is visible immediately
```

The host's own log is the first place to look:

```sh
grep -E 'core:|kernel:|mux|kill switch|dns|mtu' \
  ${XDG_RUNTIME_DIR:-/run/user/$(id -u)}/barewire/host.log | tail -30
```

`speed` needs neither the host nor `curl`: it fetches 10 MiB from Cloudflare's
endpoint, skips the first quarter as warm-up (a figure that includes slow start is
not the link) and prints both directions in Mbit/s and MiB/s.

### When something does not work

| Symptom | What it is |
|---|---|
| Link is up, nothing loads | Multiplexing is on and the server does not speak sing-mux. The client tests exactly this: one request goes through the tunnel right after connecting, and if nothing comes back the lever is turned off for the session, with the reason in the log. A server running Xray cannot speak sing-mux at all. |
| Video stalls every few minutes | Measure first: `barewire-ctl speed` with the tunnel down and up. If the numbers are fine, the tunnel is not the problem. If not, look for `mtu:` and `core:` in the log, set `MTU probing = on` and try the `mixed` stack. |
| No network after a crash | The kill-switch ruleset outlived the tunnel. The next host start clears it; `barewire-host -cleanup` does the same by hand. |
| A leak test shows the ISP | `cat /etc/resolv.conf`: `nameserver 127.0.0.53` is systemd-resolved, and its packets never enter the TUN. Turn `Auto redirect` on; the log says `dns:` at connect. |
| A lever changes nothing | Read the note under the lever and the log line that follows it. Every lever that cannot act says why. |

### Limitations and rough edges

- **Linux only.** TUN, nftables and `sysctl` are the whole design.
- **The `sing-tun` own stack in 1.15 is an alpha.** It is the default because it is
  faster and it is where sing-box is going — which is exactly why the watchdog can
  fall back to `mixed`.
- **Mux and TCP Brutal need server support.** There is no client-side trick; mux is
  at least verified with a request, and Brutal needs the `tcp_brutal` kernel module.
- **BBR is not available everywhere.** If `tcp_bbr` is neither built in nor a module,
  `cubic` and the rest remain; the interface says what it will load on apply.
- **DoH on 443 is not intercepted** — that would mean breaking TLS.
- **A resolver on the loopback** is caught only by `auto redirect`, which needs
  nftables.
- **No reconnect loop on visible failures:** a failed start is a disconnect with the
  reason, and a wedged tunnel is replaced by the watchdog, not retried forever.
- **No license file yet.** Choose one deliberately before publishing.
- **No GUI and no mobile clients** — and none are planned: this is a TUI by design.

### Development

```sh
make check                      # gofmt, go vet, the whole test suite
make lint                       # golangci-lint, when installed
go test -race ./internal/...    # the concurrency paths: host, socket, core, tui
```

The suite covers the real sing-box decoder on generated configurations, the nftables
ruleset text (checked with `nft -c` when the tool is installed), the socket protocol
over a real unix socket, the sysctl backup and restore path, the subscription
parsers, the log ring buffer, text geometry and every screen layout, the
congestion-control trial scoring, and the failure paths: a core that cannot open the
TUN device must end up disconnected with a reason.

```
cmd/barewire/          the interface
cmd/barewire-host/     the privileged host: sing-box, TUN, nftables, sysctl
cmd/barewire-ctl/      shell levers over the same socket, JSON out
cmd/tuipreview/        renders one screen to stdout
internal/config/       settings, profiles, subscriptions, validation
internal/core/         the embedded core: decode, start, stop, statistics
internal/core/config/  the configuration builder (pure functions, tested)
internal/host/         the privileged half: link lifecycle, levers, kernel
internal/hostsock/     the socket protocol
internal/system/       nftables, sysctl, probes, route watching
internal/tui/          screens, input, layout
internal/subscription/ Clash YAML and URI parsing
```

Design rules worth keeping: the builder stays pure and free of option structs, the
interface never decides anything on its own, every failure is reported rather than
retried quietly, and a lever that cannot act says so.

### Plans

The project is alive and being developed; the near queue:

1. Get to the bottom of the alpha `sing-tun` stack stalls, and report upstream if the
   cause is in the library.
2. A `make race` target and a `-race` run for the whole repository.
3. A license, git history and the first releases.
4. More presets for specific networks (mobile carrier, Wi-Fi with a captive portal).
5. Detect in-browser DoH and say how to turn it off.

### Acknowledgements

Barewire is a front end. The tunnel, the TUN stack and the protocols are
[sing-box](https://github.com/SagerNet/sing-box) and
[sing-tun](https://github.com/SagerNet/sing-tun). The interface uses
[Bubble Tea](https://github.com/charmbracelet/bubbletea).
