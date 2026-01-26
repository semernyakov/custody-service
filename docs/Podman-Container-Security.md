---
created: 2026-01-24 14:33
tags:
  - Vault
  - tasks-service-models
  - HashiCorpVault
author: Иван Семерняков
---
# Podman / Container Security - Детальное объяснение

## **Контекст безопасности контейнеров**

### **Проблема:**

Контейнеры по умолчанию имеют **слишком много привилегий**, что может быть использовано злоумышленниками для:
- Эскалации привилегий на хосте
- Доступа к хостовым ресурсам
- Breakout из контейнера
- Атак на другие контейнеры

### **Решение:**

Принцип **минимальных привилегий** - давать контейнеру только то, что ему действительно нужно.

---

## **1. `--read-only` Root Filesystem**

### **Что делает:**

```bash
podman run --read-only nginx
```

Делает корневую файловую систему контейнера **доступной только для чтения**.

### **Зачем нужно:**

**Безопасность:**
- **Предотвращает persistence:** Злоумышленник не может записать файлы в контейнер
- **Защита от модификации:** Никто не может изменить бинарники или конфиги
- **Immutable infrastructure:** Контейнер остается в неизменном состоянии

**Аудит и compliance:**
- **Детерминированное состояние:** Контейнер всегда одинаковый
- **Обнаружение изменений:** Любая попытка записи логируется
- **Соответствие стандартам:** Требуется многими security frameworks

### **Практический пример:**

**Без `--read-only` (опасно):**

```bash
# Злоумышленник может:
docker exec -it vulnerable_container bash
echo "malicious_code" > /bin/nginx  # Испортить бинарник
touch /backdoor.sh                  # Создать backdoor
```

**С `--read-only` (безопасно):**

```bash
# Те же действия приведут к:
podman exec -it secure_container bash
echo "test" > /tmp/test.txt
# Ошибка: Read-only file system

# Даже root внутри контейнера не может писать!
```

### **Как работать с данными (volumes):**

```bash
# Правильный подход: данные отдельно от кода
podman run \
  --read-only \
  -v /data/nginx/html:/usr/share/nginx/html:rw \
  -v /data/nginx/logs:/var/log/nginx:rw \
  -v /tmp/nginx:/tmp:rw \
  nginx
```

**Типичные RW директории:**

- `/tmp`, `/var/tmp` - временные файлы
- `/var/log` - логи
- `/var/lib/<service>` - данные приложения
- `/run`, `/proc`, `/sys` - runtime (обычно tmpfs)

---

## **2. Ограничение Capabilities**

### **Что такое Linux Capabilities?**

Вместо одного бита "root/not-root" в Linux есть **40+ отдельных привилегий**:

```bash
# Посмотреть все capabilities
capsh --print

# Основные опасные capabilities:
CAP_SYS_ADMIN      # ~root (администрирование системы)
CAP_NET_RAW        # RAW sockets (пакеты, ping)
CAP_SYS_MODULE     # Загрузка модулей ядра
CAP_SYS_PTRACE     # Отладка других процессов
CAP_DAC_OVERRIDE   # Обход проверок доступа к файлам
CAP_CHOWN          # Изменение владельца файлов
```

### **Podman по умолчанию:**

Podman **автоматически удаляет опасные capabilities** при запуске rootless контейнеров:

```bash
# Сравнение Docker vs Podman (rootless)
docker run --user 1000 alpine cat /proc/self/status | grep Cap
# CapEff: 0000003fffffffff  # МНОГО возможностей!

podman run --user 1000 alpine cat /proc/self/status | grep Cap  
# CapEff: 0000000000000000  # НЕТ capabilities!
```

### **Проверить текущие capabilities:**

```bash
# Внутри контейнера
cat /proc/self/status | grep -i cap

# Снаружи
podman inspect <container> | grep -A 10 SecurityOpt

# Или с помощью capsh
apk add libcap  # Alpine
apt-get install libcap2-bin  # Debian/Ubuntu
capsh --print
```

### **Явное ограничение capabilities:**

```bash
# Удалить все, добавить только нужные
podman run \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  nginx

# Только для binding на порты <1024 нужны привилегии
# NET_BIND_SERVICE позволяет слушать на порту 80/443
```

**Минимальный набор для типичных приложений:**
```bash
# Веб-сервер (nginx/apache)
--cap-drop=ALL --cap-add=NET_BIND_SERVICE

# База данных (PostgreSQL)
--cap-drop=ALL --cap-add=DAC_OVERRIDE  # для доступа к файлам БД

# Ничего не нужно (большинство приложений)
--cap-drop=ALL
```

---

## **3. Rootless Containers (`podman info | grep rootless`)**

### **Что такое rootless?**
Контейнеры запускаются **от обычного пользователя**, а не от root.

```bash
# Проверить режим
podman info | grep rootless

# Ожидаемый вывод:
rootless: true  # ХОРОШО - безопасно
# или
rootless: false # ПЛОХО - запуск от root
```

### **Почему rootless безопаснее:**

**Без rootless (опасно):**
```
Контейнер (root внутри)
    ↓
Демон Docker/Podman (root на хосте)  
    ↓
Хостовая система (полный доступ!)
```

**С rootless (безопасно):**
```
Контейнер (пользователь внутри)
    ↓
Демон Podman (пользователь на хосте)
    ↓
Хостовая система (ограниченные права!)
```

### **Преимущества rootless:**

1. **Изоляция пользователей:**
   ```bash
   # User1 не может видеть контейнеры User2
   user1$ podman ps -a
   CONTAINER ID  IMAGE  # только свои контейнеры
   
   # Даже root не может управлять контейнерами пользователя
   sudo podman ps -a  # Пусто!
   ```

2. **Нет демона с root правами:**
   - Docker: `dockerd` запущен от root
   - Podman: `podman` запущен от пользователя, нет демона

3. **Автоматическое ограничение capabilities:**
   ```bash
   # Rootless автоматически:
   # - Удаляет опасные capabilities
   # - Включает пользовательские namespaces
   # - Применяет seccomp фильтры
   ```

4. **Защита от breakout:**
   Даже если злоумышленник вырвется из контейнера, он останется **обычным пользователем** на хосте.

### **Проверить и настроить:**

```bash
# 1. Проверить текущий режим
podman info | grep -A5 rootless

# 2. Посмотреть mapping пользователей
podman unshare cat /proc/self/uid_map
# Вывод: 0       1000          1
#        контейнер -> хост (1000 = ваш UID)

# 3. Настроить rootless (если нужно)
# /etc/subuid и /etc/subgid
echo "$USER:100000:65536" | sudo tee -a /etc/subuid
echo "$USER:100000:65536" | sudo tee -a /etc/subgid
```

---

## **Полный пример безопасного запуска custody-service**

```bash
#!/bin/bash

# Безопасный запуск custody-service
podman run \
  --name custody-service \
  
  # 1. Безопасность файловой системы
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=64M \
  --tmpfs /run:rw,noexec,nosuid,size=16M \
  -v custody-data:/data:Z,rw \
  -v custody-logs:/var/log:Z,rw \
  
  # 2. Безопасность привилегий
  --user 1000:1000 \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --security-opt=no-new-privileges \
  
  # 3. Безопасность сети
  --network slirp4netns \
  --publish 127.0.0.1:8080:8080 \
  
  # 4. Безопасность ресурсов
  --memory=512m \
  --cpus=1 \
  --pids-limit=100 \
  
  # 5. Безопасность времени
  --read-only-tmpfs=false \
  
  # Образ
  custody-service:latest
```

---

## **Дополнительные меры безопасности**

### **1. `--security-opt=no-new-privileges`**

```bash
# Предотвращает получение новых привилегий
# Например, через setuid бинарники
podman run --security-opt=no-new-privileges alpine
```

### **2. Seccomp профили:**

```bash
# Использовать strict профиль
podman run --security-opt seccomp=/path/to/profile.json

# Или удалить syscalls
podman run --security-opt seccomp=unconfined:chmod,chown
```

### **3. AppArmor/SELinux:**

```bash
# AppArmor
podman run --security-opt apparmor=custody-profile

# SELinux
podman run --security-opt label=type:container_t
```

### **4. Ограничение ресурсов:**

```bash
podman run \
  --memory=256m \           # ОЗУ
  --memory-swap=512m \      # Swap
  --cpus=0.5 \              # CPU
  --blkio-weight=100 \      # IO
  --pids-limit=50 \         # Макс процессов
  --ulimit nofile=1024:1024 # Открытые файлы
```

### **5. Read-only volumes:**

```bash
# Конфиги только для чтения
podman run \
  -v /etc/custody/config.yaml:/app/config.yaml:ro \
  custody-service
```

---

## **Проверка безопасности контейнера**

### **Инструменты аудита:**

```bash
# 1. Проверить конфигурацию запуска
podman inspect <container> | jq '.[0].HostConfig'

# 2. Проверить процессы внутри
podman top <container> auxef

# 3. Сканирование на уязвимости
podman scan custody-service:latest

# 4. Проверить capabilities
podman exec <container> capsh --print

# 5. Аудит с trivy
trivy image custody-service:latest
```

### **Чеклист безопасности:**

```bash
#!/bin/bash
# security_check.sh

echo "1. Проверка rootless:"
podman info | grep rootless

echo -e "\n2. Проверка capabilities:"
podman run --rm alpine cat /proc/self/status | grep Cap

echo -e "\n3. Проверка read-only:"
podman run --read-only --rm alpine touch /test 2>&1 | head -1

echo -e "\n4. Проверка пользователя:"
podman run --rm alpine whoami

echo -e "\n5. Проверка no-new-privileges:"
podman inspect $(podman run -d alpine sleep 100) | grep noNewPrivileges
```

---

## **Опасные паттерны (чего избегать):**

### **❌ ОПАСНО:**

```bash
# 1. Привилегированный контейнер
podman run --privileged nginx

# 2. Монтирование чувствительных директорий хоста
podman run -v /:/host alpine

# 3. Запуск от root
podman run --user root alpine

# 4. Сетевой режим host
podman run --network host nginx

# 5. Отключение security features
podman run --security-opt seccomp=unconfined alpine
```

### **✅ БЕЗОПАСНО:**

```bash
# 1. Rootless с минимальными привилегиями
podman run --user 1000 --cap-drop=ALL alpine

# 2. Read-only с явными volumes
podman run --read-only -v app-data:/data alpine

# 3. Изолированная сеть
podman run --network slirp4netns --publish 127.0.0.1:8080:80 nginx

# 4. Ограниченные ресурсы
podman run --memory=256m --pids-limit=50 alpine
```

---

## **Для custody-service конкретно:**

### **Рекомендуемая конфигурация:**

```Dockerfile
# Containerfile
FROM python:3.11-slim

# Создаем непривилегированного пользователя
RUN useradd -m -u 1000 custody && \
    mkdir -p /app /data /logs && \
    chown -R custody:custody /app /data /logs

USER custody
WORKDIR /app
COPY --chown=custody:custody . .

CMD ["python", "src/main.py"]
```

### **Команда запуска:**

```bash
podman run -d \
  --name custody \
  --read-only \
  --user 1000 \
  --cap-drop=ALL \
  -v custody-vault:/data/vault:Z,rw \
  -v custody-keys:/data/keys:Z,ro \
  -v ./config:/app/config:ro \
  --network slirp4netns \
  --publish 127.0.0.1:8000:8000 \
  custody-service:latest
```

---

## **Итог:**

1. **`--read-only`** - защищает от модификации контейнера
2. **Ограничение capabilities** - убирает лишние привилегии  
3. **Rootless containers** - фундаментальная безопасность

**Для production custody-service обязательно:**

- ✅ Всегда использовать `--read-only`
- ✅ Всегда запускать rootless (`podman info | grep rootless`)
- ✅ Явно удалять ненужные capabilities
- ✅ Запускать от непривилегированного пользователя
- ✅ Использовать volumes для данных (не записывать в контейнер)

Этот подход соответствует **принципу минимальных привилегий** и стандартам безопасности для финансовых приложений, каким является custody-service.