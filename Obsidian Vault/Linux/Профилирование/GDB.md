## Основные концепции GDB

| Понятие | Объяснение |
|--------|------------|
| `breakpoint` | Точка останова, где выполнение программы приостанавливается |
| `watchpoint` | Наблюдение за изменением переменной или памяти |
| `stack frame` | Контекст вызова функции (локальные переменные, аргументы и т.д.) |
| `thread` | Поток исполнения, особенно важно при многопоточности |
| `core dump` | Снимок памяти программы в момент краха |
| `disassembly` | Дизассемблированный код (ассемблер) |
| `registers` | Регистры процессора (RAX, RSP, RIP и др.) |

## Базовые команды GDB

### 1. **Запуск программы**

```bash
gdb ./my_program
```

или сразу запустить:

```bash
gdb -ex run --args ./my_program arg1 arg2
```

> ⚠️ Компилируй с флагом `-g`, чтобы получить отладочные символы:
```bash
g++ -g my_program.cpp -o my_program
```

---

### 2. **Брейкпоинты (`break`)**
* Установка
```gdb
break main
break MyClass::myMethod
break filename.cpp:42
break "MyTemplateClass<int>::doSomething()"
rbreak "MyTemplateClass.*::insert_data()"
```

Иногда перем установкой брейкпоинта нужно найти символ:
```bash
info functions MyTemplateClass.*method
```

- `break` или `b` — поставить точку останова
- Используй `Tab` для автодополнения имён

* Просмотр списка брейкпоинтов
```bash
info breakpoints ## или коротко: i b
```
 
* Удаление
```bash
delete 1 ### Удалить конкретный брейкпоинт по номеру
delete 1 2 ### Можно указать несколько номеров
delete breakpoints ### Удалить все брейкпоинты
```
---

### 3. **Управление выполнением**
```gdb
run [args]        # Запуск программы
continue          # Продолжить выполнение до следующего break
step              # Выполнить одну строку, входя в функции
next              # Выполнить строку, не заходя в функции
finish            # Выйти из текущей функции
kill              # Остановить программу
quit              # Выйти из GDB
```

---

### 4. **Инспекция состояния программы**
```gdb
print variable    # Вывести значение переменной
print/x ptr       # Вывести в шестнадцатеричном виде
info registers    # Посмотреть регистры CPU
backtrace         # Посмотреть стек вызовов (call stack)
bt                # То же самое, короткая форма
frame N           # Переключиться на фрейм N из bt
list              # Показать исходный код вокруг текущей строки
```

---

### 5. **Условные брейкпоинты**
```gdb
break file.cpp:42 if x > 10
condition 1 x == 5   # Изменить условие для breakpoint 1
```

---

### 6. **Watchpoints (наблюдение за данными)**
```gdb
watch x            # Стоп, когда изменится x
rwatch x           # Стоп, когда x будет прочитан
awatch x           # Стоп, когда x читается или пишется
```

---

### 7. **Работа с потоками**
```gdb
info threads      # Посмотреть все потоки
thread N          # Переключиться на поток N
thread apply all bt  # Посмотреть стек всех потоков
```

---

### 8. **Дизассемблер и инструкции**
```gdb
disassemble       # Дизассемблировать текущую функцию
disassemble main
x/10i $pc        # Вывести 10 инструкций от текущего RIP
stepi             # Шагнуть на одну инструкцию
nexti             # То же, но без входа в call
```

---

### 9. **Логирование и сохранение вывода**
```gdb
set logging on    # Логировать всё в gdb.txt
set logging off
```

---

### 10. **Скрипты и автоматизация**
Создай файл `.gdbinit` или свой скрипт:

```gdb
define mybreaks
    break main
    break MyClass::myFunc
end
```

Выполни его:

```bash
gdb -x myscript.gdb ./myprogram
```

---

## Полезные советы

- **Используй `Tab`** — GDB поддерживает автодополнение.
- **`Ctrl+C`** — прерывает выполнение программы.
- **`Ctrl+L`** — перерисовывает экран (если всё перемешалось).
- **`help`** — покажет доступные команды.
- **`help breakpoints`** — справка по брейкпоинтам.
- **`set pagination off`** — отключает паузу вывода.
- **`layout asm` / `layout regs`** — удобный режим просмотра кода и регистров.

---

## 🛠 Пример сессии

```bash
$ g++ -g test.cpp -o test
$ gdb ./test
(gdb) break main
Breakpoint 1 at 0x4005f6: file test.cpp, line 5.
(gdb) run
Starting program: /home/user/test

Breakpoint 1, main () at test.cpp:5
5           int a = 10;
(gdb) next
6           int b = a * 2;
(gdb) print a
$1 = 10
(gdb) step
7           std::cout << b << std::endl;
(gdb) print b
$2 = 20
(gdb) continue
Continuing.
20
[Inferior 1 (process 12345) exited normally]
(gdb) quit
```

---

## Ресурсы для глубокого изучения

1. **Официальная документация**:  
   https://sourceware.org/gdb/current/onlinedocs/gdb/

2. **Книга "Debugging with GDB"**  
   https://ftp.gnu.org/old-gnu/Manuals/gdb/html_node/gdb_toc.html

3. **Cheat Sheet**  
   https://darkdust.net/files/GDB%20Cheat%20Sheet.pdf

4. **TUI Mode (графический интерфейс в терминале)**  
   ```bash
   layout asm
   layout regs
   ```

5. **GDB Dashboard (красивый UI поверх GDB)**  
   https://github.com/cyrus-and/gdb-dashboard
