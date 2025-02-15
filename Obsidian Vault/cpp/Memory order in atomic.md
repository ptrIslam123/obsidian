# Роль различных `std::memory_order` в C++

```c++
enum memory_order  
{  
    memory_order_relaxed,  
    memory_order_consume,  
    memory_order_acquire,  
    memory_order_release,  
    memory_order_acq_rel,  
    memory_order_seq_cst  // default for all atomic operations 

};
```

**Язык C++ предоставляет три способа синхронизации памяти. По мере возрастания строгости: `relaxed`, `release/acquire` и `sequential consistency`**.

# std::memory_order_relaxed — минимальные гарантии

* **Неделимый, но расслабленный**
Самый простой для понимания флаг синхронизации памяти — `relaxed`. Он гарантирует только свойство атомарности операций, при этом не может участвовать в процессе синхронизации данных между потоками. 

* Свойства:
	- модификация переменной "появится" в другом потоке не сразу
	    
	- поток `thread2` "увидит" значения **одной и той же** переменной в том же порядке, в котором происходили её  модификации в потоке `thread1`
	    
	- порядок модификаций разных переменных в потоке `thread1` не сохранится в потоке `thread2`

### Пример правильного использования

```c++
std::atomic<bool> isReady{false};

void thread1() {
    while (isReady.load(std::memory_order_relaxed) == true) {  // (1)
	    // ... do something with non shared data
    }
}

void thread2() {
    isReady.store(true, std::memory_order_relaxed);  // (2)
}
```

В данном примере не важен порядок в котором `thread1` увидит изменения из потока, вызывающего `thread2`. Также не важно то, чтобы `thread1` мгновенно (синхронно) увидел выставление флага `isReady` в `true`.

#### Пример кода который должен работать но не работает как ожидается из-за эффектов memory_order_relaxed

```c++
std::atomic<bool> isReady{false};
std::string data;

void thread1() {
    while (isReady.load(std::memory_order_relaxed) == true) {  // (1)
        // Может не выполниться, даже если flag == true!
        assert(!data.empty()); // (2) Потенциальная гонка данных
    }
}

void thread2() {
    data = "some data";                          // (3)
    isReady.store(true, std::memory_order_relaxed);  // (4)
}
```

Тут нет гарантий, что поток `thread1` увидит изменения `data` ранее, чем изменение флага `isReady`, т.к. синхронизацию памяти флаг `relaxed` не обеспечивает.


# std::memory_order_seq_cst — самое строгое

- **порядок модификаций разных атомарных переменных в потоке `thread1` сохранится в потоке `thread2`**
    
- все потоки будут видеть один и тот же порядок модификации всех атомарных переменных. Сами модификации могут происходить в разных потоках
    
- все модификации памяти (не только модификации над атомиками) в потоке `thread1`, выполняющей `store` на атомарной переменной, будут видны после выполнения `load` этой же переменной в потоке `thread2`


Таким образом можно представить `seq_cst` операции, как барьеры памяти, в которых состояние памяти синхронизируется между всеми потоками программы.

Этот флаг синхронизации памяти в C++ используется по умолчанию, т.к. с ним меньше всего проблем с точки зрения корректности выполнения программы. Но `seq_cst` является дорогой операцией для процессоров, в которых вычислительные ядра слабо связаны между собой в плане механизмов обеспечения консистентности памяти. Например, для x86-64 `seq_cst` дешевле, чем для ARM архитектур.

### Пример

```c++
std::atomic<bool> x, y;
std::atomic<int> z;
 
void thread_write_x() {
	x.store(true, std::memory_order_seq_cst);
}
 
void thread_write_y() {
	y.store(true, std::memory_order_seq_cst);
}
 
void thread_read_x_then_y() {
	while (!x.load(std::memory_order_seq_cst));
	if (y.load(std::memory_order_seq_cst)) {
		++z;
	}
}
 
void thread_read_y_then_x() {
	while (!y.load(std::memory_order_seq_cst));
	if (x.load(std::memory_order_seq_cst)) {
		++z;
	}
}
```

После того, как все четыре потока отработают, значение переменной `z` будет равно `1` или `2`, потому что потоки `thread_read_x_then_y` и `thread_read_y_then_x` "увидят" изменения `x` и `y` в одном и том же порядке. От запуска к запуску это могут быть: сначала `x = true`, потом `y = true`, или сначала `y = true`, потом `x = true`.

Модификатор `seq_cst` всегда может быть использован вместо `relaxed` и `acquire/release`, еще и поэтому он является модификатором по умолчанию.

# Синхронизация пары. Acquire/Release 
**Флаг синхронизации памяти `acquire/release` является более тонким способом синхронизировать данные между парой потоков. Два ключевых слова: `memory_order_acquire` и `memory_order_release` работают только в паре над одним атомарным объектом**.

