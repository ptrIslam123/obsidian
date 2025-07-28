Ядро позволяет процессу создавать сокеты в семействе адресов Address Family Express Data Path (AF_XDP). Это особый тип сокета, который в сочетании с программой XDP может выполнять полный или частичный обход ядра. Обход сетевого стека ядра может повысить производительность в определенных случаях использования. Сокет, созданный в семействе адресов AF_XDP, также называется XSK (XDP Socket).

**Примерами таких вариантов использования являются:** 
* Пользовательские реализации протоколов — если ядро ​​не понимает пользовательский протокол, оно будет выполнять много ненужной работы, обходя ядро ​​и передавая трафик процессу, который обрабатывает его правильно, избегая накладных расходов. 
* Защита от DDoS — если требуется сложная обработка нескольких пакетов, программы eBPF не справляются, поэтому может потребоваться пересылка трафика в пользовательское пространство для анализа. 
* Оптимизация, специфичная для приложения — сетевой стек Linux по необходимости должен обрабатывать множество протоколов и пограничных случаев, которые неприменимы к рабочим нагрузкам, которые вы запускаете. Это означает оплату производительности за функции, которые вы не используете. Хотя это и нелегко, можно реализовать пользовательский сетевой стек, соответствующий вашим потребностям, чтобы выжать каждую каплю производительности.

Весь входящий трафик сначала обрабатывается программой XDP, она может принять решение о том, какой трафик передать в стек, а какой обойти. Это мощно, поскольку позволяет пользователю обходить трафик для очень конкретных приложений, портов и/или протоколов, не нарушая нормальную обработку пакетов. В отличие от других методов обхода ядра, таких как PACKET_MMAP или PF_RING, которые требуют от вас обработки всего трафика и повторной реализации каждого протокола, необходимого для функционирования хоста.

---
### Как это работает 
XSK состоит из нескольких «частей», которые работают вместе и должны быть настроены и связаны правильным образом, чтобы получить работающий XSK.

![[0_initial.svg]]


Сокет (не отображается) связывает все вместе и дает процессу файловый дескриптор, который используется в системных вызовах. Далее идет «UMEM», который представляет собой часть памяти, выделенную процессом, которая используется для фактических данных пакета. Он содержит несколько фрагментов, поэтому действует очень похоже на массив. Наконец, есть 4 кольцевых буфера: RX, TX, FILL и COMPLETION. Эти кольцевые буферы используются для передачи владельца и намерения для различных фрагментов UMEM. Подробнее об этом в разделе Получение и отправка пакетов.

Эти четыре очереди — **`RX`, `TX`, `Fill`, `Completion`** — являются **основой архитектуры AF_XDP**. Они не хранят сами пакеты, а передают **указатели (адреса)** на буферы в общей памяти (UMEM), что исключает копирование данных и позволяет достичь производительности на уровне гигабит в секунду.

## Общая архитектура

```
[Приложение] ⇄ [Кольца] ⇄ [Драйвер] ⇄ [Сетевая карта]
                ↑    ↑
             [UMEM — общая память]
```

- **UMEM** — выделенный блок памяти, содержащий буферы для пакетов.
- **Кольца** — массивы фиксированного размера, через которые передаются **указатели** на буферы в UMEM.
- Все операции происходят **пачками (bursts)** — по 32–64 элементов за раз.

---

## **Fill Queue (FQ)** — очередь заполнения

### 🔹 Назначение
**Указать драйверу, где лежат свободные буферы для приёма пакетов.**

> 💡 Fill Queue — это **список адресов пустых буферов** в UMEM, которые приложение предоставляет драйверу.

1. Приложение **заполняет FQ** указателями на свободные буферы (например, `0`, `2048`, `4096`, ...).
2. Драйвер **забирает** указатель из FQ.
3. Копирует пришедший пакет в соответствующий буфер в UMEM.
4. Кладёт указатель на этот буфер в **RX Ring**, чтобы приложение его забрало.

### Цикл жизни буфера
```
FQ → RX Ring → обработка → возврат в FQ
```

---

### Ключевые функции (из `libbpf`)

```cpp
// зарезервировать место
xsk_ring_prod__reserve(&p->umem_fq, count, &pos);

// записать адрес
*xsk_ring_prod__fill_addr(&p->umem_fq, pos + i) = addr;

// подтвердить
xsk_ring_prod__submit(&p->umem_fq, count);
```

### Пример инициализации

```cpp
for (i = 0; i < umem_fq_size; i++) {
    *xsk_ring_prod__fill_addr(&p->umem_fq, pos + i) = i * frame_size;
}
xsk_ring_prod__submit(&p->umem_fq, umem_fq_size);
```

> Это даёт драйверу доступ ко **всем буферам** в UMEM.

---

## **Completion Queue (CQ)** — очередь завершения

### 🔹 Назначение
**Сообщить приложению, что буферы, использованные для отправки, больше не нужны и могут быть повторно использованы.**

> 💡 CQ — это **список адресов буферов, которые драйвер уже отправил** и освободил.

1. Приложение кладёт пакет в буфер и отправляет его через **TX Ring**.
2. Драйвер отправляет пакет и **возвращает указатель на буфер** в **CQ**.
3. Приложение забирает указатель из CQ и **возвращает буфер в FQ**.

### Цикл жизни буфера (при отправке)
```
FQ → TX Ring → отправка → CQ → FQ
```

### Ключевые функции

```cpp
// проверить
xsk_ring_cons__peek(&p->umem_cq, count, &pos);

// прочитать адрес
addr = *xsk_ring_cons__comp_addr(&p->umem_cq, pos + i);

// освободить слоты
xsk_ring_cons__release(&p->umem_cq, count);
```


Без CQ приложение **не узнало бы**, когда можно повторно использовать буфер. Оно могло бы попытаться перезаписать буфер, пока драйвер ещё отправляет пакет → **повреждение данных**.

CQ — это **механизм синхронизации** между драйвером и приложением.

---

## 3. **RX Ring** — очередь приёма

### 🔹 Назначение
**Передать приложению указатели на принятые пакеты.**

> 💡 RX Ring — это **список адресов буферов, в которые драйвер скопировал пакеты**.

1. Драйвер берёт буфер из FQ.
2. Копирует пакет в этот буфер.
3. Кладёт **указатель на буфер** в RX Ring.
4. Приложение читает RX Ring, обрабатывает пакет, затем **возвращает буфер в FQ**.

---

### 🔍 Ключевые функции

```cpp
n_pkts = xsk_ring_cons__peek(&p->rxq, MAX_BURST, &pos);  // проверить
addr = xsk_ring_cons__rx_desc(&p->rxq, pos + i)->addr;   // получить адрес
len = xsk_ring_cons__rx_desc(&p->rxq, pos + i)->len;     // получить длину
xsk_ring_cons__release(&p->rxq, n_pkts);                 // освободить слоты
```

- RX Ring содержит **описатели (descriptors)**, а не просто адреса.
- Каждый описатель — это структура:
  ```cpp
  struct xdp_ring {
      __u64 addr;
      __u32 len;
      __u32 reserved;
  };
  ```
- `addr` — смещение в UMEM,
- `len` — длина пакета.

---

## 4. **TX Ring** — очередь отправки

### 🔹 Назначение
**Передать драйверу указатели на пакеты, которые нужно отправить.**

> 💡 TX Ring — это **список адресов буферов, содержащих пакеты для отправки**.

1. Приложение копирует пакет в буфер в UMEM.
2. Кладёт **указатель на буфер** в TX Ring.
3. Драйвер забирает пакет и отправляет его.
4. После отправки возвращает буфер в **CQ**.

### Ключевые функции

```cpp
status = xsk_ring_prod__reserve(&p->txq, n_pkts, &pos);  // зарезервировать
xsk_ring_prod__tx_desc(&p->txq, pos + i)->addr = addr;   // записать адрес
xsk_ring_prod__tx_desc(&p->txq, pos + i)->len = len;     // записать длину
xsk_ring_prod__submit(&p->txq, n_pkts);                  // подтвердить
```

---

### ⚠️ Когда нужно "пробуждать" драйвер?

Если используется флаг `XDP_USE_NEED_WAKEUP`, то после заполнения TX Ring нужно вызвать:

```cpp
sendto(xsk_socket__fd(p->xsk), nullptr, 0, MSG_DONTWAIT, nullptr, 0);
```

> Это **пробуждает драйвер**, если он "спит", ожидая новых пакетов.

| Очередь | Направление | Кто заполняет | Кто читает | Назначение |
|--------|-------------|----------------|-------------|-----------|
| **Fill (FQ)** | Приложение → Драйвер | Приложение | Драйвер | Дать драйверу свободные буферы |
| **Completion (CQ)** | Драйвер → Приложение | Драйвер | Приложение | Сообщить, что буферы отправлены |
| **RX Ring** | Драйвер → Приложение | Драйвер | Приложение | Передать принятые пакеты |
| **TX Ring** | Приложение → Драйвер | Приложение | Драйвер | Передать пакеты на отправку |
## Как связаны все очереди?

```
+----------------+     +----------------+
|                |     |                |
|   Fill Queue   |<----| Completion Q   |
|    (FQ)        |     |    (CQ)        |
|                |     |                |
+-------+--------+     +--------+-------+
        |                        ^
        v                        |
+-------+--------+     +--------+-------+
|                |     |                |
|    RX Ring     |     |    TX Ring     |
|                |     |                |
+-------+--------+     +--------+-------+
        |                        ^
        |                        |
        +--------> UMEM <--------+
                   (буферы)
```

- **FQ и CQ** — управляют **жизненным циклом буферов**.
- **RX и TX Ring** — управляют **передачей пакетов**.


---

### Setting up a XSK

Первый шаг — создать сокет, к которому мы будем прикреплять UMEM и кольцевые буферы. Мы делаем это, вызывая системный вызов сокета
```c
fd = socket(AF_XDP, SOCK_RAW, 0);
```

Далее следует настроить и зарегистрировать UMEM. Необходимо учесть 2 параметра: первый — размер фрагментов, который определяет максимальный размер пакета, который мы можем поместить. Это должно быть значение между 2048 и размером страницы системы и степенью двойки. Поэтому для большинства систем выбор лежит между 2048 и 4096. Второй параметр — количество фрагментов, его можно изменить по мере необходимости, но хорошим значением по умолчанию, похоже, будет 4096. Если предположить, что размер фрагмента равен 4096, это означает, что мы должны выделить 16777216 байт (16 МБ) памяти
```c
static const int chunk_size = 4096;
static const int chunk_count = 4096;
static const int umem_len = chunk_size * chunk_count;
unsigned char[chunk_count][chunk_size]umem = malloc(umem_len);
```

Теперь, когда у нас есть UMEM, подключим его к сокету через системный вызов setsockopt:
```c
struct xdp_umem_reg {
    __u64 addr; /* Start of packet data area */
    __u64 len; /* Length of packet data area */
    __u32 chunk_size;
    __u32 headroom;
    __u32 flags;
};

struct xdp_umem_reg umem_reg = {
    .addr = (__u64)(void *)umem,
    .len = umem_len,
    .chunk_size = chunk_size,
    .headroom = 0, // see Options, variations, and exceptions
    .flags = 0, // see Options, variations, and exceptions
};
if (!setsockopt(fd, SOL_XDP, XDP_UMEM_REG, &umem_reg, sizeof(umem_reg)))
    // handle error
```

Далее идут наши кольцевые буферы. Они выделяются ядром, когда мы сообщаем ядру, насколько большим должен быть каждый кольцевой буфер с помощью системного вызова setsockopt. После выделения мы можем отобразить кольцевой буфер в память нашего процесса с помощью системного вызова mmap. Следующий процесс следует повторить для каждого кольцевого буфера (с различными параметрами, которые будут указаны): Мы должны определить желаемый размер кольцевого буфера, который должен быть степенью 2, например, 128, 256, 512, 1024 и т. д. Размеры кольцевых буферов можно настраивать, и они могут отличаться от кольцевого буфера к кольцевому буферу, для этого примера мы выберем 512.

Мы сообщаем ядру выбранный нами размер с помощью системного вызова setsockopt:
```c
static const int ring_size = 512;
if (!setsockopt(fd, SOL_XDP, {XDP_RX_RING,XDP_TX_RING,XDP_UMEM_FILL_RING,XDP_UMEM_COMPLETION_RING}, &ring_size, sizeof(ring_size)))
    // handle error
```


После того, как мы задали размеры для всех кольцевых буферов, мы можем запросить смещения mmap с помощью системного вызова getsockopt:
```c
struct xdp_ring_offset {
    __u64 producer;
    __u64 consumer;
    __u64 desc;
    __u64 flags;
};
struct xdp_mmap_offsets {
    struct xdp_ring_offset rx;
    struct xdp_ring_offset tx;
    struct xdp_ring_offset fr; /* Fill */
    struct xdp_ring_offset cr; /* Completion */
};
struct xdp_ring_offset offsets = {0};
if (!getsockopt(fd, SOL_XDP, XDP_MMAP_OFFSETS, &offsets, sizeof(offsets)))
    // handle error
```


Последний шаг — отображение кольцевых буферов в памяти процесса с помощью системного вызова mmap:
```c
struct xdp_desc {
    __u64 addr;
    __u32 len;
    __u32 options;
};

void *<rx/tx/fill/completion>_ring_mmap = mmap(
fd,       <XDP_PGOFF_RX_RING/XDP_PGOFF_TX_RING/XDP_UMEM_PGOFF_FILL_RING/XDP_UMEM_PGOFF_COMPLETION_RING>,
offsets.<rx/tx/fr/cr>.desc + ring_size * sizeof(struct xdp_desc),
PROT_READ|PROT_WRITE,
MAP_SHARED|MAP_POPULATE
);
if (!<rx/tx/fill/completion>_ring_mmap)
    // handle error

__u32 *<rx/tx/fill/completion>_ring_consumer = <rx/tx/fill/completion>_ring_mmap + offsets.<rx/tx/fr/cr>.consumer;
__u32 *<rx/tx/fill/completion>_ring_producer = <rx/tx/fill/completion>_ring_mmap + offsets.<rx/tx/fr/cr>.producer;
struct xdp_desc[ring_size] <rx/tx/fill/completion>_ring = <rx/tx/fill/completion>_ring_mmap + offsets.<rx/tx/fr/cr>.desc;
```


Мы настроили наш XSK и имеем доступ как к UMEM, так и ко всем 4 кольцевым буферам. Последний шаг — связать наш XSK с сетевым устройством и очередью.

```c
struct sockaddr_xdp {
    __u16 sxdp_family;
    __u16 sxdp_flags;
    __u32 sxdp_ifindex;
    __u32 sxdp_queue_id;
    __u32 sxdp_shared_umem_fd;
};

struct sockaddr_xdp sockaddr = {
    .sxdp_family = AF_XDP,
    .sxdp_flags = 0, // see Options, variations, and exceptions
    .sxdp_ifindex = some_ifindex, // The actual ifindex is dynamically determined, picking is up to the user.
    .sxdp_queue_id = 0,
    .sxdp_shared_umem_fd = fd, // see Options, variations, and exceptions
};

if(!bind(fd, &sockaddr, sizeof(struct sockaddr_xdp)))
    // handle error
```

XSK может быть привязан только к одной очереди на сетевой карте. Для сетевых карт с несколькими очередями процедура создания XSK должна быть повторена для каждой очереди. UMEM может быть опционально общим для экономии памяти, подробности см. в разделе «Параметры, вариации и исключения». На этом этапе мы готовы отправлять трафик, см. раздел «Прием и отправка пакетов». Но нам все еще нужно настроить нашу программу XDP и XSKMAP для обхода входящего трафика.




---
---


**AF_XDP (Address Family XDP)** – это специальный тип сокетов в Linux, предназначенный для высокопроизводительной обработки сетевых пакетов. Он позволяет пользовательским приложениям получать пакеты напрямую из XDP (eXpress Data Path) программ в ядре, минуя традиционный сетевой стек.

```c
#include <linux/if_xdp.h>
#include <net/if.h>
#include <sys/socket.h>

// Создание AF_XDP сокета
int create_af_xdp_socket(const char* ifname, int queue_id) {
    int sockfd = socket(AF_XDP, SOCK_RAW, 0);
    if (sockfd < 0) {
        perror("socket");
        return -1;
    }

    struct sockaddr_xdp sxdp = {};
    sxdp.sxdp_family = AF_XDP;
    sxdp.sxdp_ifindex = if_nametoindex(ifname); // индекс интерфейса
    sxdp.sxdp_queue_id = queue_id;              // очередь

    if (bind(sockfd, (struct sockaddr*)&sxdp, sizeof(sxdp)) {
        perror("bind");
        close(sockfd);
        return -1;
    }
    return sockfd;
}
```


Для приёма пакетов через **AF_XDP** стандартные системные вызовы `read()`/`recv()` **не используются**, так как AF_XDP работает с кольцевыми буферами (ring buffers) в обход традиционного сетевого стека.  

Вместо этого применяется механизм **memory-mapped (mmap) буферов** и **кольцевых очередей** (rings) в пользовательском пространстве, что обеспечивает минимальные накладные расходы.  

---

## **Как получать пакеты через AF_XDP?**
Для работы с AF_XDP нужно:
1. **Настроить UMEM** (буферная область для пакетов).
2. **Создать и настроить очереди (rings)**:
   - **RX-queue** (приём пакетов).
   - **TX-queue** (отправка пакетов, если нужно).
3. **Опрашивать очередь RX** для получения пакетов.

---

### **Настройка UMEM (общего буфера)**
UMEM – это замапленная в пользовательское пространство память, куда ядро будет складывать пакеты.

```c
#include <linux/if_xdp.h>
#include <sys/mman.h>
#include <unistd.h>

#define NUM_FRAMES  4096  // Количество фреймов в UMEM
#define FRAME_SIZE  2048  // Размер одного фрейма (достаточно для MTU 1500)

struct xdp_umem_reg {
    __u64 addr;          // Адрес UMEM (должен быть выровнен по странице)
    __u64 len;           // Размер UMEM (NUM_FRAMES * FRAME_SIZE)
    __u32 chunk_size;    // Размер фрейма (FRAME_SIZE)
    __u32 headroom;      // Доп. место перед пакетом (обычно 0)
};

int setup_umem(void **buffer, int sockfd) {
    size_t umem_size = NUM_FRAMES * FRAME_SIZE;
    
    // Выделяем выровненную память под UMEM
    *buffer = mmap(NULL, umem_size, PROT_READ | PROT_WRITE,
                   MAP_PRIVATE | MAP_ANONYMOUS | MAP_POPULATE, -1, 0);
    if (*buffer == MAP_FAILED) {
        perror("mmap");
        return -1;
    }

    struct xdp_umem_reg umem = {};
    umem.addr = (__u64)(*buffer);
    umem.len = umem_size;
    umem.chunk_size = FRAME_SIZE;
    umem.headroom = 0;
    if (setsockopt(sockfd, SOL_XDP, XDP_UMEM_REG, &umem, sizeof(umem))) {
        perror("setsockopt(XDP_UMEM_REG)");
        munmap(*buffer, umem_size);
        return -1;
    }

    return 0;
}
```

---

### **2. Настройка RX-очереди (приём пакетов)**
```c
int setup_rx_queue(int sockfd) {
    __u32 rx_ring_size = 1024;  // Размер RX-кольца
    if (setsockopt(sockfd, SOL_XDP, XDP_RX_RING, &rx_ring_size, sizeof(rx_ring_size))) {
        perror("setsockopt(XDP_RX_RING)");
        return -1;
    }

    // Маппим RX-кольцо в пользовательское пространство
    struct xdp_desc *rx_ring;
    size_t rx_ring_sz = rx_ring_size * sizeof(struct xdp_desc);
    rx_ring = mmap(NULL, rx_ring_sz, PROT_READ | PROT_WRITE,
                   MAP_SHARED | MAP_POPULATE, sockfd, XDP_PGOFF_RX_RING);
    if (rx_ring == MAP_FAILED) {
        perror("mmap(RX_RING)");
        return -1;
    }

    return 0;
}
```

---

### **3. Получение пакетов из RX-очереди**
Пакеты извлекаются из RX-кольца с помощью:
- **`recvfrom()` с флагом `MSG_DONTWAIT`** (неблокирующий режим).
- Или через **`poll()`/`epoll()`** для эффективного ожидания.

```c
#include <poll.h>

void receive_packets(int sockfd, void *buffer) {
    struct pollfd fds[1] = { { .fd = sockfd, .events = POLLIN } };
    struct xdp_desc desc;
    char packet[2048];

    while (1) {
        // Ждём пакеты (можно использовать epoll для большей эффективности)
        int ret = poll(fds, 1, 1000);  // Таймаут 1 сек
        if (ret <= 0) continue;

        // Получаем дескриптор пакета
        ret = recvfrom(sockfd, &desc, sizeof(desc), MSG_DONTWAIT, NULL, NULL);
        if (ret < 0) {
            if (errno == EAGAIN) continue;
            perror("recvfrom");
            break;
        }

        // Копируем пакет из UMEM
        memcpy(packet, (char *)buffer + desc.addr, desc.len);
        printf("Got packet, len=%u\n", desc.len);
    }
}
```

---

### **Полный пример**
```c
int main() {
    const char *ifname = "eth0";
    int queue_id = 0;
    void *umem_buffer;

    int sockfd = create_af_xdp_socket(ifname, queue_id);
    if (sockfd < 0) return 1;

    if (setup_umem(&umem_buffer, sockfd) < 0) 
	    return 1;
	    
    if (setup_rx_queue(sockfd) < 0) 
	    return 1;

    receive_packets(sockfd, umem_buffer);

    close(sockfd);
    munmap(umem_buffer, NUM_FRAMES * FRAME_SIZE);
    return 0;
}
```

---

### **Очереди (Rings) в AF_XDP: RX и TX**

В **AF_XDP** используется модель кольцевых буферов (ring buffers) для эффективной передачи пакетов между ядром и пользовательским пространством. Основные очереди:

1. **RX-queue (Receive Queue)** – очередь приёма пакетов (из ядра в пользовательское пространство).  
2. **TX-queue (Transmit Queue)** – очередь отправки пакетов (из пользовательского пространства в ядро).  

Обе очереди работают по принципу **Producer-Consumer**:
- **Producer** (поставщик) заполняет очередь.
- **Consumer** (потребитель) забирает данные из очереди.

### **Являются ли TX-queue и RX-queue общими для всех AF_XDP-сокетов или уникальными для каждого?**  

**Короткий ответ:**  
**Каждый AF_XDP-сокет имеет свои собственные (уникальные) TX-queue и RX-queue.**  
Но есть нюансы, связанные с разделением UMEM и привязкой к сетевому интерфейсу.  

---

## **1. Общая архитектура AF_XDP**
AF_XDP работает по модели **"один сокет — одна пара очередей (TX/RX)"**, но:
- **UMEM (буферная область)** может быть **общей** для нескольких сокетов (если явно указано).
- **Очереди (rings) всегда принадлежат конкретному сокету** и не разделяются между разными сокетами.

### **Примеры конфигураций**
#### **a) Один сокет — одна пара очередей (базовый случай)**
```text
Сокет 1:
   UMEM → [RX-queue] ← XDP-программа
          [TX-queue] → Сеть
```
- Все пакеты идут в RX-queue этого сокета.
- TX-queue этого сокета отправляет только его пакеты.

#### **b) Несколько сокетов с общим UMEM (но разными очередями)**
```text
UMEM (общий буфер)
├─ Сокет 1:
│     [RX-queue 1] ← XDP-программа
│     [TX-queue 1] → Сеть
└─ Сокет 2:
      [RX-queue 2] ← XDP-программа
      [TX-queue 2] → Сеть
```
- UMEM общий, но **очереди у каждого сокета свои**.
- XDP-программа может распределять пакеты между разными RX-queue (например, по хэшу).

#### **c) Привязка нескольких сокетов к одной сети. очереди (не рекомендуется)**
```text
Сетевой интерфейс eth0, очередь 0:
├─ Сокет 1: [RX-queue 1], [TX-queue 1]
└─ Сокет 2: [RX-queue 2], [TX-queue 2]  ← Конфликт! (обычно так нельзя)
```
- По умолчанию **нельзя** привязать два AF_XDP-сокета к **одной и той же сетевой очереди** (если только не используется **XDP_SHARED_UMEM**).

---

## **2. Детали работы очередей**
### **RX-queue (уникальная для каждого сокета)**
- Когда XDP-программа решает передать пакет в AF_XDP, она указывает **конкретный RX-queue** (через `bpf_redirect_map` или `xdp_sock`).
- Если два сокета используют **один UMEM**, но разные RX-queue, то:
  - Пакеты могут распределяться между ними (например, по хэшу или правилам XDP).
  - Каждый сокет видит только свои пакеты.

### **TX-queue (уникальная для каждого сокета)**
- Пакеты, отправленные через `sendto()`, попадают **только в TX-queue этого сокета**.
- Даже если UMEM общий, TX-queue одного сокета не влияет на другой.

---

## **3. Как разделить UMEM между несколькими сокетами?**
Если нужно, чтобы несколько AF_XDP-сокетов использовали **один и тот же UMEM** (например, для экономии памяти), нужно:
1. Создать UMEM и зарегистрировать его в первом сокете:
   ```c
   setsockopt(sock1, SOL_XDP, XDP_UMEM_REG, &umem_cfg, sizeof(umem_cfg));
   ```
2. Передать тот же UMEM в другие сокеты с флагом `XDP_SHARED_UMEM`:
   ```c
   setsockopt(sock2, SOL_XDP, XDP_UMEM_REG, &umem_cfg, sizeof(umem_cfg));
   setsockopt(sock2, SOL_XDP, XDP_SHARED_UMEM, &enable, sizeof(enable));
   ```

### **Пример разделяемого UMEM**
```c
// Сокет 1
int sock1 = socket(AF_XDP, ...);
setsockopt(sock1, SOL_XDP, XDP_UMEM_REG, &umem_cfg, ...);

// Сокет 2 (использует тот же UMEM)
int sock2 = socket(AF_XDP, ...);
setsockopt(sock2, SOL_XDP, XDP_UMEM_REG, &umem_cfg, ...);
int enable = 1;
setsockopt(sock2, SOL_XDP, XDP_SHARED_UMEM, &enable, sizeof(enable));
```
Теперь:
- Оба сокета используют **один UMEM**, но **разные RX/TX-очереди**.
- XDP-программа может направлять пакеты в `sock1` или `sock2` через `bpf_redirect_map()`.

---

## **4. Когда очереди могут конфликтовать?**
- Если два AF_XDP-сокета привязаны к **одному сетевому интерфейсу и одной очереди (queue_id)** без `XDP_SHARED_UMEM`, второй `bind()` вернёт ошибку.
- Решение:
  - Использовать `XDP_SHARED_UMEM`.
  - Или привязывать сокеты к разным `queue_id` (если NIC поддерживает несколько очередей).

---

## **RX-queue (Receive Queue)**
### **Назначение**
- Получает пакеты, которые были переданы из XDP-программы (`xdp_sifter`) в AF_XDP-сокет.
- Работает в **zero-copy** режиме (пакеты не копируются, а просто передаются указатели на UMEM).

### **Как устроена?**
- Это кольцевой буфер (ring buffer) из дескрипторов `struct xdp_desc`:
  ```c
  struct xdp_desc {
      __u64 addr;  // Адрес в UMEM, где лежит пакет
      __u32 len;   // Длина пакета
      __u32 options;
  };
  ```
- **Producer** (ядро Linux) записывает сюда дескрипторы принятых пакетов.
- **Consumer** (пользовательская программа) читает эти дескрипторы и обрабатывает пакеты.

### **Как работает приём пакетов?**
1. Ядро (XDP-программа) решает, что пакет нужно передать в AF_XDP.
2. Оно помещает дескриптор пакета в **RX-queue** (указывает на адрес в UMEM).
3. Пользовательская программа видит новый дескриптор в RX-queue и обрабатывает пакет.

### **Пример кода (опроса RX-queue)**
```cpp
// Чтение пакетов из RX-queue
void process_rx_queue(int sockfd, void *umem_buffer) {
    struct xdp_desc desc;
    char packet[2048];

    while (1) {
        // Получаем дескриптор пакета (неблокирующий режим)
        int ret = recvfrom(sockfd, &desc, sizeof(desc), MSG_DONTWAIT, NULL, NULL);
        if (ret < 0) {
            if (errno == EAGAIN) 
	            break;  // Нет новых пакетов
            perror("recvfrom");
            exit(1);
        }

        // Копируем пакет из UMEM
        memcpy(packet, (char *)umem_buffer + desc.addr, desc.len);
        printf("Received packet (len=%u)\n", desc.len);

        // Возвращаем фрейм в UMEM для повторного использования
        // (если используется режим с заполнением FILL-queue)
    }
}
```

---

## **TX-queue (Transmit Queue)**
### **Назначение**
- Отправляет пакеты из пользовательского пространства обратно в сетевой стек или наружу (если нужно).
- Например, если пользовательская программа модифицирует пакеты и хочет их переслать.

### **Как устроена?**
- Аналогично RX-queue: кольцевой буфер из `struct xdp_desc`.
- **Producer** (пользовательская программа) кладёт сюда дескрипторы пакетов для отправки.
- **Consumer** (ядро) забирает их и отправляет в сеть.

### **Как работает отправка пакетов?**
1. Пользовательская программа записывает пакет в UMEM.
2. Помещает дескриптор (`addr` и `len`) в **TX-queue**.
3. Ядро забирает дескриптор и отправляет пакет.

### **Пример кода (отправки пакета)**
```cpp
// Отправка пакета через TX-queue
void send_packet(int sockfd, void *umem_buffer, const char *data, size_t len) {
    // Записываем данные в UMEM (в реальности нужно брать свободный фрейм из FILL-queue)
    static __u64 current_addr = 0;
    memcpy((char *)umem_buffer + current_addr, data, len);

    // Формируем дескриптор для TX
    struct xdp_desc desc;
    desc.addr = current_addr;
    desc.len = len;

    // Отправляем дескриптор в TX-queue
    if (sendto(sockfd, &desc, sizeof(desc), 0, NULL, NULL) < 0) {
        perror("sendto");
        return;
    }

    current_addr += len;  // В реальности нужно управлять фреймами через FILL-queue!
}
```

---

## **Дополнительные очереди (FILL и COMPLETION)**
Для эффективного управления памятью в AF_XDP используются ещё две очереди:
1. **FILL-queue**  
   - Пользовательская программа кладёт сюда **свободные фреймы UMEM**, чтобы ядро могло использовать их для новых пакетов.
   - Без этого RX-queue не сможет получать пакеты (некуда будет класть данные).

2. **COMPLETION-queue**  
   - Ядро возвращает сюда фреймы, которые были отправлены через TX-queue и больше не нужны.
   - Пользовательская программа может переиспользовать их.

### **Пример настройки FILL-queue**
```cpp
int setup_fill_queue(int sockfd) {
    __u32 fill_ring_size = 1024;
    if (setsockopt(sockfd, SOL_XDP, XDP_UMEM_FILL_RING, &fill_ring_size, sizeof(fill_ring_size))) {
        perror("setsockopt(XDP_UMEM_FILL_RING)");
        return -1;
    }

    // Маппим FILL-queue в пользовательское пространство
    struct xdp_desc *fill_ring;
    size_t fill_ring_sz = fill_ring_size * sizeof(struct xdp_desc);
    fill_ring = mmap(NULL, fill_ring_sz, PROT_READ | PROT_WRITE,
                     MAP_SHARED | MAP_POPULATE, sockfd, XDP_PGOFF_UMEM_FILL_RING);
    if (fill_ring == MAP_FAILED) {
        perror("mmap(FILL_RING)");
        return -1;
    }

    // Заполняем FILL-queue свободными фреймами UMEM
    for (int i = 0; i < fill_ring_size; i++) {
        fill_ring[i].addr = i * FRAME_SIZE;  // Указываем адрес фрейма в UMEM
        fill_ring[i].len = FRAME_SIZE;
    }

    // Сообщаем ядру, сколько фреймов доступно
    __u32 num_filled = fill_ring_size;
    if (setsockopt(sockfd, SOL_XDP, XDP_UMEM_FILL_RING_SIZE, &num_filled, sizeof(num_filled))) {
        perror("setsockopt(XDP_UMEM_FILL_RING_SIZE)");
        return -1;
    }

    return 0;
}
```

---

## **Схема работы всех очередей**
```
Пользовательское пространство       Ядро Linux
-----------------------------      -------------------
1. FILL-queue   → (Свободные фреймы) → Ядро берёт фреймы для RX
2. RX-queue     ← (Принятые пакеты)  ← Ядро кладёт пакеты
3. TX-queue     → (Пакеты на отправку) → Ядро отправляет их
4. COMPLETION-queue ← (Освободившиеся фреймы) ← Ядро возвращает фреймы
```










---
---

Несколько сокетов могут быть привязаны (через системные вызовы bind()) к общему адресу (смотри опцию сокета SO_REUSEPORT). Для TCP-сокетов это позволит распределить нагрузку по обработке клиентов на несколько потоков. Для UDP-сокетов это позволит распределить нагрузку по приему дэйтаграмм на несколько потоков.


Сокет может быть ассоциирован с одним сетевым интерфейсом - см. опцию сокета SO_BINDTODEVICE(например, "eth0"). Для raw-сокета привязка к интерфейсу выполняется с помощью системного вызова bind(3).

Сокет может быть ассоциирован с одним ядром CPU - см. опцию сокета SO_INCOMING_CPU.

Сокет может быть ассоциирован с физической очередью NIC - см. опцию сокета SO_INCOMING_NAPI_ID.

Для raw-сокета рекомендуется использовать общий(для ядра ОС и пользовательского приложения) кольцевой буфер с помощью опций PACKET_RX_RING и PACKET_TX_RING.

С версии 3.19 ядра Linux eBPF-фильтры могут быть прикреплены к сокетам, и с версии 4.1 - к классификатору трафика tc.

В cBPF загрузка и подсоединение фильтра происходили как атомарная операция. А в eBPF загрузка программы и привязка ее к генератору событий разделены по времени.

