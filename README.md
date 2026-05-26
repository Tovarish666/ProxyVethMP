# PSC — Proxy Control Service

Управление mobile proxy инфраструктурой на Proxmox VE.
Три компонента, один инструмент.

## Стек

| # | Компонент | Что делает |
|---|-----------|-----------|
| 1 | **Ubuntu VM** | Гостевая ВМ (cloud-init, SSH, DHCP) |
| 2 | **mobileproxy.space** | mproxy + nodejs-server — основной прокси-демон |
| 3 | **PSC** | SOCKS5 → veth-пара → mproxy (source routing) |

### Как работает PSC

Каждый модем получает изолированный network namespace с TUN-интерфейсом:

```
SOCKS5 (модем N)
      │
  tun2socks
      │
    tun{N}  ←── ns_{N} ──── veth_ext{N}_ns ──── 192.168.N.254
                                                       │
              HOST ──── veth_ext{N}_host ──── 192.168.N.100
                                                       │
                                                   mproxy
                                              (source routing)
```

mproxy отправляет трафик через `192.168.N.100` → veth → namespace → tun2socks → реальный SOCKS5 (мобильный модем).

## Быстрый старт

```bash
# На хосте Proxmox, от root:
bash <(curl -s https://raw.githubusercontent.com/Tovarish666/ProxyVethMP/main/pscctl.sh)
```

Меню само предложит что установить — VM, PSC, mp.space, или всё сразу.

## Команды psc (внутри VM)

| Команда | Описание |
|---------|----------|
| `psc status` | Таблица NS + WAN IP каждого модема |
| `psc check` | Диагностика: скорость + ping + 2ip |
| `psc up [N\|all]` | Поднять namespace(ы) |
| `psc down [N\|all]` | Опустить namespace(ы) |
| `psc restart [N\|all]` | Перезапустить |
| `psc sync` | Синхронизировать конфиг из Google Sheets |
| `psc autosync` | Sync с применением diff (добавить/удалить/перезапустить) |
| `psc watchdog` | Разовая проверка watchdog |
| `psc watchdog-loop` | Watchdog-демон (бесконечный цикл) |
| `psc show-config` | Показать конфиг модемов |
| `psc cleanup` | Полная очистка (NS, iptables, veth) |

### `psc check` — диагностика

```
  N  │  WAN IP         │  DL        │  Ping  │ Статус
  ───┼─────────────────┼────────────┼────────┼────────
   1 │ 1.2.3.4         │ 15.2 Mb/s  │  45ms  │ OK
   2 │ 1.2.3.5         │ 12.8 Mb/s  │  67ms  │ WLIST
   3 │ —               │    —       │   —    │ DEAD
```

| Статус | Значение |
|--------|----------|
| **OK** | Всё работает |
| **WLIST** | Белый список: скорость есть, 2ip.ru заблокирован |
| **DEAD** | Модем не отвечает |

Скорость измеряется через yaspeed (если установлен через `YASPEED_URL`) или curl + ping как fallback.

## Формат Google Sheets

Два поддерживаемых формата:

**Компактный** (один столбец proxy):

| n | proxy |
|---|-------|
| 1 | host:port:login:password |

**Раздельный**:

| n | proxy_host | proxy_port | login | password |
|---|-----------|-----------|-------|---------|

URL для pscctl: `https://...pub?gid=XXXX&single=true&output=csv`

## Файловая структура

```
/etc/psc/
├── config.json          # конфиг модемов (из Google Sheets)
├── env                  # переменные окружения
└── logs/
    └── watchdog.log

/usr/local/bin/
├── psc.py               # основной Python-скрипт
├── psc -> ...           # симлинк
└── tun2socks            # бинарь tun2socks v2.5.2

/etc/systemd/system/
├── psc.service           # up all при старте системы
├── psc-watchdog.service  # watchdog-демон
├── psc-autosync.service  # разовый sync
└── psc-autosync.timer    # sync каждые 5 минут
```

## Переменные окружения (`/etc/psc/env`)

| Переменная | Описание | Дефолт |
|------------|----------|--------|
| `SHEET_CSV_URL` | URL CSV-экспорта Google Sheets | — |
| `SHEET_ID` + `SHEET_GID` | ID таблицы + ID листа (альтернатива URL) | — |
| `YASPEED_URL` | URL бинаря yaspeed для `psc check` | — |
| `WATCHDOG_INTERVAL` | Интервал watchdog (сек) | 60 |
| `WATCHDOG_WAN_EVERY` | WAN-проверка каждые N итераций | 10 |
| `WATCHDOG_MAX_RESTART` | Макс. перезапусков на модем | 3 |

## Требования

- Proxmox VE 7+
- Python 3.10+
- tun2socks — скачивается автоматически при установке
- Ubuntu 24.04 cloud image — скачивается автоматически
