### Основные команды (инструкции) Dockerfile  

Dockerfile состоит из **инструкций**, которые выполняются по порядку при сборке образа. Вот основные из них:  

---

### 1. **`FROM`** – Базовый образ  
Определяет, на основе какого образа будет строиться ваш контейнер.  
```dockerfile
FROM ubuntu:22.04        # Официальный образ Ubuntu
FROM python:3.9-slim     # Облегченный образ Python
FROM alpine:latest       # Минималистичный Alpine Linux
```  
**Важно:** Всегда указывайте тег (`:22.04`, `:3.9-slim`), иначе будет использоваться `latest`.  

---

### 2. **`WORKDIR`** – Рабочая директория  
Устанавливает текущую папку внутри контейнера (автоматически создается, если не существует).  
```dockerfile
WORKDIR /app   # Все последующие команды выполняются в /app
```  

---

### 3. **`COPY` и `ADD`** – Копирование файлов  
Переносит файлы **с хоста** (вашего компьютера) **в контейнер**.  

#### `COPY` (простое копирование)  
```dockerfile
COPY ./src /app/src      # Копирует папку `src` в `/app/src`
COPY file.txt /app/      # Копирует `file.txt` в `/app/`
```  

#### `ADD` (с дополнительными возможностями)  
- Может распаковывать архивы (`.tar`, `.gz`).  
- Может загружать файлы по URL (но лучше использовать `curl` + `RUN`).  
```dockerfile
ADD https://example.com/file.tar.gz /tmp/  # Не рекомендуется
ADD archive.tar.gz /app/                   # Распакует архив
```  
**Совет:** Лучше использовать `COPY`, если не нужны особые функции `ADD`.  

---

### 4. **`RUN`** – Выполнение команд при сборке  
Устанавливает пакеты, настраивает окружение и т. д.  
```dockerfile
RUN apt-get update && apt-get install -y curl  # Ubuntu/Debian
RUN pip install -r requirements.txt           # Python
RUN npm install                               # Node.js
```  
**Оптимизация:** Объединяйте команды в один `RUN` через `&&`, чтобы уменьшить количество слоев.  

---

### 5. **`ENV`** – Переменные окружения  
Устанавливает переменные, которые будут доступны в контейнере.  
```dockerfile
ENV PYTHONUNBUFFERED=1
ENV DB_HOST=postgres
```  

---

### 6. **`EXPOSE`** – Объявление порта  
Говорит Docker, что контейнер слушает указанный порт (но не пробрасывает его!).  
```dockerfile
EXPOSE 80      # Контейнер "открывает" порт 80
EXPOSE 3000    # Например, для Node.js-приложения
```  
**Важно:** Чтобы пробросить порт наружу, используйте `-p` при запуске:  
```bash
docker run -p 8080:80 my-image  # Хост:8080 → Контейнер:80
```  

---

### 7. **`CMD` и `ENTRYPOINT`** – Запуск процесса  
Определяют, какая команда выполнится при старте контейнера.  

#### `CMD` (можно переопределить при `docker run`)  
```dockerfile
CMD ["python", "app.py"]           # Предпочтительный формат (JSON-массив)
CMD python app.py                   # Shell-формат (менее надежный)
```  
**Пример переопределения:**  
```bash
docker run my-image python server.py  # Заменит CMD
```  

#### `ENTRYPOINT` (жестко фиксирует команду)  
```dockerfile
ENTRYPOINT ["python"]  
CMD ["app.py"]          # Теперь можно передать только аргументы:
```  
```bash
docker run my-image server.py  # Выполнит `python server.py`
```  

---

### 8. **`.dockerignore`** – Игнорирование файлов  
Аналог `.gitignore`. Указывает, какие файлы **не** копировать в образ.  
```text
.git/
node_modules/
*.log
```  

---

## Шаринг файлов между Docker и основной системой  

Чтобы обмениваться файлами между хост-системой (вашим компьютером) и контейнером, Docker предоставляет несколько механизмов:  

---

## 1. **Volumes (Тома Docker)**  
**Лучший способ для постоянного хранения данных.**  

### Особенности:  
✅ Управляются Docker (автоматическое создание, резервное копирование).  
✅ Высокая производительность (особенно на Linux).  
✅ Данные сохраняются после удаления контейнера.  

### Создание и использование:  
#### Способ 1: Создать том вручную  
```bash
docker volume create my_volume  # Создает том
docker run -v my_volume:/data alpine  # Подключает том в /data
```  

#### Способ 2: Автоматическое создание  
```bash
docker run -v /data alpine  # Docker сам создаст том
```  

#### Просмотр томов:  
```bash
docker volume ls
docker volume inspect my_volume  # Путь к данным на хосте (Linux: /var/lib/docker/volumes/)
```  

---

## 2. **Bind Mounts (Привязка директорий)**  
**Прямое подключение папки с хоста в контейнер.**  
### Особенности:  
🔹 Файлы изменяются **в реальном времени** с обеих сторон.  
🔹 Подходит для разработки (например, live-релоад кода).  
🔹 Нет автоуправления (Docker не резервирует данные).  

### Пример:  
```bash
# Подключение папки ./host_dir в /container_dir
docker run -v $(pwd)/host_dir:/container_dir alpine

# Для Windows (PowerShell):
docker run -v ${PWD}/host_dir:/container_dir alpine
```  

### Варианты записи:  
```bash
# Полный формат (рекомендуется для Windows/macOS)
docker run --mount type=bind,source="$(pwd)"/host_dir,target=/container_dir alpine

# Короткий формат (Linux/macOS)
docker run -v /абсолютный/путь/host_dir:/container_dir alpine
```  

---

## 3. **Tmpfs Mounts (Временная память)**  
**Хранение данных только в RAM (исчезает после остановки контейнера).**  

### Пример:  
```bash
docker run --tmpfs /cache alpine
```  

---

## 🔹 **Разница между `-v` и `--mount`**  
| Параметр       | `-v` (короткий)          | `--mount` (полный)          |
|---------------|--------------------------|----------------------------|
| Читаемость     | Проще                   | Более явный                |
| Поддержка      | Все версии Docker       | Только новые версии        |
| Доп. опции     | Нет                     | Есть (readonly, selinux)  |

---

## **Практические примеры**  

### 1. **Разработка Python-приложения**  
```bash
# Код редактируется на хосте, а контейнер сразу видит изменения
docker run -v $(pwd)/app:/app -p 8000:8000 python:3.9 sh -c "cd /app && pip install -r requirements.txt && python app.py"
```  

### 2. **База данных PostgreSQL**  
```bash
# Данные БД сохраняются в томе даже после удаления контейнера
docker run -v pg_data:/var/lib/postgresql/data -e POSTGRES_PASSWORD=12345 postgres
```  

### 3. **Nginx с конфигом с хоста**  
```bash
# Подключаем кастомный nginx.conf
docker run -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf -p 80:80 nginx
```  

---

##  **Ограничения и подводные камни**  
1. **Права доступа**  
   - Файлы в контейнере создаются от имени `root` (если не указан `USER`).  
   - Решение:  
     ```dockerfile
     RUN chown -R 1000:1000 /data  # Меняем владельца (UID/GID пользователя на хосте)
     ```  

2. **Производительность на macOS/Windows**  
   - Bind Mounts работают медленнее (из-за виртуализации).  
   - Решение: Используйте `delegated` (для macOS):  
     ```bash
     docker run -v $(pwd):/app:delegated alpine
     ```  

3. **Пути в Windows**  
   - Используйте `/c/путь` (Git Bash) или `C:\путь` (PowerShell).  

---

## **Когда что использовать?**  
| Механизм          | Лучший случай использования         |
|-------------------|------------------------------------|
| **Volumes**       | Базы данных, постоянные данные     |
| **Bind Mounts**   | Разработка, конфиги                |
| **Tmpfs**         | Временные файлы (кеш, сессии)      |

**Совет:** Для продакшена предпочитайте `volumes`, для разработки — `bind mounts`.  

## Пример простого Dockerfile

```Dockerfile
# Используем Ubuntu 24.04
FROM ubuntu:24.04

# Обновляем систему и ставим зависимости
RUN apt-get update && apt-get install -y \
build-essential \
meson \
ninja-build \
wget \
git \
linux-headers-generic \
libnuma-dev \
pkg-config \
python3-pip \
kmod

CMD ["/bin/bash"]
```