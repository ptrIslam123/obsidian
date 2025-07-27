**`perf` — это мощный инструмент для профилирования в Linux, поддерживающий множество параметров и событий.**

## **1. Основные команды `perf`**

| Команда         | Описание                                                |
| --------------- | ------------------------------------------------------- |
| `perf stat`     | Сбор общей статистики (инструкции, cache-misses, время) |
| `perf record`   | Запись профиля в файл (`perf.data`)                     |
| `perf report`   | Анализ записанного профиля                              |
| `perf top`      | Режим реального времени (как `top`, но для CPU-функций) |
| `perf annotate` | Показывает ассемблерный код с аннотацией                |
| `perf script`   | Экспорт данных для скриптовой обработки                 |

---
##  **2. Основные аргументы `perf stat`**
```bash
perf stat [опции] [программа]
```

| Аргумент       | Описание                                             |
| -------------- | ---------------------------------------------------- |
| `-e <событие>` | Замер конкретного события (например, `cache-misses`) |
| `-a`           | Замер всех ядер CPU                                  |
| `-p <PID>`     | Профилирование работающего процесса                  |
| `-r <N>`       | Запуск программы `N` раз и усреднение результатов    |
| `-d`           | Дополнительные события (L1, LLC, TLB)                |
| `-I <мс>`      | Периодический вывод статистики                       |
| `--per-core`   | Статистика по каждому ядру                           |
| `--per-socket` | Статистика по сокетам CPU                            |

---
## **3. Аппаратные события (`perf list`)**

Доступные события можно посмотреть:
```bash
perf list
```

### Основные группы событий:

| Категория           | Примеры событий                              |
| ------------------- | -------------------------------------------- |
| **CPU-циклы**       | `cycles`, `instructions`                     |
| **Кэш**             | `L1-dcache-load-misses`, `LLC-load-misses`   |
| **Ветвления**       | `branch-misses`, `branch-load-misses`        |
| **Страницы памяти** | `page-faults`, `dTLB-load-misses`            |
| **Ресурсы CPU**     | `stalled-cycles-frontend`, `resource-stalls` |

---

## **4. Аргументы `perf record` (запись профиля)**

```bash
perf record [опции] [программа]
```

|Аргумент|Описание|
|---|---|
|`-g`|Запись call-graph (стек вызовов)|
|`-F <частота>`|Частота семплирования (например, `-F 99`)|
|`-p <PID>`|Профилирование процесса|
|`-o <файл>`|Сохранение в файл (по умолчанию `perf.data`)|
|`-e <событие>`|Запись по конкретному событию|
|`--call-graph dwarf`|Точный call-graph (требует `-fno-omit-frame-pointer`)|

---

## **`perf top` (режим реального времени)**

```bash
perf top [опции]
```

| Аргумент       | Описание                                   |
| -------------- | ------------------------------------------ |
| `-e <событие>` | Фильтрация по событию                      |
| `-p <PID>`     | Мониторинг процесса                        |
| `-K`           | Скрыть ядерные функции                     |
| `-U`           | Показывать только пользовательские функции |

---
## **Кэш-промахи/попадания**

```bash
sudo perf stat -e cache-references,cache-misses,L1-dcache-loads,L1-dcache-load-misses,LLC-loads,LLC-load-misses ./program
```
- **`cache-references`** — общее число обращений к кэшу.
- **`cache-misses`** — промахи в кэше.
	- **`L1-dcache-loads`** — загрузки из L1-кэша.
- **`L1-dcache-load-misses`** — промахи в L1.
- **`LLC-loads`** — загрузки из Last-Level Cache (L3).
- **`LLC-load-misses`** — промахи в L3 (очень дорогие!).
### 🔹 **Интерпретация**
- Высокий `L1-dcache-load-misses` → Плохая локальность данных.
- Высокий `LLC-load-misses` → Частые обращения к RAM.

## **I/O-операции (read/write, select, epoll)**

#### **События для анализа системных вызовов и I/O:**
```bash
sudo perf stat -e syscalls:sys_enter_read,syscalls:sys_exit_read,syscalls:sys_enter_write,syscalls:sys_exit_write,syscalls:sys_enter_epoll_wait,syscalls:sys_exit_epoll_wait ./program
```

- **`syscalls:sys_enter_*`** — вход в системный вызов.
- **`syscalls:sys_exit_*`** — выход из системного вызова.

#### **Анализ `epoll` и `select`**
```bash
perf stat -e 'syscalls:sys_enter_epoll*' ./program
```

Покажет, сколько раз вызывался `epoll_wait`.

## **NUMA-промахи (локальные vs удалённые доступы)**

#### **События для анализа NUMA:**
```bash

```


## **Анализ CPU-циклов**
```bash
sudo perf stat -e cycles,instructions,stalled-cycles-frontend,stalled-cycles-backend,branch-misses,branch-load-misses ./program
```

- **`cycles`** — общее число циклов CPU.
- **`instructions`** — выполненные инструкции.
- **`stalled-cycles-frontend`** — циклы, потерянные из-за нехватки инструкций (например, промахов в кэше).
- **`stalled-cycles-backend`** — циклы, потерянные из-за нехватки данных (например, ожидание RAM).
### 🔹 **Как найти узкие места?**
```bash
perf record -e cycles -g ./program
perf report -n --stdio | head -20
```

Покажет функции, которые тратят больше всего CPU.


## **Анализ блокировок (futex, mutex)**
##### Можно использовать `sudo perf list | grep -E 'sched:|lock:'` для опеределения доступных в системе событий для анализа блокировок

#### Анализ contention (блокировок)
```bash
sudo perf stat -e 'lock:contention_begin','lock:contention_end','sched:sched_stat_blocked' ./program
```
#### Анализ futex
```bash
sudo perf stat -e 'syscalls:sys_enter_futex','syscalls:sys_exit_futex' ./program
```



## Анализ переключений контекста
```bash
sudo perf stat -e 'sched:sched_switch','sched:sched_wakeup','sched:sched_stat_runtime' ./program
```

## Посмотреть список поддерживаемых событий в системе для отслеживания через perf(eBPF)

```bash
sudo find /sys/kernel/debug/tracing/events/
```

## Анализ потребления функциями ресурсов CPU

```bash
sudo perf record -F 99 -g -p <PID> -o <PERF_REPORT_FILE>
perf report -i <PERF_REPORT_FILE>
```

Пример:
```bash
sudo perf record -F 99 -g -p 374356 -o perf_data.perf
perf report -i perf_data.perf
```