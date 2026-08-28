# Лабораторная работа. Оптимизация настроек ядра для определенных задач

**Продолжительность:** 2 академических часа

**Оборудование:** Виртуальная машина с Debian 11/12 или Ubuntu 20.04/22.04 LTS, сетевое подключение к интернету

---

## Цель работы

Научиться оптимизировать параметры ядра Linux с помощью утилиты `sysctl` для повышения производительности специализированных серверов. В ходе работы вы освоите:

- понимание ключевых параметров ядра, влияющих на производительность файловых серверов и баз данных;
- создание и применение профилей оптимизации для файлового обменника (Samba/NFS);
- создание и применение профилей оптимизации для сервера баз данных (MySQL/PostgreSQL);
- измерение и анализ влияния оптимизаций на производительность;
- сохранение настроек для постоянного применения после перезагрузки.

---

## Теоретическая справка

### Что такое sysctl и зачем нужна оптимизация ядра?

`sysctl` — это утилита, позволяющая в реальном времени изменять параметры ядра Linux. Параметры ядра определяют, как операционная система управляет памятью, сетью, файловой системой и другими ресурсами. Правильная настройка этих параметров может значительно повысить производительность сервера для конкретных задач.

### Почему разные задачи требуют разных настроек?

| Задача | Особенности | Ключевые параметры |
|--------|-------------|---------------------|
| **Файловый сервер** | Много открытых файлов, высокий сетевой трафик, большие объёмы данных | `fs.file-max`, сетевые буферы, параметры NFS/Samba |
| **Сервер БД** | Много соединений, интенсивная работа с памятью и диском, низкая задержка | `vm.swappiness`, параметры разделяемой памяти, `fs.aio-max-nr` |

### Ключевые параметры ядра

| Параметр | Назначение | Для файлового сервера | Для БД |
|----------|------------|----------------------|--------|
| `fs.file-max` | Максимальное количество открытых файловых дескрипторов | Высокое | Высокое |
| `fs.aio-max-nr` | Максимальное число асинхронных операций I/O | Среднее | Высокое |
| `vm.swappiness` | Склонность ядра использовать swap (0-100) | 30 | 10-30 |
| `net.core.somaxconn` | Максимальный размер очереди соединений | Высокое | Высокое |
| `kernel.shmmax` | Максимальный размер сегмента разделяемой памяти | — | Высокое |
| `kernel.shmall` | Общее количество страниц разделяемой памяти | — | Высокое |

---

## Подготовка стенда

### Шаг 1. Установка необходимых пакетов

```bash
sudo apt update
sudo apt install -y sysstat procps net-tools nfs-kernel-server nfs-common \
    samba samba-client postgresql postgresql-contrib mysql-server \
    stress-ng iperf3 sysbench
```

**Пояснение пакетов:**
- `sysstat` — инструменты для мониторинга системы (iostat, sar)
- `nfs-kernel-server`, `samba` — для имитации файлового сервера
- `postgresql`, `mysql-server` — для имитации сервера БД
- `stress-ng`, `sysbench` — для создания нагрузки и тестирования производительности

### Шаг 2. Создание рабочей директории

```bash
mkdir -p ~/lab_kernel_tuning
cd ~/lab_kernel_tuning
```

### Шаг 3. Создание резервной копии текущих настроек

```bash
# Сохранение текущих параметров
sudo sysctl -a > ~/sysctl_original_backup.txt

# Сохранение конфигурационных файлов
sudo cp /etc/sysctl.conf /etc/sysctl.conf.backup
sudo mkdir -p /etc/sysctl.d/backup
sudo cp /etc/sysctl.d/*.conf /etc/sysctl.d/backup/ 2>/dev/null || true

echo "Резервные копии созданы"
```

### Шаг 4. Просмотр текущих параметров

```bash
# Просмотр всех параметров
sudo sysctl -a | head -30

# Ключевые параметры для начала работы
echo "=== КЛЮЧЕВЫЕ ПАРАМЕТРЫ ==="
sudo sysctl fs.file-max
sudo sysctl vm.swappiness
sudo sysctl net.core.somaxconn
sudo sysctl kernel.shmmax
sudo sysctl fs.aio-max-nr
```

---

## Часть 1. Оптимизация для файлового обменника (Samba/NFS) (30 минут)

### Теоретическая справка

Файловые серверы характеризуются:
- большим количеством одновременно открытых файлов;
- интенсивным сетевым трафиком;
- необходимостью эффективного кеширования.

### Шаг 1. Создание профиля для файлового сервера

Создайте файл `/etc/sysctl.d/99-file-server.conf`:

```bash
sudo tee /etc/sysctl.d/99-file-server.conf > /dev/null <<'EOF'
# ============================================
# ОПТИМИЗАЦИЯ ЯДРА ДЛЯ ФАЙЛОВОГО СЕРВЕРА
# ============================================

# -------------------- Файловая система --------------------
# Максимальное количество файловых дескрипторов
# Для файлового сервера требуется высокое значение
fs.file-max = 2097152

# -------------------- Управление памятью --------------------
# Снижаем склонность к использованию swap (для ускорения)
# 30 — хороший баланс для файловых серверов
vm.swappiness = 30

# Увеличиваем минимальный объём свободной памяти (в КБ)
# Предотвращает исчерпание памяти для критических операций
vm.min_free_kbytes = 131072

# -------------------- Сеть и сетевые буферы --------------------
# Максимальный размер очереди соединений
net.core.somaxconn = 65535

# Максимальное количество пакетов в очереди при перегрузке
net.core.netdev_max_backlog = 32768

# Размер буферов сокетов (по умолчанию и максимум)
net.core.rmem_default = 262144
net.core.wmem_default = 262144
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216

# Оптимизация TCP для высоких нагрузок
net.ipv4.tcp_window_scaling = 1
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_max_syn_backlog = 8192

# -------------------- Дисковый I/O --------------------
# Увеличиваем размер очереди запросов (для SCSI/ATA дисков)
# (применяется при загрузке ядра или через sysfs)
# echo 256 > /sys/block/sdX/queue/nr_requests

EOF

echo "Файл конфигурации создан: /etc/sysctl.d/99-file-server.conf"
```

### Шаг 2. Применение настроек

```bash
# Применение всех настроек из /etc/sysctl.d/
sudo sysctl --system

# Или применение конкретного файла
sudo sysctl -p /etc/sysctl.d/99-file-server.conf
```

### Шаг 3. Проверка применения

```bash
echo "=== ПРОВЕРКА ПРИМЕНЕНИЯ ==="
sudo sysctl fs.file-max
sudo sysctl vm.swappiness
sudo sysctl net.core.somaxconn
sudo sysctl net.core.rmem_max
```

### Шаг 4. Настройка ограничений для пользователей (ulimit)

Для файлового сервера также важно увеличить ограничения для пользовательских процессов:

```bash
sudo tee /etc/security/limits.d/99-file-server.conf > /dev/null <<'EOF'
# Ограничения для файлового сервера
* soft nofile 2097152
* hard nofile 2097152
* soft nproc 65536
* hard nproc 65536
EOF

echo "Ограничения ulimit настроены"
```

### Шаг 5. Сравнение производительности

**Тест до оптимизации (если выполнено в начале):**

```bash
# Генерация тестового файла
dd if=/dev/urandom of=/tmp/testfile.bin bs=1M count=100

# Тест чтения
time cat /tmp/testfile.bin > /dev/null

# Тест записи
time dd if=/dev/zero of=/tmp/testfile2.bin bs=1M count=100
```

**Тест после оптимизации — повторите те же команды и сравните время выполнения.**

---

## Часть 2. Оптимизация для сервера баз данных (30 минут)

### Теоретическая справка

Серверы баз данных характеризуются:
- большим количеством соединений;
- интенсивной работой с памятью;
- множеством файлов данных;
- требовательностью к задержкам I/O.

Ключевые параметры для БД:
- `fs.aio-max-nr` — для асинхронного I/O (InnoDB/PostgreSQL)
- `kernel.shmmax` и `kernel.shmall` — для разделяемой памяти
- `vm.swappiness` — низкое значение для ускорения работы с памятью

### Шаг 1. Создание профиля для сервера БД

Создайте файл `/etc/sysctl.d/99-database-server.conf`:

```bash
sudo tee /etc/sysctl.d/99-database-server.conf > /dev/null <<'EOF'
# ============================================
# ОПТИМИЗАЦИЯ ЯДРА ДЛЯ СЕРВЕРА БАЗ ДАННЫХ
# ============================================

# -------------------- Асинхронный I/O --------------------
# Максимальное число операций асинхронного ввода-вывода
# Критично для InnoDB (MySQL) и PostgreSQL
fs.aio-max-nr = 1048576

# -------------------- Файловые дескрипторы --------------------
# Высокое значение для множества файлов данных и соединений
fs.file-max = 4194304

# -------------------- Разделяемая память --------------------
# Максимальный размер сегмента разделяемой памяти (в байтах)
# Для PostgreSQL/MySQL требуется достаточно большой размер
kernel.shmmax = 68719476736

# Общее количество страниц разделяемой памяти
kernel.shmall = 4294967296

# Максимальное количество сегментов разделяемой памяти
kernel.shmmni = 819200

# -------------------- Семафоры --------------------
# Настройка семафоров для работы с БД
kernel.sem = 4096 2147483647 2147483646 512000

# -------------------- Управление памятью --------------------
# Минимальное использование swap — данные должны оставаться в памяти
vm.swappiness = 10

# Процент памяти для "грязных" страниц (ожидающих записи на диск)
# Более низкие значения ускоряют запись, но увеличивают нагрузку на диск
vm.dirty_background_ratio = 5
vm.dirty_ratio = 15

# Переполнение памяти — разрешаем выделение всей запрошенной памяти
vm.overcommit_memory = 1

# Минимальный объём свободной памяти (в КБ)
vm.min_free_kbytes = 262144

# -------------------- Сеть и соединения --------------------
# Максимальный размер очереди соединений
net.core.somaxconn = 4096

# TCP-буферы для многих соединений
net.core.rmem_default = 262144
net.core.wmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_max = 4194304

# Диапазон локальных портов для исходящих соединений
net.ipv4.ip_local_port_range = 40000 65535

# -------------------- TCP Keepalive --------------------
# Более агрессивное обнаружение мёртвых соединений
net.ipv4.tcp_keepalive_time = 60
net.ipv4.tcp_keepalive_intvl = 20
net.ipv4.tcp_keepalive_probes = 3

# Очистка завершённых соединений
net.ipv4.tcp_fin_timeout = 5

EOF

echo "Файл конфигурации создан: /etc/sysctl.d/99-database-server.conf"
```

### Шаг 2. Применение настроек

```bash
sudo sysctl --system
```

### Шаг 3. Проверка применения

```bash
echo "=== ПРОВЕРКА ПРИМЕНЕНИЯ ==="
sudo sysctl fs.aio-max-nr
sudo sysctl kernel.shmmax
sudo sysctl vm.swappiness
sudo sysctl net.ipv4.ip_local_port_range
```

### Шаг 4. Настройка ограничений для БД

Создайте файл `/etc/security/limits.d/99-database.conf`:

```bash
sudo tee /etc/security/limits.d/99-database.conf > /dev/null <<'EOF'
# Ограничения для сервера БД
postgres soft nofile 655360
postgres hard nofile 655360
mysql soft nofile 655360
mysql hard nofile 655360
* soft nofile 2097152
* hard nofile 2097152
EOF

echo "Ограничения ulimit для БД настроены"
```

### Шаг 5. Тестирование производительности БД

**Установка тестовых данных для PostgreSQL:**

```bash
sudo -u postgres psql -c "CREATE DATABASE testdb;"
sudo -u postgres psql -d testdb -c "CREATE TABLE test (id serial PRIMARY KEY, data text);"
```

**Тест вставки данных:**

```bash
# Создание тестового скрипта
cat > /tmp/insert_test.sql <<'EOF'
INSERT INTO test (data) SELECT generate_series(1, 10000), 'test_data';
EOF

# Выполнение теста
time sudo -u postgres psql -d testdb -f /tmp/insert_test.sql
```

**Тест выборки данных:**

```bash
sudo -u postgres psql -d testdb -c "EXPLAIN ANALYZE SELECT * FROM test WHERE id < 100;"
```

---

## Часть 3. Мониторинг и сравнение производительности (25 минут)

### Шаг 1. Установка инструментов мониторинга

```bash
sudo apt install -y sysstat htop iotop
```

### Шаг 2. Сбор статистики до и после оптимизации

**Скрипт для сбора метрик:**

```bash
cat > ~/collect_metrics.sh <<'EOF'
#!/bin/bash
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
LOGDIR=~/metrics_$TIMESTAMP
mkdir -p $LOGDIR

echo "=== СБОР МЕТРИК В $LOGDIR ==="

# 1. Параметры ядра
echo "=== SYSCTL ===" > $LOGDIR/sysctl.txt
sysctl fs.file-max vm.swappiness net.core.somaxconn kernel.shmmax fs.aio-max-nr >> $LOGDIR/sysctl.txt

# 2. Использование памяти
echo "=== MEMORY ===" > $LOGDIR/memory.txt
free -h >> $LOGDIR/memory.txt
vmstat 1 5 >> $LOGDIR/vmstat.txt

# 3. Использование диска
echo "=== DISK I/O ===" > $LOGDIR/disk.txt
iostat -x 1 3 >> $LOGDIR/disk.txt

# 4. Сетевые интерфейсы
echo "=== NETWORK ===" > $LOGDIR/network.txt
ip -s link >> $LOGDIR/network.txt

# 5. Активные соединения
echo "=== CONNECTIONS ===" > $LOGDIR/connections.txt
ss -tulpn >> $LOGDIR/connections.txt

echo "Метрики сохранены в $LOGDIR"
EOF

chmod +x ~/collect_metrics.sh
```

### Шаг 3. Сбор метрик до оптимизации

```bash
~/collect_metrics.sh
```

**Выполните оптимизации из Части 1 и/или Части 2, затем соберите метрики снова:**

```bash
~/collect_metrics.sh
```

### Шаг 4. Сравнение результатов

```bash
# Сравнение использования памяти
diff ~/metrics_*/memory.txt

# Сравнение параметров ядра
diff ~/metrics_*/sysctl.txt
```

### Шаг 5. Мониторинг в реальном времени

```bash
# Просмотр загрузки CPU и памяти
htop

# Мониторинг дискового I/O
sudo iotop -o

# Мониторинг сетевого трафика
sudo iftop -i $(ip route | grep default | awk '{print $5}')
```

---

## Часть 4. Управление профилями (20 минут)

### Шаг 1. Просмотр активных профилей

```bash
# Просмотр всех файлов в /etc/sysctl.d/
ls -la /etc/sysctl.d/

# Просмотр применённых параметров
sudo sysctl --system 2>&1 | grep -E "^(fs|vm|net|kernel)"
```

### Шаг 2. Переключение между профилями

```bash
# Отключение файлового профиля
sudo mv /etc/sysctl.d/99-file-server.conf /etc/sysctl.d/99-file-server.conf.disabled
sudo sysctl --system

# Проверка
sysctl vm.swappiness
```

```bash
# Включение профиля БД
sudo mv /etc/sysctl.d/99-file-server.conf.disabled /etc/sysctl.d/99-file-server.conf
sudo sysctl --system

# Проверка
sysctl vm.swappiness
```

### Шаг 3. Создание комбинированного профиля

Для серверов, совмещающих функции файлового обменника и БД, создайте объединённый профиль:

```bash
sudo tee /etc/sysctl.d/99-combined-server.conf > /dev/null <<'EOF'
# ============================================
# КОМБИНИРОВАННЫЙ ПРОФИЛЬ (ФАЙЛОВЫЙ СЕРВЕР + БД)
# ============================================

fs.file-max = 4194304
fs.aio-max-nr = 1048576
vm.swappiness = 20
vm.dirty_background_ratio = 5
vm.dirty_ratio = 15
vm.min_free_kbytes = 131072
net.core.somaxconn = 65535
net.core.netdev_max_backlog = 32768
net.core.rmem_default = 262144
net.core.wmem_default = 262144
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
kernel.shmmax = 68719476736
kernel.shmall = 4294967296
kernel.sem = 4096 2147483647 2147483646 512000
net.ipv4.tcp_window_scaling = 1
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_max_syn_backlog = 8192
net.ipv4.ip_local_port_range = 40000 65535
vm.overcommit_memory = 1
EOF

sudo sysctl --system
```

---

## Часть 5. Скрипт автоматической настройки (бонус)

Создайте универсальный скрипт `setup_kernel_tuning.sh`:

```bash
#!/bin/bash
# Универсальный скрипт настройки параметров ядра

echo "=== НАСТРОЙКА ПАРАМЕТРОВ ЯДРА ==="
echo "Выберите профиль:"
echo "  1) Файловый сервер"
echo "  2) Сервер баз данных"
echo "  3) Комбинированный"
echo "  4) Восстановить исходные настройки"
read -p "Ваш выбор (1-4): " CHOICE

case $CHOICE in
    1)
        echo "Применение профиля файлового сервера..."
        sudo cp /etc/sysctl.d/99-file-server.conf /etc/sysctl.d/99-active.conf 2>/dev/null || \
        echo "Файл 99-file-server.conf не найден. Сначала создайте профиль."
        ;;
    2)
        echo "Применение профиля сервера БД..."
        sudo cp /etc/sysctl.d/99-database-server.conf /etc/sysctl.d/99-active.conf 2>/dev/null || \
        echo "Файл 99-database-server.conf не найден. Сначала создайте профиль."
        ;;
    3)
        echo "Применение комбинированного профиля..."
        sudo tee /etc/sysctl.d/99-active.conf > /dev/null <<'EOF'
fs.file-max = 4194304
fs.aio-max-nr = 1048576
vm.swappiness = 20
vm.dirty_background_ratio = 5
vm.dirty_ratio = 15
vm.min_free_kbytes = 131072
net.core.somaxconn = 65535
net.core.netdev_max_backlog = 32768
net.core.rmem_default = 262144
net.core.wmem_default = 262144
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
kernel.shmmax = 68719476736
kernel.shmall = 4294967296
kernel.sem = 4096 2147483647 2147483646 512000
net.ipv4.tcp_window_scaling = 1
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_max_syn_backlog = 8192
net.ipv4.ip_local_port_range = 40000 65535
vm.overcommit_memory = 1
EOF
        ;;
    4)
        echo "Восстановление исходных настроек..."
        sudo rm -f /etc/sysctl.d/99-active.conf
        ;;
    *)
        echo "Неверный выбор"
        exit 1
        ;;
esac

sudo sysctl --system
echo "=== ГОТОВО ==="
```

