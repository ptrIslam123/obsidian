**Алгоритм K-средних (или K-means) — это один из самых популярных методов кластеризации, который используется для разбиения набора данных на заданное количество кластеров. Давайте подробно рассмотрим, как он работает.**

## Принцип работы алгоритма K-средних

1. **Выбор количества кластеров (k):** Количество кластеров k обычно определяется на основе априорных знаний о данных или с использованием методов, таких как метод локтя (elbow method) или silhouette score, чтобы определить оптимальное количество кластеров.

2. **Инициализация центроидов:** Центроиды (центры масс) часто инициализируются случайным образом, но могут быть использованы и другие методы, такие как k-means++, чтобы улучшить начальное расположение центроидов.

3. **Расчет расстояний:** Для каждой точки (признака) вычисляется расстояние до каждого из текущих центроидов. Обычно используется евклидово расстояние, но могут быть применены и другие метрики.

4. **Присвоение точек кластерам:** Точки (признаки) присваиваются кластерам на основе минимального расстояния до центроидов. Например, для признака/точки x1 вводим d1 как квадрат расстояния от x1 до c1 (центр масс кластера C1) и d2 как квадрат расстояния от x1 до c2 (центр масс кластера C2). x1  будет присвоен кластеру C1, если d1 < d2.

5. **Пересчет центроидов:** После того как все точки (признаки) распределены по кластерам на основе минимального расстояния до центроидов, необходимо обновить положение центроидов. Это делается для того, чтобы центроиды более точно представляли собой "центры масс" своих кластеров.

	1. **Вычисление новых центроидов:**
	   - Для каждого кластера Ci вычисляется новое положение центроида ci.
	   - Новое положение центроида ci вычисляется как среднее арифметическое всех точек, принадлежащих кластеру Ci.
	   - Если Ci содержит точки $$ x_1, x_2, \ldots, x_n $$, то новый центроид ci вычисляется по формуле:
	     $$
	     c_i = \frac{1}{n} \sum_{j=1}^n x_j
	     $$
	     где n — количество точек в кластере Ci.
	
	2. **Проверка сходимости:**
	   - После пересчета центроидов проверяется, насколько изменились их положения по сравнению с предыдущей итерацией.
	   
	   - Если изменения центроидов незначительны (например, меньше заданного порога), то алгоритм считается сошедшимся, и процесс завершается.
	   
	   - Если изменения центроидов значительны, то алгоритм продолжает работу, повторяя шаги 3 и 4 (расчет расстояний и присвоение точек кластерам) с новыми центроидами.
	   
	Таким образом, пересчет центроидов является важным шагом в алгоритме k-средних, так как он позволяет постепенно улучшать качество кластеризации, приближая центроиды к реальным центрам масс кластеров. Этот процесс повторяется до тех пор, пока не будет достигнута стабильность центроидов или не будет выполнено заданное количество итераций.

6. **Итерации:** Процесс присваивания точек кластерам и пересчета центроидов повторяется до тех пор, пока центроиды не перестанут изменяться или изменения не станут незначительными. Это означает, что алгоритм сошелся к некоторому стабильному состоянию.

7. **Оценка качества кластеризации:** Алгоритм k-средних не требует наличия выборки для тренировки, так как это метод обучения без учителя. Однако можно оценить качество кластеризации с использованием метрик, таких как сумма квадратов ошибок (SSE), silhouette score или других, чтобы определить, насколько хорошо данные разделены на кластеры и разработать более осмысленный перерасчет координат для новых центров масс кластеров на основе ошибки.

### Завершение работы алгоритма

Алгоритм завершает свою работу, когда:

- Центры кластеров остаются неизменными после итерации.
- Или достигается заранее установленное количество итераций.

### Цель алгоритма

Основная цель алгоритма K-средних — минимизация суммы квадратов расстояний между объектами и центрами их кластеров. Это называется инерцией. Формально это можно записать как:

$$
\text{Inertia} = \sum_{i=1}^{k} \sum_{x \in C_i} d(x, c_i)^2
$$

где:
- $$ C_i $$ — набор объектов в $$ i $$-ом кластере,
- $$ c_i $$ — центр $$ i $$-ого кластера,
- $$ d(x, c_i) $$ — расстояние между объектом $$ x $$ и центром $$ c_i $$
## Преимущества и недостатки

### Преимущества:
- Простота реализации и понимания.
- Быстрота работы, особенно на больших наборах данных.
- Хорошо работает при наличии четко разделенных кластеров.

### Недостатки:
- Необходимость заранее задавать количество кластеров $$ k $$
- Чувствительность к выбору начальных центров (может привести к локальным минимумам).

- Неэффективен при наличии шумов и выбросов в данных.

- **Итеративный процесс поиска ближайшего кластерного центра для элементов данных ограничен его радиусом, поэтому итоговый кластер похож на плотную сферу. Это может стать проблемой, если фактическая форма кластера, например, эллипс. Тогда кластер может быть усечен, а некоторые его члены отнесены к другому.**

Алгоритм K-средних широко используется в различных областях, включая анализ данных, маркетинг, обработку изображений и многие другие.