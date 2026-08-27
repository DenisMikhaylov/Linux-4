# Лабораторная работа. Устранение неполадок менеджера пакетов

**Продолжительность:** 2 академических часа

**Оборудование:** Виртуальная машина с Debian 11/12 или Ubuntu 20.04/22.04

---

## Цель работы

Научиться диагностировать и устранять различные неполадки, возникающие при работе с менеджером пакетов APT в системах на основе Debian. В ходе работы вы освоите:

- диагностику и восстановление повреждённой базы данных dpkg;
- устранение проблем с зависимостями и «битыми» пакетами;
- удаление «залипших» блокировочных файлов;
- очистку и восстановление кеша APT;
- восстановление после прерванного обновления;
- диагностику проблем с репозиториями и ключами.

---

## Теоретическая справка

**APT** (Advanced Package Tool) — система управления пакетами, используемая в Debian, Ubuntu и их производных. Она работает поверх **dpkg** — низкоуровневого инструмента для установки, удаления и управления пакетами `.deb`.

**Основные компоненты системы управления пакетами:**

| Компонент | Путь | Назначение |
|-----------|------|------------|
| База данных dpkg | `/var/lib/dpkg/status` | Список установленных пакетов и их состояний |
| Информация о пакетах | `/var/lib/dpkg/info/` | Скрипты управления, списки файлов |
| Кеш APT | `/var/cache/apt/archives/` | Скачанные пакеты |
| Списки репозиториев | `/var/lib/apt/lists/` | Информация о доступных пакетах |
| Блокировочные файлы | `/var/lib/dpkg/lock`, `/var/lib/apt/lists/lock` | Предотвращают одновременный запуск |

**Типичные проблемы и их причины:**

| Проблема | Причина |
|----------|---------|
| `dpkg: error: parsing file '/var/lib/dpkg/status'` | Повреждение базы данных dpkg |
| `E: Unmet dependencies` | Нарушенные зависимости между пакетами |
| `Could not get lock /var/lib/dpkg/lock` | Другой процесс использует менеджер пакетов |
| `Failed to fetch ... Hash Sum mismatch` | Проблемы с кешем или зеркалом репозитория |
| `E: Sub-process /usr/bin/dpkg returned an error code (1)` | Ошибка при выполнении dpkg |
| `W: GPG error: ... NO_PUBKEY` | Отсутствует ключ репозитория |

---

## Подготовка стенда

Убедитесь, что на вашей виртуальной машине установлены необходимые пакеты:

```bash
sudo apt update
sudo apt install -y curl wget gnupg software-properties-common
```

Создайте рабочую директорию для лабораторной работы:

```bash
mkdir -p ~/lab_package_troubleshooting
cd ~/lab_package_troubleshooting
```

---

## Часть 1. Повреждение базы данных dpkg (25 минут)

### Теоретическая справка

Файл `/var/lib/dpkg/status` содержит информацию обо всех установленных пакетах. При его повреждении dpkg не может определить, какие пакеты установлены, что приводит к ошибкам при установке, удалении или обновлении пакетов.

---

### Скрипт поломки

Создайте скрипт `break_dpkg_db.sh`:

```bash
#!/bin/bash
# Повреждение базы данных dpkg

echo "=== ПОВРЕЖДЕНИЕ БАЗЫ ДАННЫХ DPKG ==="

# Создаём резервную копию
sudo cp /var/lib/dpkg/status /var/lib/dpkg/status.backup
echo "Резервная копия сохранена"

# Повреждаем файл status — удаляем несколько строк в середине
sudo head -n 100 /var/lib/dpkg/status > /tmp/status_good
echo "!!! ПОВРЕЖДЕНИЕ ВНЕСЕНО !!!" | sudo tee -a /tmp/status_good
sudo tail -n +200 /var/lib/dpkg/status >> /tmp/status_good
sudo mv /tmp/status_good /var/lib/dpkg/status

echo "=== БАЗА ДАННЫХ DPKG ПОВРЕЖДЕНА ==="
echo "При попытке использования apt или dpkg возникнут ошибки."
```

Выполните скрипт:

```bash
chmod +x break_dpkg_db.sh
./break_dpkg_db.sh
```

---

### Диагностика

Попробуйте выполнить простую команду:

```bash
sudo apt update
```

**Ожидаемый результат:** Ошибка, связанная с парсингом файла `/var/lib/dpkg/status`.

Проверьте статус dpkg:

```bash
sudo dpkg --list
```

**Ожидаемый результат:** Ошибка `dpkg: error: parsing file '/var/lib/dpkg/status'`.

---

### Восстановление

**Шаг 1. Восстановление из резервной копии**

Если вы создали резервную копию:

```bash
sudo cp /var/lib/dpkg/status.backup /var/lib/dpkg/status
```

**Шаг 2. Если резервной копии нет — восстановление из архивов**

В системе хранятся резервные копии состояния dpkg:

```bash
# Просмотр доступных резервных копий
ls -la /var/backups/dpkg.status*

# Восстановление из последней резервной копии
sudo cp /var/backups/dpkg.status.0 /var/lib/dpkg/status
```

**Шаг 3. Переконфигурация dpkg**

После восстановления базы данных выполните:

```bash
sudo dpkg --configure -a
```

Эта команда настраивает все пакеты, которые были распакованы, но не настроены.

**Шаг 4. Проверка**

```bash
sudo apt update
sudo dpkg --list | head -10
```

---

## Часть 2. «Битые» пакеты и проблемы с зависимостями (25 минут)

### Теоретическая справка

При прерывании установки пакета или при конфликте зависимостей пакет может остаться в состоянии «битого» (broken). APT имеет встроенный механизм автоматического исправления таких проблем.

---

### Скрипт поломки

Создайте скрипт `break_dependencies.sh`:

```bash
#!/bin/bash
# Создание ситуации с "битыми" зависимостями

echo "=== СОЗДАНИЕ ПРОБЛЕМ С ЗАВИСИМОСТЯМИ ==="

# 1. Скачиваем тестовый пакет
wget -O /tmp/test_package.deb http://ftp.ru.debian.org/debian/pool/main/h/hello/hello_2.10-2_amd64.deb 2>/dev/null || \
wget -O /tmp/test_package.deb http://ftp.debian.org/debian/pool/main/h/hello/hello_2.10-2_amd64.deb 2>/dev/null

# 2. Имитация прерванной установки — удаляем файл настроек пакета,
#    оставляя пакет в состоянии "распакован, но не настроен"
sudo dpkg -i /tmp/test_package.deb 2>/dev/null || true

# 3. Создаём "битый" пакет — удаляем информацию о нём из базы, но оставляем файлы
sudo dpkg --remove hello 2>/dev/null || true

# 4. Создаём конфликт зависимостей — устанавливаем пакет с несуществующей зависимостью
#    Создаём фальшивый пакет с зависимостью от несуществующего пакета
cat > /tmp/fake_package/DEBIAN/control <<EOF
Package: fake-package
Version: 1.0-1
Section: misc
Priority: optional
Architecture: all
Depends: non-existent-package (>= 1.0)
Description: Fake package with broken dependency
EOF

mkdir -p /tmp/fake_package/DEBIAN
dpkg-deb --build /tmp/fake_package /tmp/fake-package.deb 2>/dev/null

# 5. Пытаемся установить фальшивый пакет
sudo dpkg -i /tmp/fake-package.deb 2>/dev/null || true

echo "=== ПРОБЛЕМЫ С ЗАВИСИМОСТЯМИ СОЗДАНЫ ==="
echo "Попробуйте выполнить: sudo apt install -f"
```

Выполните скрипт:

```bash
chmod +x break_dependencies.sh
./break_dependencies.sh
```

---

### Диагностика

Попробуйте установить или удалить любой пакет:

```bash
sudo apt install curl
```

**Ожидаемый результат:** Сообщение о неудовлетворённых зависимостях.

Проверьте состояние пакетов:

```bash
dpkg -l | grep -E "^[^i]"
```

Эта команда покажет пакеты, которые не находятся в состоянии `ii` (установлены и настроены).

---

### Восстановление

**Шаг 1. Автоматическое исправление зависимостей**

```bash
sudo apt --fix-broken install
```

или

```bash
sudo apt install -f
```

Эта команда пытается исправить нарушенные зависимости, устанавливая недостающие пакеты или удаляя конфликтующие.

**Шаг 2. Переконфигурация всех пакетов**

Если первый шаг не помог:

```bash
sudo dpkg --configure -a
```

**Шаг 3. Принудительная установка конкретного пакета**

Если известен проблемный пакет:

```bash
sudo apt install --reinstall hello
```

**Шаг 4. Удаление проблемного пакета**

Если пакет не поддаётся восстановлению:

```bash
sudo dpkg --remove --force-remove-reinstreq fake-package
sudo apt --fix-broken install
```

**Шаг 5. Проверка**

```bash
dpkg -l | grep -E "^[^i]"
```

Все пакеты должны быть в состоянии `ii`.

---

## Часть 3. Блокировочные файлы (15 минут)

### Теоретическая справка

APT и dpkg используют блокировочные файлы, чтобы предотвратить одновременный запуск нескольких экземпляров менеджера пакетов. Если процесс был прерван, блокировочный файл может остаться, что заблокирует дальнейшую работу.

**Основные блокировочные файлы:**
- `/var/lib/dpkg/lock`
- `/var/lib/dpkg/lock-frontend`
- `/var/lib/apt/lists/lock`
- `/var/cache/apt/archives/lock`

---

### Скрипт поломки

Создайте скрипт `break_lock.sh`:

```bash
#!/bin/bash
# Создание "залипшей" блокировки

echo "=== СОЗДАНИЕ ПРОБЛЕМЫ С БЛОКИРОВКОЙ ==="

# Создаём фальшивые блокировочные файлы
sudo touch /var/lib/dpkg/lock
sudo touch /var/lib/dpkg/lock-frontend
sudo touch /var/lib/apt/lists/lock
sudo touch /var/cache/apt/archives/lock

# Устанавливаем права, чтобы блокировка выглядела "активной"
sudo chmod 644 /var/lib/dpkg/lock
sudo chmod 644 /var/lib/dpkg/lock-frontend
sudo chmod 644 /var/lib/apt/lists/lock
sudo chmod 644 /var/cache/apt/archives/lock

echo "=== БЛОКИРОВОЧНЫЕ ФАЙЛЫ СОЗДАНЫ ==="
echo "При попытке использования apt возникнет ошибка:"
echo "  Could not get lock /var/lib/dpkg/lock-frontend"
```

Выполните скрипт:

```bash
chmod +x break_lock.sh
./break_lock.sh
```

---

### Диагностика

Попробуйте выполнить любую команду APT:

```bash
sudo apt update
```

**Ожидаемый результат:** Ошибка `Could not get lock /var/lib/dpkg/lock-frontend`

Проверьте, какие процессы используют блокировки:

```bash
sudo lsof /var/lib/dpkg/lock
sudo lsof /var/lib/dpkg/lock-frontend
```

Или используйте `fuser`:

```bash
sudo fuser -v /var/lib/dpkg/lock
```

---

### Восстановление

**Шаг 1. Проверка, не запущен ли другой процесс**

```bash
ps aux | grep -E "apt|dpkg" | grep -v grep
```

Если есть активный процесс — дождитесь его завершения.

**Шаг 2. Удаление блокировочных файлов (если процессы отсутствуют)**

```bash
sudo rm /var/lib/dpkg/lock
sudo rm /var/lib/dpkg/lock-frontend
sudo rm /var/lib/apt/lists/lock
sudo rm /var/cache/apt/archives/lock
```



**Шаг 3. Переконфигурация dpkg**

```bash
sudo dpkg --configure -a
```

**Шаг 4. Проверка**

```bash
sudo apt update
```

---

## Часть 4. Проблемы с кешем и списками репозиториев (20 минут)

### Теоретическая справка

Кеш APT (`/var/cache/apt/archives/`) содержит скачанные пакеты. Списки репозиториев (`/var/lib/apt/lists/`) содержат информацию о доступных пакетах. Их повреждение может привести к ошибкам `Hash Sum mismatch` или `Failed to fetch`.

---

### Скрипт поломки

Создайте скрипт `break_cache.sh`:

```bash
#!/bin/bash
# Повреждение кеша и списков APT

echo "=== ПОВРЕЖДЕНИЕ КЕША И СПИСКОВ APT ==="

# 1. Повреждаем кеш пакетов — записываем мусор в файлы
sudo dd if=/dev/urandom of=/var/cache/apt/archives/partial/test bs=1K count=1 2>/dev/null || true

# 2. Повреждаем списки репозиториев
sudo rm -rf /var/lib/apt/lists/* 2>/dev/null

# 3. Создаём повреждённый файл кеша
echo "CORRUPTED" | sudo tee /var/cache/apt/pkgcache.bin 2>/dev/null

echo "=== КЕШ И СПИСКИ APT ПОВРЕЖДЕНЫ ==="
echo "При попытке apt update возникнут ошибки."
echo "При попытке установки пакетов — ошибки Hash Sum mismatch."
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
sudo apt update
```

**Ожидаемый результат:** Ошибки `Failed to fetch` или `Hash Sum mismatch`.

Проверьте состояние кеша:

```bash
ls -la /var/cache/apt/archives/
ls -la /var/lib/apt/lists/
```

---

### Восстановление

**Шаг 1. Очистка кеша**

```bash
sudo apt clean
```

Эта команда удаляет все скачанные пакеты из кеша.

**Шаг 2. Удаление повреждённых списков**

```bash
sudo rm -rf /var/lib/apt/lists/*
```

**Шаг 3. Удаление повреждённого кеша (если остался)**

```bash
sudo rm -f /var/cache/apt/pkgcache.bin
sudo rm -f /var/cache/apt/srcpkgcache.bin
```

**Шаг 4. Обновление списков пакетов**

```bash
sudo apt update
```

**Шаг 5. Принудительное исправление пропущенных файлов (если необходимо)**

```bash
sudo apt update --fix-missing
```



---

## Часть 5. Восстановление после прерванного обновления (20 минут)

### Теоретическая справка

Если обновление системы было прервано (например, при выключении питания или закрытии терминала), пакеты могут остаться в нестабильном состоянии. Это одна из самых сложных ситуаций для восстановления.

---

### Скрипт поломки

Создайте скрипт `break_upgrade.sh`:

```bash
#!/bin/bash
# Имитация последствий прерванного обновления

echo "=== ИМИТАЦИЯ ПРЕРВАННОГО ОБНОВЛЕНИЯ ==="

# 1. Создаём ситуацию с "зависшим" пакетом
#    Удаляем скрипт настройки пакета, оставляя его в состоянии "unpacked"
sudo mkdir -p /var/lib/dpkg/info
sudo touch /var/lib/dpkg/info/hello.postinst
sudo chmod 000 /var/lib/dpkg/info/hello.postinst 2>/dev/null || true

# 2. Помечаем несколько пакетов как "требующие переустановки"
sudo dpkg --force-depends --remove hello 2>/dev/null || true
sudo dpkg --force-depends --purge hello 2>/dev/null || true

# 3. Создаём флаг "прерванного обновления"
echo "triggers-pending" | sudo tee /var/lib/dpkg/triggers/Unincorp 2>/dev/null || true

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

Проверьте статус пакетов:

```bash
dpkg -l | grep -E "^[^i]"
```

Проверьте наличие незавершённых операций:

```bash
sudo dpkg --audit
```

---

### Восстановление

**Шаг 1. Переконфигурация всех пакетов**

```bash
sudo dpkg --configure -a
```

**Шаг 2. Исправление битых зависимостей**

```bash
sudo apt --fix-broken install
```

**Шаг 3. Очистка и принудительное обновление**

```bash
sudo apt clean
sudo apt update --fix-missing
sudo apt dist-upgrade -f
```

**Шаг 4. Переустановка проблемных пакетов**

Если известны проблемные пакеты:

```bash
sudo apt install --reinstall hello
```

**Шаг 5. Принудительное завершение (крайний случай)**

Если ничего не помогает, можно удалить проблемный пакет и установить заново:

```bash
sudo dpkg --remove --force-remove-reinstreq hello
sudo apt install hello
```

---

## Часть 6. Проблемы с репозиториями и ключами (15 минут)

### Теоретическая справка

APT проверяет подлинность пакетов с помощью GPG-ключей. Если ключ репозитория отсутствует или устарел, возникает ошибка `NO_PUBKEY` или `EXPKEYSIG`.

---

### Скрипт поломки

Создайте скрипт `break_repo.sh`:

```bash
#!/bin/bash
# Создание проблем с репозиториями

echo "=== СОЗДАНИЕ ПРОБЛЕМ С РЕПОЗИТОРИЯМИ ==="

# 1. Добавляем репозиторий без ключа
echo "deb http://archive.ubuntu.com/ubuntu/ focal universe" | sudo tee /etc/apt/sources.list.d/bad-repo.list

# 2. Создаём повреждённый файл sources.list
sudo chmod 000 /etc/apt/sources.list.d/bad-repo.list

# 3. Добавляем репозиторий с неправильным URL
echo "deb http://nonexistent-repo.example.com/ubuntu focal main" | sudo tee /etc/apt/sources.list.d/fake-repo.list

echo "=== ПРОБЛЕМЫ С РЕПОЗИТОРИЯМИ СОЗДАНЫ ==="
echo "При apt update возникнут ошибки:"
echo "  - NO_PUBKEY"
echo "  - Failed to fetch"
echo "  - Unable to lock directory"
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
sudo apt update
```

**Ожидаемый результат:** Ошибки `NO_PUBKEY`, `Failed to fetch` или `Unable to lock directory`.

Проверьте файлы репозиториев:

```bash
ls -la /etc/apt/sources.list.d/
cat /etc/apt/sources.list.d/bad-repo.list
```

---

### Восстановление

**Шаг 1. Удаление проблемных репозиториев**

```bash
sudo rm /etc/apt/sources.list.d/bad-repo.list
sudo rm /etc/apt/sources.list.d/fake-repo.list
```

**Шаг 2. Восстановление прав на файлы (если были изменены)**

```bash
sudo chmod 644 /etc/apt/sources.list.d/*.list 2>/dev/null || true
```

**Шаг 3. Обновление ключей**

Если есть ошибка `NO_PUBKEY`:

```bash
# Получение недостающего ключа (пример)
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys <PUBKEY>
```

Или используя `wget`:

```bash
wget -qO- https://example.com/repo.gpg | sudo apt-key add -
```

**Шаг 4. Проверка**

```bash
sudo apt update
```

---

## Часть 7. Скрипт общего восстановления (бонус)

Создайте универсальный скрипт `fix_package_manager.sh`, который выполняет все основные шаги восстановления:

```bash
#!/bin/bash
# Универсальный скрипт восстановления менеджера пакетов

echo "=== УНИВЕРСАЛЬНОЕ ВОССТАНОВЛЕНИЕ МЕНЕДЖЕРА ПАКЕТОВ ==="

# 1. Остановка возможных процессов
echo "[1/8] Проверка активных процессов..."
sudo killall apt apt-get dpkg 2>/dev/null || true

# 2. Удаление блокировочных файлов
echo "[2/8] Удаление блокировочных файлов..."
sudo rm -f /var/lib/dpkg/lock
sudo rm -f /var/lib/dpkg/lock-frontend
sudo rm -f /var/lib/apt/lists/lock
sudo rm -f /var/cache/apt/archives/lock

# 3. Переконфигурация dpkg
echo "[3/8] Переконфигурация dpkg..."
sudo dpkg --configure -a

# 4. Исправление зависимостей
echo "[4/8] Исправление зависимостей..."
sudo apt --fix-broken install -y

# 5. Очистка кеша
echo "[5/8] Очистка кеша..."
sudo apt clean
sudo apt autoclean

# 6. Удаление повреждённых списков
echo "[6/8] Удаление повреждённых списков..."
sudo rm -rf /var/lib/apt/lists/*
sudo rm -f /var/cache/apt/pkgcache.bin
sudo rm -f /var/cache/apt/srcpkgcache.bin

# 7. Обновление списков
echo "[7/8] Обновление списков пакетов..."
sudo apt update --fix-missing

# 8. Удаление ненужных пакетов
echo "[8/8] Удаление ненужных пакетов..."
sudo apt autoremove -y
sudo apt autoclean

echo "=== ВОССТАНОВЛЕНИЕ ЗАВЕРШЕНО ==="
```

Сделайте скрипт исполняемым:

```bash
chmod +x fix_package_manager.sh
```


