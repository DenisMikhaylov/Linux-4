# Лабораторная работа. Устранение неполадок менеджера пакетов в ALT Linux

**Продолжительность:** 2 академических часа

**Оборудование:** Виртуальная машина с ALT Linux (рабочая станция или сервер, например, ALT Workstation 10 или ALT Server 10)

---

## Цель работы

Научиться диагностировать и устранять различные неполадки, возникающие при работе с менеджером пакетов **APT** в дистрибутиве ALT Linux.

В ходе работы вы освоите:

- диагностику и восстановление повреждённой базы данных APT;
- устранение проблем с блокировками и «залипшими» процессами;
- решение проблем с зависимостями и конфликтами пакетов;
- восстановление менеджера пакетов при его повреждении;
- исправление проблем с репозиториями и ключами;
- проверку целостности установленных пакетов.

---

## Теоретическая справка

### Особенности управления пакетами в ALT Linux

ALT Linux использует **уникальную связку** – **APT** (Advanced Packaging Tool) поверх **RPM**-пакетов. Это означает, что:

- Пакеты распространяются в формате `.rpm`.
- Для управления пакетами используется утилита `apt-get`, знакомая пользователям Debian-подобных систем.
- APT автоматически определяет зависимости между пакетами и строго следит за их соблюдением.

### Архитектура системы управления пакетами в ALT Linux

| Уровень | Инструмент | Назначение | Ключевые файлы/каталоги |
|---------|------------|------------|-------------------------|
| **Низкоуровневый** | `rpm` | Непосредственная установка, удаление и управление отдельными `.rpm`-пакетами | `/var/lib/rpm/` — база данных RPM |
| **Высокоуровневый** | `apt-get` / `apt-cache` | Работа с репозиториями, разрешение зависимостей, обновление системы | `/var/lib/apt/lists/` — кеш списков пакетов<br>`/var/cache/apt/archives/` — кеш скачанных `.rpm`-файлов |

**Важно:** В ALT Linux для обновления системы используется команда `apt-get dist-upgrade`, а не `apt-get upgrade`.

### Критические файлы и их назначение

| Файл/каталог | Назначение | Что происходит при повреждении |
|--------------|------------|-------------------------------|
| `/var/lib/rpm/` | База данных RPM со списком установленных пакетов | Ошибки при выполнении любых операций с пакетами |
| `/var/lib/apt/lists/*` | Кеш списков пакетов из репозиториев | Ошибки `Failed to fetch`, `Hash Sum mismatch` |
| `/var/cache/apt/archives/*.rpm` | Скачанные, но ещё не установленные пакеты | Можно использовать для ручной установки |
| `/var/cache/apt/archives/lock` | Блокировка кеша APT | Ошибка при `apt-get install` |
| `/etc/apt/sources.list` и `/etc/apt/sources.list.d/` | Список репозиториев | Ошибки при `apt-get update` |

---

## Подготовка стенда

### Шаг 1. Настройка виртуальной машины

Создайте виртуальную машину с одним из следующих дистрибутивов ALT Linux:
- ALT Workstation 10
- ALT Server 10
- ALT Education 10

**Минимальные требования:**
- 2 ГБ оперативной памяти
- 20 ГБ дискового пространства
- Сетевое подключение для доступа к репозиториям

### Шаг 2. Установка необходимых пакетов

Подключитесь к виртуальной машине и выполните установку инструментов, необходимых для диагностики:

```bash
# Обновление информации о репозиториях
sudo apt-get update

# Установка диагностических инструментов
sudo apt-get install -y strace lsof nano wget curl rpm
```

**Пояснение пакетов:**
- `strace` — трассировка системных вызовов
- `lsof` — просмотр открытых файлов
- `nano` — текстовый редактор
- `rpm` — низкоуровневый менеджер пакетов (уже должен быть установлен)

### Шаг 3. Создание рабочей директории

```bash
mkdir -p ~/lab_apt_troubleshooting
cd ~/lab_apt_troubleshooting
```

### Шаг 4. Проверка текущего состояния системы

Выполните диагностическую команду для проверки целостности системы:

```bash
# Проверка наличия проблем с зависимостями
sudo apt-get check

# Просмотр установленных пакетов
rpm -qa | head -10
```

---

## Часть 1. Повреждение базы данных RPM (30 минут)

### Теоретическая справка

В ALT Linux база данных установленных пакетов хранится в `/var/lib/rpm/`. При её повреждении APT и RPM не могут определить, какие пакеты установлены в системе.

**Симптомы повреждения:**
```
error: rpmdb: BDB0113 Thread/process failed
error: cannot open Packages database in /var/lib/rpm
```

---

### Скрипт поломки

Создайте скрипт `break_rpmdb.sh`:

```bash
#!/bin/bash
# Скрипт повреждает базу данных RPM в ALT Linux

echo "=== ПОВРЕЖДЕНИЕ БАЗЫ ДАННЫХ RPM ==="

# 1. Создаём резервную копию базы данных
sudo mkdir -p /var/preserve
sudo tar -czf /var/preserve/rpmdb_backup_$(date +%Y%m%d_%H%M%S).tar.gz /var/lib/rpm/ 2>/dev/null
echo "Резервная копия создана в /var/preserve/"

# 2. Определяем тип бэкенда
BACKEND=$(rpm -E "%{_db_backend}" 2>/dev/null)
echo "Тип бэкенда: $BACKEND"

# 3. Повреждаем базу данных в зависимости от бэкенда
if [ "$BACKEND" = "sqlite" ]; then
    sudo dd if=/dev/urandom of=/var/lib/rpm/rpmdb.sqlite bs=512 count=10 conv=notrunc 2>/dev/null
    echo "Повреждён файл /var/lib/rpm/rpmdb.sqlite"
elif [ "$BACKEND" = "ndb" ]; then
    sudo dd if=/dev/urandom of=/var/lib/rpm/Packages.db bs=512 count=10 conv=notrunc 2>/dev/null
    echo "Повреждён файл /var/lib/rpm/Packages.db"
else
    # Для BDB
    sudo dd if=/dev/urandom of=/var/lib/rpm/Packages bs=512 count=10 conv=notrunc 2>/dev/null
    echo "Повреждён файл /var/lib/rpm/Packages"
fi

echo "=== БАЗА ДАННЫХ RPM ПОВРЕЖДЕНА ==="
echo "Попробуйте выполнить: rpm -qa"
```

Выполните скрипт:

```bash
chmod +x break_rpmdb.sh
./break_rpmdb.sh
```

---

### Диагностика

**Шаг 1. Проверка состояния базы данных**

Попробуйте выполнить простую команду для проверки базы данных:

```bash
rpm -qa
```

**Ожидаемый результат:** Ошибка, указывающая на повреждение базы данных.

**Шаг 2. Проверка работы APT**

Попробуйте выполнить команду APT:

```bash
sudo apt-get update
```

**Ожидаемый результат:** Ошибка при обращении к базе данных RPM.

**Шаг 3. Определение причины**

Проверьте, какие процессы используют файлы RPM:

```bash
sudo lsof | grep /var/lib/rpm
```

---

### Восстановление

**Шаг 1. Остановка процессов и удаление блокировок**

Если есть активные процессы, использующие базу данных RPM, остановите их:

```bash
# Просмотр процессов
sudo fuser -v /var/lib/rpm

# Удаление файлов блокировок
cd /var/lib/rpm
sudo rm -f __db*
```

**Шаг 2. Перестройка базы данных RPM**

Основная команда для восстановления базы данных:

```bash
sudo rpm --rebuilddb
```

Или, в зависимости от версии:

```bash
sudo rpmdb --rebuilddb
```

Эта команда перестраивает базу данных из заголовков установленных пакетов.

**Шаг 3. Проверка восстановления**

```bash
rpm -qa | head -10
sudo apt-get update
```

**Шаг 4. Если `rpm --rebuilddb` не помог**

Если база данных не восстанавливается стандартным способом:

```bash
# Удаление всех файлов базы данных
sudo rm -rf /var/lib/rpm/__db*
sudo rm -f /var/lib/rpm/Packages
sudo rm -f /var/lib/rpm/*.db

# Полная перестройка базы данных
sudo rpm --rebuilddb
```

---

## Часть 2. Устранение проблем с блокировками (15 минут)

### Теоретическая справка

APT и RPM используют блокировочные файлы для предотвращения одновременного запуска нескольких экземпляров. При аварийном завершении процесса блокировочные файлы могут остаться.

**Основные блокировочные файлы:**
- `/var/lib/rpm/__db*` — файлы блокировок BDB
- `/var/cache/apt/archives/lock` — блокировка кеша APT
- `/var/lib/apt/lists/lock` — блокировка списков APT

---

### Скрипт поломки

Создайте скрипт `break_lock.sh`:

```bash
#!/bin/bash
# Создание "залипшей" блокировки

echo "=== СОЗДАНИЕ ПРОБЛЕМЫ С БЛОКИРОВКОЙ ==="

# Создаём фальшивые блокировочные файлы
sudo touch /var/lib/rpm/__db.001
sudo touch /var/lib/rpm/__db.002
sudo chmod 644 /var/lib/rpm/__db.* 2>/dev/null

# Блокировка кеша APT
sudo touch /var/cache/apt/archives/lock
sudo chmod 644 /var/cache/apt/archives/lock

# Блокировка списков APT
sudo touch /var/lib/apt/lists/lock
sudo chmod 644 /var/lib/apt/lists/lock

echo "=== БЛОКИРОВОЧНЫЕ ФАЙЛЫ СОЗДАНЫ ==="
echo "При попытке использования apt-get возникнут ошибки."
```

Выполните скрипт:

```bash
chmod +x break_lock.sh
./break_lock.sh
```

---

### Диагностика

Попробуйте выполнить команду APT:

```bash
sudo apt-get update
```

**Ожидаемый результат:** Ошибка `Could not get lock` или подобная.

Проверьте, какие процессы используют файлы:

```bash
sudo lsof | grep -E "lock|__db"
```

---

### Восстановление

**Шаг 1. Проверка активных процессов**

```bash
ps aux | grep -E "apt|rpm|dpkg" | grep -v grep
```

Если есть активный процесс — дождитесь его завершения.

**Шаг 2. Удаление блокировочных файлов**

```bash
# Удаление блокировок RPM
cd /var/lib/rpm
sudo rm -f __db*

# Удаление блокировок APT
sudo rm -f /var/cache/apt/archives/lock
sudo rm -f /var/lib/apt/lists/lock
```

**Шаг 3. Проверка**

```bash
sudo apt-get update
```

---

## Часть 3. Проблемы с зависимостями и «битыми» пакетами (25 минут)

### Теоретическая справка

При установке или обновлении пакетов могут возникать ошибки зависимостей. APT имеет встроенный механизм автоматического исправления таких проблем.

---

### Скрипт поломки

Создайте скрипт `break_dependencies.sh`:

```bash
#!/bin/bash
# Создание проблем с зависимостями

echo "=== СОЗДАНИЕ ПРОБЛЕМ С ЗАВИСИМОСТЯМИ ==="

# 1. Скачиваем тестовый пакет с зависимостями
cd /tmp
wget -q http://ftp.altlinux.org/pub/distributions/ALTLinux/p10/branch/noarch/RPMS.classic/hello-2.10-alt2.noarch.rpm 2>/dev/null || \
wget -q http://ftp.altlinux.org/pub/distributions/ALTLinux/Sisyphus/noarch/RPMS.classic/hello-2.10-alt2.noarch.rpm 2>/dev/null || \
echo "Не удалось скачать пакет hello"

# 2. Попытка установки без зависимостей (создаст проблему)
if [ -f /tmp/hello-*.rpm ]; then
    sudo rpm -ivh --nodeps /tmp/hello-*.rpm 2>/dev/null || true
    echo "Пакет hello установлен с нарушением зависимостей"
fi

# 3. Создаём конфликт — удаляем важный файл базы данных
sudo rm -f /var/lib/rpm/__db.001 2>/dev/null || true

echo "=== ПРОБЛЕМЫ С ЗАВИСИМОСТЯМИ СОЗДАНЫ ==="
echo "Попробуйте выполнить: sudo apt-get install -f"
```

Выполните скрипт:

```bash
chmod +x break_dependencies.sh
./break_dependencies.sh
```

---

### Диагностика

Проверьте состояние зависимостей:

```bash
sudo apt-get check
```

**Ожидаемый результат:** Сообщение о нарушенных зависимостях.

Проверьте состояние пакетов:

```bash
rpm -qa | grep -E "(hello|broken)"
```

---

### Восстановление

**Шаг 1. Автоматическое исправление зависимостей**

Основная команда для исправления «битых» зависимостей:

```bash
sudo apt-get install -f
```

Или с полным синтаксисом:

```bash
sudo apt-get --fix-broken install
```

Эта команда пытается исправить нарушенные зависимости, устанавливая недостающие пакеты или удаляя конфликтующие.

**Шаг 2. Переустановка проблемного пакета**

Если известен проблемный пакет:

```bash
sudo apt-get install --reinstall hello
```

**Шаг 3. Принудительное удаление**

Если пакет не поддаётся восстановлению:

```bash
sudo rpm -e --nodeps hello
sudo apt-get install -f
```

**Шаг 4. Проверка**

```bash
sudo apt-get check
```

---

## Часть 4. Повреждение кеша и списков репозиториев (20 минут)

### Теоретическая справка

Кеш APT (`/var/cache/apt/archives/`) содержит скачанные пакеты. Списки репозиториев (`/var/lib/apt/lists/`) содержат информацию о доступных пакетах. Их повреждение может привести к ошибкам при `apt-get update`.

---

### Скрипт поломки

Создайте скрипт `break_cache.sh`:

```bash
#!/bin/bash
# Повреждение кеша и списков APT

echo "=== ПОВРЕЖДЕНИЕ КЕША И СПИСКОВ APT ==="

# 1. Повреждаем кеш пакетов
sudo dd if=/dev/urandom of=/var/cache/apt/archives/partial/test bs=1K count=1 2>/dev/null || true

# 2. Удаляем списки репозиториев
sudo rm -rf /var/lib/apt/lists/* 2>/dev/null

# 3. Создаём повреждённый файл кеша
echo "CORRUPTED" | sudo tee /var/cache/apt/pkgcache.bin 2>/dev/null

echo "=== КЕШ И СПИСКИ APT ПОВРЕЖДЕНЫ ==="
echo "При попытке apt-get update возникнут ошибки."
```

Выполните скрипт:

```bash
chmod +x break_cache.sh
./break_cache.sh
```

---

### Диагностика

Попробуйте обновить списки пакетов:

```bash
sudo apt-get update
```

**Ожидаемый результат:** Ошибки `Failed to fetch` или `Hash Sum mismatch`.

---

### Восстановление

**Шаг 1. Очистка кеша**

```bash
sudo apt-get clean
```

Эта команда удаляет все скачанные пакеты из кеша.

**Шаг 2. Удаление повреждённых списков**

```bash
sudo rm -rf /var/lib/apt/lists/*
```

**Шаг 3. Обновление списков пакетов**

```bash
sudo apt-get update
```

**Шаг 4. Если ошибки сохраняются**

Проверьте файлы репозиториев:

```bash
cat /etc/apt/sources.list
ls -la /etc/apt/sources.list.d/
```

При необходимости исправьте или удалите проблемные репозитории.

---

## Часть 5. Восстановление после прерванного обновления (20 минут)

### Теоретическая справка

Если обновление системы было прервано (например, при выключении питания или закрытии терминала), пакеты могут остаться в нестабильном состоянии.

---

### Скрипт поломки

Создайте скрипт `break_upgrade.sh`:

```bash
#!/bin/bash
# Имитация последствий прерванного обновления

echo "=== ИМИТАЦИЯ ПРЕРВАННОГО ОБНОВЛЕНИЯ ==="

# 1. Создаём ситуацию с "зависшим" пакетом
sudo touch /var/lib/rpm/__db.001
sudo chmod 000 /var/lib/rpm/__db.001 2>/dev/null || true

# 2. Удаляем скрипт настройки пакета (если есть)
sudo rm -f /var/lib/rpm/__db.002 2>/dev/null || true

echo "=== ПРЕРВАННОЕ ОБНОВЛЕНИЕ ИМИТИРОВАНО ==="
echo "Система может сообщать о нестабильных пакетах."
```

Выполните скрипт:

```bash
chmod +x break_upgrade.sh
./break_upgrade.sh
```

---

### Диагностика

Проверьте состояние системы:

```bash
sudo apt-get check
rpm -qa | wc -l
```

---

### Восстановление

**Шаг 1. Удаление блокировок**

```bash
sudo rm -f /var/lib/rpm/__db*
sudo rm -f /var/cache/apt/archives/lock
sudo rm -f /var/lib/apt/lists/lock
```

**Шаг 2. Перестройка базы данных**

```bash
sudo rpm --rebuilddb
```

**Шаг 3. Исправление зависимостей**

```bash
sudo apt-get install -f
```

**Шаг 4. Полное обновление системы**

```bash
sudo apt-get update
sudo apt-get dist-upgrade
```

> ⚠️ **Важно:** В ALT Linux для обновления системы используется `apt-get dist-upgrade`, а не `apt-get upgrade`!

---

## Часть 6. Проблемы с репозиториями (15 минут)

### Теоретическая справка

Проблемы с репозиториями могут возникать из-за недоступных зеркал, неправильных записей в `/etc/apt/sources.list` или отсутствующих ключей.

---

### Скрипт поломки

Создайте скрипт `break_repo.sh`:

```bash
#!/bin/bash
# Создание проблем с репозиториями

echo "=== СОЗДАНИЕ ПРОБЛЕМ С РЕПОЗИТОРИЯМИ ==="

# 1. Добавляем несуществующий репозиторий
echo "deb http://nonexistent-repo.example.com/altlinux p10 main" | sudo tee /etc/apt/sources.list.d/fake-repo.list

# 2. Повреждаем основной файл репозиториев
sudo cp /etc/apt/sources.list /etc/apt/sources.list.backup
echo "# Повреждённый файл" | sudo tee /etc/apt/sources.list

echo "=== ПРОБЛЕМЫ С РЕПОЗИТОРИЯМИ СОЗДАНЫ ==="
echo "При apt-get update возникнут ошибки."
```

Выполните скрипт:

```bash
chmod +x break_repo.sh
./break_repo.sh
```

---

### Диагностика

Попробуйте обновить списки:

```bash
sudo apt-get update
```

**Ожидаемый результат:** Ошибки `Failed to fetch` или `Unable to connect`.

---

### Восстановление

**Шаг 1. Удаление проблемных репозиториев**

```bash
sudo rm /etc/apt/sources.list.d/fake-repo.list
```

**Шаг 2. Восстановление основного файла**

```bash
sudo cp /etc/apt/sources.list.backup /etc/apt/sources.list
```

Если резервной копии нет, создайте базовый файл для ALT Linux:

```bash
echo "rpm http://ftp.altlinux.org/pub/distributions/ALTLinux p10 branch classic" | sudo tee /etc/apt/sources.list
```

**Шаг 3. Обновление**

```bash
sudo apt-get update
```

---

## Часть 7. Скрипт общего восстановления (бонус)

Создайте универсальный скрипт `fix_apt.sh`:

```bash
#!/bin/bash
# Универсальный скрипт восстановления APT в ALT Linux

echo "=== УНИВЕРСАЛЬНОЕ ВОССТАНОВЛЕНИЕ APT ==="

# 1. Создание резервной копии
echo "[1/8] Создание резервной копии базы данных RPM..."
sudo mkdir -p /var/preserve
sudo tar -czf /var/preserve/rpmdb_$(date +%Y%m%d_%H%M%S).tar.gz /var/lib/rpm/ 2>/dev/null

# 2. Проверка активных процессов
echo "[2/8] Проверка активных процессов..."
sudo fuser -v /var/lib/rpm 2>/dev/null

# 3. Удаление блокировок
echo "[3/8] Удаление блокировочных файлов..."
cd /var/lib/rpm
sudo rm -f __db* 2>/dev/null
sudo rm -f /var/cache/apt/archives/lock 2>/dev/null
sudo rm -f /var/lib/apt/lists/lock 2>/dev/null

# 4. Перестройка базы данных RPM
echo "[4/8] Перестройка базы данных RPM..."
sudo rpm --rebuilddb 2>/dev/null || sudo rpmdb --rebuilddb 2>/dev/null

# 5. Исправление зависимостей
echo "[5/8] Исправление зависимостей..."
sudo apt-get install -f -y 2>/dev/null

# 6. Очистка кеша
echo "[6/8] Очистка кеша..."
sudo apt-get clean 2>/dev/null

# 7. Удаление списков и обновление
echo "[7/8] Обновление списков репозиториев..."
sudo rm -rf /var/lib/apt/lists/* 2>/dev/null
sudo apt-get update 2>/dev/null

# 8. Проверка
echo "[8/8] Проверка состояния..."
sudo apt-get check 2>/dev/null

echo "=== ВОССТАНОВЛЕНИЕ ЗАВЕРШЕНО ==="
echo "Для проверки выполните: rpm -qa | head -10"
```

Сделайте скрипт исполняемым:

```bash
chmod +x fix_apt.sh
./fix_apt.sh
```
