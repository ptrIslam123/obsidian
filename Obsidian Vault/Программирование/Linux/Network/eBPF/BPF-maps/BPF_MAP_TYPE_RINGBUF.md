`BPF_MAP_TYPE_RINGBUF` — это высокопроизводительная структура данных для **односторонней передачи данных** из eBPF-программ (ядро) в пользовательское пространство. Она пришла на смену устаревшему `BPF_MAP_TYPE_PERF_EVENT_ARRAY` и обеспечивает **меньшие накладные расходы** и **более эффективное использование памяти**.

---

## **1. Основные характеристики**
| Характеристика          | Описание |
|-------------------------|----------|
| **Тип данных**          | Кольцевой буфер (ring buffer) |
| **Назначение**          | Передача событий/данных из ядра в userspace |
| **Производительность**  | Высокая (меньшие накладные расходы, чем у perf_buffer) |
| **Семантика доступа**   | Single Producer, Single Consumer (SPSC) |
| **Размер**              | Фиксируется при создании (обычно несколько мегабайт) |
| **Поддержка в ядре**    | Linux 5.8+ |

---

## **2. Объявление в eBPF-программе**
```c
struct {
    __uint(type, BPF_MAP_TYPE_RINGBUF);
    __uint(max_entries, 1 << 24); // 16 MB (размер буфера)
} ringbuf SEC(".maps");
```
- **`max_entries`** — размер буфера **должен быть степенью двойки** (например, `1 << 20` = 1 MB).
- **Нет `key` и `value`** — данные записываются как произвольные структуры.

---

## **3. Работа с `ringbuf` в eBPF (ядро)**

#### **1. `bpf_ringbuf_reserve()` — резервирование места в буфере**
**Назначение**:  
Выделяет непрерывный участок памяти в кольцевом буфере для записи данных.

**Сигнатура**:
```c
void *bpf_ringbuf_reserve(struct bpf_map *ringbuf, u64 size, u64 flags);
```

**Параметры**:
- `ringbuf` — указатель на кольцевой буфер (объявленный в `SEC(".maps")`).
- `size` — размер данных, которые нужно записать (в байтах).
- `flags`:
  - `0` — стандартное поведение.
  - `BPF_RB_NO_WAKEUP` / `BPF_RB_FORCE_WAKEUP` (влияют на уведомление при `submit`).

**Возвращаемое значение**:
- Указатель на зарезервированную память (если место есть).
- `NULL` — если буфер переполнен.

**Пример**:
```c
struct event *e = bpf_ringbuf_reserve(&ringbuf, sizeof(struct event), 0);
if (!e) {
    bpf_printk("Buffer full!");
    return 0;
}
```

**Как работает**:
1. Проверяет, есть ли в буфере свободное место.
2. Если есть — возвращает указатель на память, которую можно заполнить.
3. Если нет — возвращает `NULL`.

---

#### **2. `bpf_ringbuf_submit()` — фиксация данных**
**Назначение**:  
Публикует зарезервированные данные, делая их доступными для чтения в userspace.

**Сигнатура**:
```c
void bpf_ringbuf_submit(void *data, u64 flags);
```

**Параметры**:
- `data` — указатель, полученный от `bpf_ringbuf_reserve()`.
- `flags`:
  - `0` — стандартное уведомление (пользовательский процесс получит сигнал).
  - `BPF_RB_NO_WAKEUP` — данные записаны, но уведомление не отправлено.
  - `BPF_RB_FORCE_WAKEUP` — форсированное уведомление (даже если буфер почти пуст).

**Пример**:
```c
bpf_ringbuf_submit(e, 0);  // С уведомлением
bpf_ringbuf_submit(e, BPF_RB_NO_WAKEUP);  // Без уведомления
```

**Что происходит**:
1. Данные помещаются в буфер.
2. Если `flags != BPF_RB_NO_WAKEUP` — userspace получит сигнал о новом событии.

---

#### **3. `bpf_ringbuf_discard()` — отмена записи**
**Назначение**:  
Отменяет резервирование места (данные не попадут в буфер).

**Сигнатура**:
```c
void bpf_ringbuf_discard(void *data, u64 flags);
```

**Параметры**:
- `data` — указатель от `bpf_ringbuf_reserve()`.
- `flags`:
  - `0` — стандартное поведение.
  - `BPF_RB_NO_WAKEUP` / `BPF_RB_FORCE_WAKEUP` (аналогично `submit`).

**Пример**:
```c
if (e->pid == 0) {
    bpf_ringbuf_discard(e, 0);  // Не записывать события от PID=0
    return 0;
}
```

**Когда использовать**:
- Если данные стали неактуальными (например, фильтрация в eBPF).
- Для экономии места в буфере.

---

#### **4. `bpf_ringbuf_query()` — получение статистики**
**Назначение**:  
Возвращает информацию о состоянии буфера (кол-во данных, свободное место и т.д.).

**Сигнатура**:
```c
u64 bpf_ringbuf_query(struct bpf_map *ringbuf, u64 flags);
```

**Параметры**:
- `ringbuf` — указатель на буфер.
- `flags` — тип запрашиваемой информации:
  - `BPF_RB_AVAIL_DATA` — объём данных, готовых для чтения.
  - `BPF_RB_RING_SIZE` — общий размер буфера.
  - `BPF_RB_CONS_POS` / `BPF_RB_PROD_POS` — позиции потребителя и производителя.

**Пример**:
```c
u64 avail_data = bpf_ringbuf_query(&ringbuf, BPF_RB_AVAIL_DATA);
bpf_printk("Available data: %llu bytes", avail_data);
```

**Вывод**:
```
Available data: 128 bytes
```

---

#### **5. `bpf_ringbuf_output()` — запись без резервирования**
**Назначение**:  
Аналог `bpf_ringbuf_reserve()` + `bpf_ringbuf_submit()` в одном вызове (удобно для простых случаев).

**Сигнатура**:
```c
long bpf_ringbuf_output(struct bpf_map *ringbuf, void *data, u64 size, u64 flags);
```

**Параметры**:
- `ringbuf` — указатель на буфер.
- `data` — указатель на данные для записи.
- `size` — размер данных.
- `flags` — аналогично `bpf_ringbuf_submit()`.

**Пример**:
```c
struct event e = { .pid = 123 };
bpf_ringbuf_output(&ringbuf, &e, sizeof(e), 0);
```

**Плюсы**:
- Удобно для небольших данных.
- Не требует ручного управления памятью.

**Минусы**:
- Медленнее, чем `reserve` + `submit` (из-за копирования данных).

---

## **Сравнение функций**
| Функция                 | Когда использовать?                    | Производительность |
| ----------------------- | -------------------------------------- | ------------------ |
| `bpf_ringbuf_reserve()` | Для сложных данных (заполнение полей). | ⚡️ Высокая         |
| `bpf_ringbuf_output()`  | Для простых данных (структур).         | 🐢 Средняя         |
| `bpf_ringbuf_query()`   | Для мониторинга состояния буфера.      | 🛠️ Сервисная      |

---

## **Пример полного цикла**
```c
struct event *e = bpf_ringbuf_reserve(&ringbuf, sizeof(struct event), 0);
if (!e) return 0;

e->pid = bpf_get_current_pid_tgid();
bpf_ringbuf_submit(e, 0);  // Отправляем с уведомлением

u64 avail = bpf_ringbuf_query(&ringbuf, BPF_RB_AVAIL_DATA);
bpf_printk("Available: %llu", avail);
```

---

## **4. Чтение данных в пользовательском пространстве (libbpf)**
### **Пример на C (libbpf)**
```c
#include <bpf/libbpf.h>
#include <stdio.h>

// Колбэк для обработки событий
static int handle_event(void *ctx, void *data, size_t size) {
    struct event *e = data;
    printf("PID: %d, COMM: %s, Time: %llu\n", e->pid, e->comm, e->timestamp);
    return 0;
}

int main() {
    struct ring_buffer *rb = NULL;
    struct bpf_object *obj;
    int map_fd;

    // Загрузка eBPF-программы
    obj = bpf_object__open("my_prog.bpf.o");
    bpf_object__load(obj);

    // Получение fd ringbuf-буфера
    map_fd = bpf_object__find_map_fd_by_name(obj, "ringbuf");

    // Инициализация ring buffer
    rb = ring_buffer__new(map_fd, handle_event, NULL, NULL);

    while (1) {
        ring_buffer__poll(rb, 1000); // Ожидание событий (таймаут 1 сек)
    }

    ring_buffer__free(rb);
    return 0;
}
```

---

## Пример использования в ядре (eBPF программа)
```c
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>

// Структура события
struct event {
    u32 pid;
    char message[32];
};

// Кольцевой буфер
struct {
    __uint(type, BPF_MAP_TYPE_RINGBUF);
    __uint(max_entries, 1 << 20); // 1 MB
} ringbuf SEC(".maps");

// Функция для логирования события
static void log_event(const char *msg) {
    struct event *e;
    
    // Резервируем место в буфере
    e = bpf_ringbuf_reserve(&ringbuf, sizeof(struct event), 0);
    if (!e) return; // Если буфер полон

    e->pid = bpf_get_current_pid_tgid();
    bpf_probe_read_kernel_str(e->message, sizeof(e->message), msg);

    // Отправляем событие:
    // 1-е событие — с уведомлением (по умолчанию)
    // 2-е событие — без уведомления (BPF_RB_NO_WAKEUP)
    if (msg[0] == 'A') {
        bpf_ringbuf_submit(e, 0); // Уведомляем пользователя
    } else {
        bpf_ringbuf_submit(e, BPF_RB_NO_WAKEUP); // Без уведомления
    }
}

// Точка входа (например, при системном вызове)
SEC("kprobe/sys_execve")
int handle_execve(void *ctx) {
    log_event("Event A: Process started");
    log_event("Event B: No wakeup");
    return 0;
}

char _license[] SEC("license") = "GPL";
```

## **Пользовательская программа (Python)**
```python
from bcc import BPF
import time

# Загружаем eBPF-программу
bpf = BPF(src_file="ringbuf_example.bpf.c")

# Колбэк для обработки событий
def handle_event(ctx, data, size):
    event = bpf["ringbuf"].event(data)
    print(f"PID: {event.pid}, Message: {event.message.decode()}")

# Открываем кольцевой буфер('ringbuf') и подписываемся на события
bpf["ringbuf"].open_ring_buffer(handle_event)

print("Listening for events... Press Ctrl+C to exit")
try:
    while True:
        # Ожидаем события (таймаут 1 сек)
        bpf.ring_buffer_consume()
        time.sleep(0.1)
except KeyboardInterrupt:
    pass
```

1. **При вызове `sys_execve`** (например, запуск программы):
    - eBPF-программа запишет **2 события** в `ringbuf`:
        - `Event A` — с уведомлением (`flags = 0`).
        - `Event B` — без уведомления (`BPF_RB_NO_WAKEUP`).
2. **Пользовательская программа**:
    - Сразу получит уведомление о `Event A` и выведет его.
    - `Event B` будет прочитан **только при следующем вызове `ring_buffer_consume()`** (из-за `BPF_RB_NO_WAKEUP`).

---
## **5. Преимущества `BPF_MAP_TYPE_RINGBUF`**
### **Перед `BPF_MAP_TYPE_PERF_EVENT_ARRAY` (perf_buffer)**
✅ **Меньшие накладные расходы** (нет копирования данных через `perf_event_output`).  
✅ **Более эффективное использование памяти** (общая память между ядром и userspace).  
✅ **Автоматическое управление переполнением** (старые данные перезаписываются).  
✅ **Поддержка "zero-copy"** (пользовательское пространство читает данные напрямую).  

### **Перед `BPF_MAP_TYPE_QUEUE/STACK`**
✅ **Гораздо большая пропускная способность**.  
✅ **Не блокируется при переполнении** (старые данные перезаписываются).  
✅ **Подходит для потоковой передачи** (например, трейсинг системных вызовов).  

---

## **6. Ограничения**
❌ **Только однонаправленная передача** (ядро → userspace).  
❌ **Нет гарантии доставки** (при переполнении старые данные теряются).  
❌ **Требует Linux 5.8+**.  

---

## **7. Примеры использования**
1. **Трейсинг системных вызовов** (например, `openat`, `execve`).  
2. **Мониторинг сетевых пакетов** (XDP/TC-программы).  
3. **Профилирование производительности** (передача метрик CPU/памяти).  
4. **Логирование событий ядра** (например, блокировка файлов).  

---

## **8. Производительность**
- **Пропускная способность**: до **миллионов событий в секунду**.  
- **Задержка**: микросекунды (зависит от размера буфера).  
- **Потребление CPU**: значительно меньше, чем у `perf_buffer`.  
