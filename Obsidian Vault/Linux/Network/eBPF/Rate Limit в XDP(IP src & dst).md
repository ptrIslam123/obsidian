В контексте программ XDP/eBPF-системы фильтрации **Rate Limit** — это механизм ограничения скорости трафика для защиты от перегрузки сети или DDoS-атак.  
Различают два основных типа:

---

## **1. Rate Limit по IP-источнику (`IP src`)**

**Что ограничивает**:  
Количество пакетов/байт, которые **один IP-адрес отправителя** может передать за определенное время.  

**Зачем нужно**:  
- Защита от DDoS-атак (например, ботнет шлет тысячи запросов с одного IP).  
- Предотвращение злоупотреблений (сканирование портов, brute-force).  

**Как работает в eBPF**:  
- В eBPF-карте (`BPF_MAP_TYPE_LRU_HASH`) хранится счетчик пакетов/байт для каждого `src_ip`.  
- При превышении лимита пакеты от этого IP дропаются (`XDP_DROP`).  

**Пример псевдокода**:  
```c
struct {
    __uint(type, BPF_MAP_TYPE_LRU_HASH);
    __type(key, __be32);  // src_ip
    __type(value, struct counter);  // { packets, bytes }
    __uint(max_entries, 100000);
} src_rate_limit SEC(".maps");

SEC("xdp")
int xdp_rate_limit(struct xdp_md *ctx) {
    __be32 src_ip = get_src_ip(ctx);
    struct counter *cnt = bpf_map_lookup_elem(&src_rate_limit, &src_ip);
    
    if (!cnt) {
        struct counter new_cnt = {1, ctx->data_end - ctx->data};
        bpf_map_update_elem(&src_rate_limit, &src_ip, &new_cnt, BPF_NOEXIST);
    } else {
        cnt->packets++;
        cnt->bytes += (ctx->data_end - ctx->data);
        if (cnt->packets > MAX_PACKETS_PER_SEC) 
            return XDP_DROP;
    }
    return XDP_PASS;
}
```

---

## **2. Rate Limit по IP-назначению (`IP dst`)**

**Что ограничивает**:  
Количество пакетов/байт, которые могут быть отправлены **на один IP-адрес получателя** (например, ваш сервер).  

**Зачем нужно**:  
- Защита сервера от перегрузки (например, если клиенты слишком часто стучатся на порт 80).  
- Балансировка нагрузки (равномерное распределение трафика между серверами).  

**Как работает в eBPF**:  
Аналогично `IP src`, но ключом в карте является `dst_ip`:  
```c
struct {
    __uint(type, BPF_MAP_TYPE_LRU_HASH);
    __type(key, __be32);  // dst_ip
    __type(value, struct counter);
    __uint(max_entries, 1000);
} dst_rate_limit SEC(".maps");

SEC("xdp")
int xdp_rate_limit(struct xdp_md *ctx) {
    __be32 dst_ip = get_dst_ip(ctx);
    struct counter *cnt = bpf_map_lookup_elem(&dst_rate_limit, &dst_ip);
    // ... логика как в примере выше ...
}
```

---

## **🔹 Разница между `IP src` и `IP dst` Rate Limit**
| Характеристика          | Rate Limit по `src_ip`                     | Rate Limit по `dst_ip`                     |
|-------------------------|-------------------------------------------|--------------------------------------------|
| **Цель**               | Защита от атак **отправителей**           | Защита **серверов** от перегрузки          |
| **Тип угроз**          | DDoS, сканирование портов                 | Флуд одного сервера (например, HTTP-запросы) |
| **Ключ в eBPF-карте**  | IP-адрес источника                       | IP-адрес назначения                        |
| **Пример применения**  | Блокировка ботнета                       | Ограничение запросов к API                 |

---

## **🔹 Как вычисляются лимиты?**
1. **Пакеты/секунду**:  
   ```c
   if (cnt->packets > 1000) return XDP_DROP;  // Не более 1000 пакетов/сек
   ```
2. **Байты/секунду**:  
   ```c
   if (cnt->bytes > 10 * 1024 * 1024) return XDP_DROP;  // Не более 10 МБ/сек
   ```

---

## **🔹 Где применяется?**
- **Cloudflare**: Использует `src_ip` rate limiting для защиты от DDoS.  
- **Kubernetes Ingress**: Ограничивает запросы к сервисам (`dst_ip`).  
- **Игровые сервера**: Блокируют игроков, которые шлют слишком много пакетов.  

---

## **🔹 Технические нюансы**
- **LRU-карты**: Для автоматического вытеснения старых IP.  
- **Атомарные счетчики**:  
  ```c
  __sync_fetch_and_add(&cnt->packets, 1);  // Атомарное увеличение
  ```
- **Сброс счетчиков**: Через пользовательское приложение (например, каждую секунду).  
