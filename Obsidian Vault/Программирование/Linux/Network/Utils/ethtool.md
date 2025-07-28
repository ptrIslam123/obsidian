`ethtool` — это мощная утилита в Linux для **настройки и диагностики сетевых интерфейсов**. Она позволяет просматривать и изменять параметры сетевых карт (NIC), такие как скорость, дуплекс, автосогласование, очереди (queues), offload-опции и многое другое.

---
## **Основные команды `ethtool`**

### **1. Просмотр общей информации (`-i`, `--driver`)**
```bash
ethtool <интерфейс>  # Например: ethtool enp1s0
```
**Вывод:**
```
Settings for enp1s0:
	Supported ports: [ TP ]          # Поддерживаемые типы портов (витая пара, оптика)
	Supported link modes:  10baseT/Half 10baseT/Full  # Доступные скорости
	                         100baseT/Half 100baseT/Full
	                         1000baseT/Full
	Supported pause frame use: No    # Поддержка flow control (PAUSE frames)
	Supports auto-negotiation: Yes   # Поддержка автосогласования
	Advertised link modes:  ...      # Какие режимы объявляются при автосогласовании
	Speed: 1000Mb/s                  # Текущая скорость
	Duplex: Full                     # Режим дуплекса (Full/Half)
	Auto-negotiation: on             # Включено ли автосогласование
	Port: Twisted Pair               # Тип порта
	PHYAD: 1                         # Адрес PHY-устройства
	Transceiver: internal            # Внутренний/внешний трансивер
	Link detected: yes               # Есть ли link (подключение)
```

#### **Дополнительная информация о драйвере (`-i`)**
```bash
ethtool -i enp1s0
```
**Вывод:**
```
driver: e1000e              # Драйвер устройства
version: 5.15.0-60-generic  # Версия драйвера
firmware-version: 1.20-2    # Прошивка
bus-info: 0000:00:1f.6      # PCI-адрес
supports-statistics: yes    # Поддержка статистики
supports-test: yes          # Поддержка тестов
supports-eeprom-access: yes # Можно ли читать EEPROM
supports-register-dump: yes # Доступен ли дамп регистров
```

---

### **2. Настройка скорости и дуплекса (`-s`)**
Можно **принудительно выставить** скорость и дуплекс (если автосогласование отключено):
```bash
ethtool -s enp1s0 speed 100 duplex full autoneg off
```
- `speed` — `10`, `100`, `1000`, `2500`, `10000` (зависит от NIC).
- `duplex` — `full` (полный дуплекс) или `half` (полудуплекс).
- `autoneg` — `on`/`off` (включить/выключить автосогласование).

⚠ **Важно:** Не все сетевые карты поддерживают ручную настройку. Если интерфейс не поднялся, верните автосогласование:
```bash
ethtool -s enp1s0 autoneg on
```

---

### **3. Управление offload-опциями (`-k`, `--offload`)**
Offload-опции позволяют **разгрузить CPU**, передавая задачи (например, подсчёт контрольных сумм) на сетевую карту.

#### **Просмотр текущих offload-опций (`-k`)**
```bash
ethtool -k enp1s0
```
**Вывод:**
```
rx-checksumming: on       # Проверка контрольных сумм на приём
tx-checksumming: on       # Расчёт контрольных сумм на передачу
scatter-gather: on        # Scatter-Gather DMA
tcp-segmentation-offload: on  # TSO (разбивка TCP-пакетов)
udp-fragmentation-offload: off
generic-segmentation-offload: on  # GSO
generic-receive-offload: on       # GRO
...
```

#### **Изменение offload-опций (`-K`)**
```bash
ethtool -K enp1s0 tx off rx off  # Отключить checksum offload
ethtool -K enp1s0 tso on gso on  # Включить TSO и GSO
```
🔹 **Когда отключать offload?**  
- При проблемах с сетью (например, некорректные checksums).
- Для тестирования производительности.

---

### **4. Управление очередями (`-l`, `--show-channels`, `--set-channels`)**
Современные NIC поддерживают **множественные очереди (RSS, RPS, XPS)** для балансировки нагрузки по ядрам CPU.

#### **Просмотр текущих очередей (`-l`)**
```bash
ethtool -l enp1s0
```
**Вывод:**
```
Channel parameters for enp1s0:
Pre-set maximums:
RX:             4  # Максимум RX-очередей
TX:             4  # Максимум TX-очередей
Other:          1
Combined:       4  # Комбинированные (RX+TX) очереди
Current hardware settings:
RX:             2  # Текущее значение RX
TX:             2  # Текущее значение TX
Other:          1
Combined:       2  # Текущее комбинированных
```

#### **Изменение количества очередей (`-L`)**
```bash
ethtool -L enp1s0 combined 4  # Установить 4 комбинированные очереди
ethtool -L enp1s0 rx 4 tx 4   # Раздельные RX/TX очереди
```
🔹 **Оптимальное значение:** Обычно равно количеству логических ядер CPU.

---

### **5. Статистика (`-S`, `--statistics`)**
```bash
ethtool -S enp1s0
```
**Вывод:**
```
rx_packets: 123456   # Принято пакетов
tx_packets: 789012   # Передано пакетов
rx_bytes: 45678901   # Принято байт
tx_bytes: 98765432   # Передано байт
rx_errors: 0         # Ошибок приёма
tx_errors: 5         # Ошибок передачи
...
```
🔹 **Сброс статистики:**
```bash
ethtool --reset-statistics enp1s0  # Не все драйверы поддерживают!
```

---

### **6. Wake-on-LAN (`-w`, `--wol`)**
Настройка пробуждения компьютера по сети (Wake-on-LAN):
```bash
ethtool enp1s0 | grep -i wake  # Проверить поддержку
ethtool -s enp1s0 wol g        # Включить WOL (g = MagicPacket)
```
🔹 **Параметры WOL:**
- `d` — отключено.
- `p` — Wake on PHY activity.
- `u` — Wake on unicast.
- `m` — Wake on multicast.
- `b` — Wake on broadcast.
- `a` — Wake on ARP.
- `g` — Wake on MagicPacket (стандартный WOL).

---

## 🔥 **Практические примеры использования**
### **1. Оптимизация сетевой карты для сервера**
```bash
# Установить 8 очередей
ethtool -L enp1s0 combined 8

# Включить все offload-опции
ethtool -K enp1s0 tx on rx on tso on gso on gro on

# Проверить настройки
ethtool enp1s0
ethtool -k enp1s0
```

### **2. Диагностика проблем с сетью**
```bash
# Проверить, есть ли link
ethtool enp1s0 | grep "Link detected"

# Если нет link, проверить автосогласование
ethtool -s enp1s0 autoneg on

# Посмотреть ошибки
ethtool -S enp1s0 | grep errors
```

---

## ⚠️ **Ограничения**
- Не все сетевые карты поддерживают все функции.
- Изменения через `ethtool` **сбрасываются после перезагрузки**.  
  Чтобы сохранить настройки, используйте:
  - `systemd-networkd` (для systemd).
  - Скрипты в `/etc/network/if-pre-up.d/`.
  - `NetworkManager` (если используется).
