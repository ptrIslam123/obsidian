## Двоичная(Бинарная) куча(d-куча)

**Двоичная куча (binary heap) – просто реализуемая структура данных, позволяющая быстро (за логарифмическое время) добавлять элементы и извлекать элемент с максимальным приоритетом (например, максимальный по значению). Это полное бинарное дерево, в котором каждый узел больше (или меньше) любого из его потомков.**

- Если узел больше любого из потомков, это max-куча. Если меньше, это min-куча;
- Двоичную кучу удобно хранить в виде массива, заполняя его уровнями слева направо;

## Основные свойства

- **Корень кучи всегда содержит максимальный (min-куча: минимальный) элемент**

- **Высота кучи, хранящейся в массиве размера n, равна log⁡2n(Сбалансированное дерево, что бы все операции над кучей выпонялись за логорифм)**

Удобно и эффектино реализовать бинарную кучу как массив а не как реальное дерево(но разумеется можно и реализовать как реальное дерево)
- Для узла с индексом i:
    
    - Индекс левого потомка: $$ 2*i+1 $$
    - Индекс правого потомка: $$ 2*i+2 $$
    - Индекс родителя: $$ \frac{i−1​} {2} $$

### поэтапно создадим максимальную бинарную кучу 

Заданного множества чисел `[4, 3, 7, 1, 5]` и визуализируем каждый шаг изменения дерева

### Шаг 1: Начальное состояние

Начальное состояние дерева:
 ```
 4
```

### Шаг 2: Добавление элемента 3

Добавляем элемент 3:
```
	 4
   /
  3
```

### Шаг 3: Добавление элемента 7

Добавляем элемент 7 и выполняем операцию "погружение" (sink): (обменяли позициями 4 и 7, так как это нарушало своиство дерева что каждый узел поддерева является наибольшим)
```
    4              7
   / \     =>     / \
  3   7          3   4
```

### Шаг 5: Добавление элемента 5

Добавляем элемент 5 и выполняем операцию "погружение" (sink): обменяли позициями 3 и 5, так как это нарушало своиство дерева что каждый узел поддерева является наибольшим)
```
    7             7
   / \           / \
  3   4    =>   5   4
 / \           / \
1   5         1   3
```

### Итоговое дерево

Итоговое дерево после всех операций:
```    
    7
   / \
  5   4
 / \
1   3
```
Таким образом, мы получили максимальную бинарную кучу, где каждый узел является наибольшим в своем поддереве.

## Добавление элемента

- Новый элемент добавляется в конец массива/дерева

- Затем выполняется "просеивание вверх" (sift-up/bubble-up): элемент "всплывает" вверх, пока не будет восстановлено свойство кучи

- Сложность: O(log⁡n)

## Извлечение максимума
1) Корневой элемент (максимум) извлекается

2) На его место ставится последний элемент массива

3) Затем выполняется "просеивание вниз" (sift-down): элемент "опускается" вниз, пока не будет восстановлено свойство кучи

- Сложность: O(log⁡n)

## Изменение приоритета

- Если приоритет элемента меняется, нарушается свойство кучи
- Применяется "просеивание вверх" или "вниз", чтобы восстановить свойство

## Применение

Вот несколько случаев, когда использование бинарной кучи является эффективным:

1. **Очередь с приоритетом**:
    
    - Бинарная куча является основой для реализации очереди с приоритетом, где элементы извлекаются в порядке их приоритетов.
    
    - Операции вставки и извлечения элемента с наивысшим (или низшим) приоритетом выполняются за время O(log n), что делает её очень эффективной.
    
2. **Сортировка (Пирамидальная сортировка)**:
    
    - Алгоритм сортировки, известный как пирамидальная сортировка (heapsort), использует бинарную кучу для сортировки массива за время O(n log n).
    
    - Этот алгоритм не требует дополнительной памяти (кроме стека вызовов) и является устойчивым к входным данным.
    
3. **Алгоритмы на графах**:
    
    - Алгоритм Дейкстры для поиска кратчайших путей в графе использует очередь с приоритетом, реализованную на основе бинарной кучи.
    
    - Алгоритм Прима для нахождения минимального остовного дерева также использует бинарную кучу для выбора ребер с минимальным весом.
    
4. **Симуляция событий**:
    
    - В симуляциях, где события должны обрабатываться в порядке их времени возникновения, можно использовать бинарную кучу для хранения событий.
    
    - Это позволяет быстро извлекать события с наименьшим временем.
    
5. **Планирование задач**:
    
    - В системах управления задачами, где задачи должны выполняться в порядке их приоритетов, можно использовать бинарную кучу.
    
    - Это позволяет быстро извлекать задачи с наивысшим приоритетом.
    
6. **Обработка данных в реальном времени**:
    
    - В системах, где данные должны обрабатываться в порядке их важности, например, в системах мониторинга или управления трафиком, можно использовать бинарную кучу.
    
    - Это позволяет быстро извлекать данные с наивысшим приоритетом.
    
7. **Сжатие данных (алгоритм Хаффмана)**:
    
    - Алгоритм Хаффмана для сжатия данных использует очередь с приоритетом для построения оптимального префиксного кода.
    
    - Бинарная куча позволяет эффективно извлекать символы с наименьшей частотой.


## D-кучи
**Термин "d-куча" (d-ary heap) является обобщением бинарной кучи (binary heap). В бинарной куче каждый узел имеет не более двух дочерних узлов, тогда как в d-куче каждый узел может иметь до d дочерних узлов. Таким образом, бинарная куча — это частный случай d-кучи, где d = 2.**

### Основные характеристики d-кучи:

1. **Степень узла**:
    
    - В d-куче каждый узел имеет не более d дочерних узлов.
    - При d = 2, d-куча превращается в бинарную кучу.
2. **Свойство кучи**:
    
    - Как и в бинарной куче, d-куча может быть минимальной (min-heap) или максимальной (max-heap).
    - В минимальной d-куче значение каждого узла больше или равно значению его родителя.
    - В максимальной d-куче значение каждого узла меньше или равно значению его родителя.
3. **Операции**:
    
    - Основные операции в d-куче (вставка, извлечение минимального/максимального элемента) также выполняются с использованием операций "всплытие" (swim) и "погружение" (sink), аналогично бинарной куче.
    - Время выполнения этих операций зависит от значения d. Вставка выполняется за время O(log n), а извлечение минимального/максимального элемента — за время O(d log n / log d).

### Преимущества d-кучи:

1. **Больше дочерних узлов**:
    
    - При больших значениях d (например, d = 4 или d = 8), d-куча может быть более эффективной в некоторых приложениях, таких как обработка больших объемов данных, где часто требуется извлечение минимального/максимального элемента.
    - Операция "всплытие" (swim) выполняется быстрее, так как узел может быть ближе к корню дерева.

2. **Гибкость**:
    
    - d-куча позволяет настраивать степень узла в зависимости от конкретных требований приложения, что может улучшить производительность в определенных сценариях.

### Примеры использования d-кучи:

1. **Очередь с приоритетом**:
    
    - d-куча может использоваться для реализации очереди с приоритетом, где требуется быстрое извлечение минимального/максимального элемента.
        
2. **Алгоритмы на графах**:
    
    - Алгоритм Дейкстры и другие алгоритмы на графах могут использовать d-кучу для улучшения производительности.
        
3. **Сжатие данных**:
    
    - Алгоритм Хаффмана может использовать d-кучу для построения оптимального префиксного кода.
        

В целом, d-куча является мощным инструментом для решения задач, где требуется эффективное управление элементами с приоритетами, и позволяет настраивать степень узла в зависимости от конкретных требований приложения.

## АНАЛИЗ КОЭФФИЦИЕНТА ВЕТВЛЕНИЯ
Теперь, после знакомства с d-ичными кучами, правомерно задать вопрос: не будет ли
достаточно обычной двоичной кучи? Дает ли какие-то преимущества более высокий
коэффициент ветвления?
## Нужны ли d-ичные кучи?
[доработать]