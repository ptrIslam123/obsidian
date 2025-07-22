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

Все, что делатеся с помощью команды `RUN ...` (или любой другой команды `RUN`) во время выполнения `docker build`, **полностью сохраняется в образе** и будет **доступно при запуске контейнера через `docker run`** . Каждая команда (включая `RUN`) создает **новый слой (layer)** в образе.

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

## Пример Dockerfile для развертывания dpdk

```Dockerfile
# Базовый образ
FROM ubuntu:22.04

# Установка переменных среды для автоматического ответа при apt
ENV DEBIAN_FRONTEND=noninteractive

# Обновление системы и установка базовых инструментов
RUN apt-get update && \
apt-get install -y --no-install-recommends \
build-essential gcc g++ make cmake git wget curl \
python3 python3-pip pkg-config libnuma-dev libssl-dev \
libpcap-dev libsctp-dev vim nano iproute2 net-tools \
tcpdump nmap wireshark tshark hping3 dnsutils netcat-openbsd \
socat strace lsof iperf3 ethtool bridge-utils vlan iptables \ 
whois gdb valgrind clang bear openssh-client libelf-dev \
meson python3-pyelftools linux-headers-generic \
&& rm -rf /var/lib/apt/lists/*

# Очистка кэша (экономим место)
RUN apt-get clean && \
rm -rf /tmp/* /var/tmp/*

# Установка Python-инструментов для анализа и тестирования
RUN pip3 install --no-cache-dir \
scapy paramiko pwntools requests beautifulsoup4 cryptography

# Настройка директории для DPDK
WORKDIR /opt/dpdk

# Версия DPDK
ARG DPDK_VERSION=25.07

# Скачивание исходников DPDK
RUN if [ ! -d "dpdk" ]; then \
git clone --recurse-submodules https://github.com/DPDK/dpdk.git ; \
fi


# Сборка DPDK через Meson
RUN cd dpdk && mkdir build && cd build && \
meson setup .. \
--prefix=/usr \
--libdir=lib/x86_64-linux-gnu \
--buildtype=release \
--default-library=static \
-Dexamples=all && \
ninja && ninja install && \
ldconfig

# Добавляем переменные окружения DPDK
ENV RTE_SDK=/opt/dpdk
ENV RTE_TARGET=build
```

```bash
# пример команды для сборки орбраза
docker build -t dpdk:1.0 .

# Пример команды тестового запуска образа в интерактивном режиме 
docker run -it dpdk:1.0 

# Пример команды тестового запуска образа в интерактивном режиме \
# и с указанием общей разделяемой директории
docker run -it -v /home/islam_kardanov/c/tmp/dpdk-test/src:/home/src dpdk:1.0
```
