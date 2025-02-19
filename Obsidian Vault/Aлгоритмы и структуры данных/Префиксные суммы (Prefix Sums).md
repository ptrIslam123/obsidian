**Префиксные суммы** — это техника предварительного вычисления сумм элементов массива, которая позволяет быстро отвечать на запросы о сумме элементов в подмассиве. Эта техника особенно полезна, когда требуется многократно вычислять суммы различных подмассивов одного и того же массива.

### Основные идеи:

1. **Префиксная сумма:**
    * Префиксная сумма для индекса `i` — это сумма всех элементов массива от начала до индекса `i`.
    * Обозначается как `prefix_sum[i]`.

2. **Вычисление префиксных сумм:**
    * Для массива `arr`, префиксная сумма `prefix_sum[i]` вычисляется как:
    $$ \text{prefix\_sum}[i] = \text{prefix\_sum}[i-1] + \text{arr}[i] $$
    * Базовый случай: `prefix_sum[0] = arr[0]`.

3. **Использование префиксных сумм:**
    * Сумма элементов подмассива от индекса `i` до индекса `j` вычисляется как:
    $$ \text{sum}(i, j) = \text{prefix\_sum}[j] - \text{prefix\_sum}[i-1] $$
    * Если `i = 0`, то `sum(0, j) = prefix_sum[j]`.

### Пример:

Рассмотрим массив `arr = [1, 2, 3, 4, 5]`.

1. **Вычисление префиксных сумм:**
    * `prefix_sum[0] = 1`
    * `prefix_sum[1] = 1 + 2 = 3`
    * `prefix_sum[2] = 3 + 3 = 6`
    * `prefix_sum[3] = 6 + 4 = 10`
    * `prefix_sum[4] = 10 + 5 = 15`

    Таким образом, массив префиксных сумм будет `prefix_sum = [1, 3, 6, 10, 15]`.

2. **Использование префиксных сумм:**
    * Найдем сумму элементов подмассива от индекса `1` до индекса `3`:
    $$ \text{sum}(1, 3) = \text{prefix\_sum}[3] - \text{prefix\_sum}[0] = 10 - 1 = 9 $$
    * Найдем сумму элементов подмассива от индекса `0` до индекса `2`:
    $$  \text{sum}(0, 2) = \text{prefix\_sum}[2] = 6 $$

### Пример кода на Python:

```python
def calculate_prefix_sums(arr):
    prefix_sum = [0] * len(arr)
    prefix_sum[0] = arr[0]
    for i in range(1, len(arr)):
        prefix_sum[i] = prefix_sum[i - 1] + arr[i]
    return prefix_sum

def get_subarray_sum(prefix_sum, i, j):
    if i == 0:
        return prefix_sum[j]
    return prefix_sum[j] - prefix_sum[i - 1]

# Пример использования
arr = [1, 2, 3, 4, 5]
prefix_sum = calculate_prefix_sums(arr)

# Запросы на сумму подмассивов
print(get_subarray_sum(prefix_sum, 1, 3))  # Вывод: 9
print(get_subarray_sum(prefix_sum, 0, 2))  # Вывод: 6
```

### Объяснение:

1. **Вычисление префиксных сумм:**
    * Создаем массив `prefix_sum` длиной, равной длине исходного массива.
    * Заполняем `prefix_sum` по формуле `prefix_sum[i] = prefix_sum[i-1] + arr[i]`.

2. **Использование префиксных сумм:**
    * Для запроса суммы подмассива от индекса `i` до индекса `j` используем формулу `prefix_sum[j] - prefix_sum[i-1]`.
    * Если `i = 0`, то сумма подмассива равна `prefix_sum[j]`.

### Преимущества:

* **Эффективность:** Вычисление суммы подмассива за время **O(1)** после предварительного вычисления префиксных сумм за время **O(n)**.
* **Простота реализации:** Легко реализуется и подходит для множества задач, связанных с суммами подмассивов.

### Применение:

* **Запросы на сумму подмассивов:** Быстрое вычисление суммы элементов в подмассиве.
* **Динамическое программирование:** Используется в задачах, где требуется многократно вычислять суммы подмассивов.
* **Оптимизация алгоритмов:** Ускоряет алгоритмы, которые требуют частого обращения к суммам подмассивов.

### Заключение:

Префиксные суммы — это мощная техника, которая позволяет эффективно вычислять суммы элементов в подмассивах после предварительной обработки массива. Это делает ее особенно полезной в задачах, где требуется многократно вычислять суммы различных подмассивов одного и того же массива.