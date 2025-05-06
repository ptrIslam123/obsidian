
### 1. **Для систем с `systemd` (большинство современных дистрибутивов)**
```bash
systemctl list-unit-files --type=service
```
или с фильтрацией по состоянию:
```bash
systemctl list-units --type=service --all
```
**Пояснение:**
- `--type=service` — показывает только сервисы.
- `--all` — включает неактивные сервисы.

**Пример вывода:**
```
UNIT FILE                                  STATE
sshd.service                              enabled
nginx.service                             disabled
...
```

### 2. **Для систем с `init.d` (старые дистрибутивы, например, Debian 7)**
```bash
ls /etc/init.d/
```
или
```bash
service --status-all
```

### 3. **Просмотр только активных (запущенных) сервисов**
```bash
systemctl list-units --type=service --state=running
```
или
```bash
service --status-all | grep +
```

### 4. **Просмотр сервисов, которые запускаются при загрузке**
```bash
systemctl list-unit-files --type=service --state=enabled
```

### 5. **Проверка состояния конкретного сервиса**
```bash
systemctl status <имя_сервиса>
```
или
```bash
service <имя_сервиса> status
```

### 6. **Альтернативные утилиты**
- **`chkconfig`** (в RHEL/CentOS 6 и старых версиях):
  ```bash
  chkconfig --list
  ```
- **`sv`** (в системах с `runit`):
  ```bash
  sv status /var/service/*
  ```

### 7. **Просмотр сервисов через `ps`**
```bash
ps aux | grep -E '^[^ ]+ +[0-9]+ +[0-9.]+ +[0-9.]+ +[0-9]+ +[^ ]+ +[^ ]+ +[^ ]+ +[^ ]+ +[^ ]+ +[^ ]+ +(.*)$'
```
(это сложное регулярное выражение пытается отфильтровать процессы-демоны)

### 8. **Использование `netstat`/`ss` для просмотра сетевых сервисов**
```bash
sudo netstat -tulnp
```
или
```bash
sudo ss -tulnp
```

**Совет:** В большинстве современных дистрибутивов (Ubuntu 16.04+, Debian 8+, CentOS 7+) используется `systemd`, поэтому первый метод (`systemctl`) будет наиболее полезным.