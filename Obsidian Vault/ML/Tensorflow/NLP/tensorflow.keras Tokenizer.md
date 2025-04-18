`Tokenizer` из TensorFlow/Keras — это мощный инструмент для преобразования текста в числовые данные, которые могут быть использованы в моделях машинного обучения. Он позволяет создавать словари, преобразовывать тексты в последовательности индексов и приводить их к одинаковой длине с помощью паддинга. Важно помнить, что токенизатор должен быть обучен только на обучающих данных, чтобы избежать утечки информации из тестовой выборки.

```python
from tensorflow.keras.preprocessing.text import Tokenizer

# По умолчанию `Tokenizer` использует все слова, встречающиеся в тексте, и 
# создает словарь, который сопоставляет каждое слово с уникальным индексом.
tokenizer = Tokenizer()

# Метод `fit_on_texts` используется для обучения токенизатора 
# на наборе текстовых данных.
tokenizer.fit_on_texts(train_X)
```


### Обучение Tokenizer на текстовых данных

Метод `fit_on_texts` используется для обучения токенизатора на наборе текстовых данных. Вот что происходит на этом этапе:

- **Токенизация**: Текст разбивается на слова (токены). По умолчанию токенизатор использует пробелы и знаки препинания для разделения слов.

- **Создание словаря**: Токенизатор создает словарь, который сопоставляет каждое уникальное слово с уникальным целочисленным индексом. Индексы начинаются с 1 (0 обычно зарезервирован для специального токена, такого как `<PAD>`).

- **Преобразование текста в последовательности индексов**: После обучения токенизатора вы можете использовать его для преобразования текстовых данных в последовательности индексов слов

- **Паддинг последовательностей**: Поскольку тексты могут иметь разную длину, часто необходимо привести все последовательности к одинаковой длине. Это делается с помощью функции `pad_sequences`. Функция `pad_sequences` добавляет нули (или другие специальные токены) в конец последовательностей, чтобы все они имели одинаковую длину `max_len`. Если последовательность длиннее `max_len`, она усекается с конца.

Например, если `train_X` содержит следующие тексты:
```python
train_X = [
    "Hello world",
    "Hello everyone",
    "This is a test"
]
# Токенизатор создаст словарь, который может выглядеть примерно так:
{
    "Hello": 1,
    "world": 2,
    "everyone": 3,
    "This": 4,
    "is": 5,
    "a": 6,
    "test": 7
}

# Преобразование текста в последовательности индексов
train_sequences = tokenizer.texts_to_sequences(train_X)
train_sequences = [
    [1, 2],       # "Hello world"
    [1, 3],       # "Hello everyone"
    [4, 5, 6, 7]  # "This is a test"
]

max_len = 100  # максимальная длина последовательности
train_sequences = pad_sequences(train_sequences, maxlen=max_len, padding='post', truncating='post')
```

