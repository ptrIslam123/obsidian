**BCC** — это мощная и удобная платформа, которая позволяет **писать eBPF-программы на Python или C++**, а также **загружать, управлять и присоединять их к событиям ядра прямо из пользовательского пространства**.

**BCC** — это инструментарий, предоставляющий:

- Фреймворк для написания eBPF-программ
- Интеграцию с Python и Lua
- Возможность прикреплять программы к:
  - `kprobe` / `uprobe`
  - `tracepoint`
  - `perf_event`
  - Сетевые события (`XDP`, `TC`)
  - LSM хуки
  - И другим источникам событий

BCC сам заботится о компиляции, загрузке и взаимодействии с картами. Это делает его идеальным для **быстрого прототипирования, отладки и анализа системных событий**.

---

##  Пример: Захват вызовов `execve()` через Python + BCC

```python
from bcc import BPF

# eBPF программа на C, встроенная в Python
program = """
int handle_execve(struct pt_regs *ctx) {
    char comm[16];
    bpf_get_current_comm(&comm, sizeof(comm));
    bpf_trace_printk("New process: %s\\n", comm);
    return 0;
}
"""

# Загружаем программу
bpf = BPF(text=program)

# Привязываем к событию execve (sys_enter_execve)
bpf.attach_kprobe(event="sys_enter_execve", fn_name="handle_execve")

# Читаем вывод trace_printk
print("Tracing execve()... Ctrl+C to end.")
try:
    while True:
        try:
            (task, pid, cpu, flags, ts, msg) = bpf.trace_fields()
            print(f"{msg}")
        except KeyboardInterrupt:
            break
except KeyboardInterrupt:
    pass
```

> 💡 Когда ты запускаешь этот скрипт, он:
> - Компилирует eBPF-программу
> - Прикрепляет её к событию `sys_enter_execve`
> - Выводит имя каждого нового процесса

---

## Как выгрузить программу?

Как только ты завершишь выполнение скрипта (например, нажав `Ctrl+C`), BCC **автоматически выгрузит программу** из ядра.

Если хочешь сделать это явно:

```python
bpf.remove_kprobe(event="sys_enter_execve")
```

---

## Другие типы событий, к которым можно прикрепляться:

| Тип | Метод в BCC | Описание |
|-----|-------------|----------|
| `kprobe` | `attach_kprobe()` | Хук на функцию ядра |
| `uprobe` | `attach_uprobe()` | Хук на функцию в пользовательском пространстве |
| `tracepoint` | `enable_tracepoint()` | Подписка на tracepoint из ядра |
| `xdp` | `XDP` класс | Прикрепление XDP-программы к интерфейсу |
| `perf_event` | `open_perf_buffer()` | Чтение данных по таймеру или аппаратному событию |

---

## Установка BCC

### На Ubuntu/Debian:

```bash
sudo apt install bpfcc-tools linux-headers-$(uname -r)
```

Или установка библиотеки для разработки:

```bash
sudo apt install libbpfcc-dev
```

### Проверка установки:

```bash
python3 -c "import bcc; print(bcc.__version__)"
```
