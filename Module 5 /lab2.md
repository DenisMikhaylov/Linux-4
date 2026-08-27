# Лабораторные работы по диагностике и восстановлению систем хранения данных с LVM

---

## Подготовка стенда

### 1. Добавление четырёх дисков

В настройках виртуальной машины добавьте **4 дополнительных виртуальных диска** (например, по 2 ГБ). В системе они появятся как:

- `/dev/sdb`
- `/dev/sdc`
- `/dev/sdd`
- `/dev/sde`

### 2. Установка необходимых пакетов

```bash
sudo apt update
sudo apt install -y smartmontools util-linux lvm2 btrfs-progs testdisk extundelete gdisk
```

---

# Лабораторная работа №1. Диагностика устройств хранения

**Цель:** Научиться выявлять проблемы с накопителями с помощью SMART, анализа журналов и сканирования поверхности.

**Продолжительность:** 1,5–2 часа.

---

## Теоретическая справка

- **SMART** (Self-Monitoring, Analysis and Reporting Technology) — система самодиагностики накопителей.
- **Ключевые SMART-атрибуты:**
  - `Reallocated_Sector_Ct` (ID 5) — переназначенные сектора (любое ненулевое значение — тревога)
  - `Current_Pending_Sector` (ID 197) — секторы, ожидающие переназначения
  - `Offline_Uncorrectable` (ID 198) — невосстановимые секторы
- **badblocks** — утилита для сканирования поверхности диска на наличие повреждённых блоков.

---

## Скрипт поломки

Создайте скрипт `break_storage.sh`, который **имитирует** проблемы с диском:

```bash
#!/bin/bash
# Скрипт создаёт тестовый диск с "повреждёнными" блоками

echo "=== СОЗДАНИЕ ТЕСТОВОГО ДИСКА С ПРОБЛЕМАМИ ==="

# 1. Создаём файл-образ для тестового диска
dd if=/dev/zero of=/tmp/test_disk.img bs=1M count=200
LOOP=$(sudo losetup -f --show /tmp/test_disk.img)
echo "Создано loop-устройство: $LOOP"

# 2. Создаём файловую систему
sudo mkfs.ext4 -F $LOOP

# 3. Монтируем и создаём тестовые файлы
sudo mkdir -p /mnt/test_disk
sudo mount $LOOP /mnt/test_disk
echo "Тестовые данные" | sudo tee /mnt/test_disk/testfile.txt
sudo dd if=/dev/urandom of=/mnt/test_disk/data.bin bs=1M count=50

# 4. Имитация повреждённых блоков (принудительная запись в "плохие" места)
sudo dd if=/dev/urandom of=$LOOP bs=512 seek=1000 count=10 conv=notrunc 2>/dev/null

# 5. Размонтируем
sudo umount /mnt/test_disk

echo "=== ТЕСТОВЫЙ ДИСК С ПРОБЛЕМАМИ СОЗДАН ==="
echo "Устройство: $LOOP"
echo "Для диагностики используйте:"
echo "  sudo smartctl -a $LOOP"
echo "  sudo badblocks -sv $LOOP"
```

Выполните скрипт:

```bash
chmod +x break_storage.sh
./break_storage.sh
```

---

## Пошаговая диагностика

### Шаг 1. Анализ системных журналов

```bash
# Просмотр ошибок в журнале ядра
dmesg -T | grep -iE "i/o error|read error|write error|sector|bad block"

# Просмотр системного журнала
sudo journalctl -xb | grep -iE "i/o error|ata|scsi|sd"
```

**Ожидаемый результат:** Сообщения, указывающие на проблемы с чтением/записью.

---

### Шаг 2. Проверка SMART-информации

Определите устройство:

```bash
sudo losetup -l
```

Получите SMART-информацию:

```bash
# Для реального диска:
sudo smartctl -a /dev/sdb

# Для тестового loop-устройства (если SMART не поддерживается):
sudo smartctl -a /dev/loop0 2>/dev/null || echo "SMART не поддерживается"
```

Проверьте общее состояние:

```bash
sudo smartctl -H /dev/sdb
```

---

### Шаг 3. Запуск тестов SMART

```bash
# Короткий тест
sudo smartctl -t short /dev/sdb

# Просмотр результатов (через 2-3 минуты)
sudo smartctl -l selftest /dev/sdb
```

---

### Шаг 4. Сканирование поверхности (badblocks)

Размонтируйте раздел (если смонтирован):

```bash
sudo umount /dev/sdb1 2>/dev/null
```

Запустите **недеструктивное** сканирование:

```bash
sudo badblocks -sv /dev/sdb1
```

---

### Шаг 5. Проверка файловой системы

```bash
sudo fsck -n /dev/sdb1
```

---

## Восстановление

Если badblocks нашёл повреждённые блоки, их можно изолировать:

```bash
# Получение списка плохих блоков
sudo badblocks -sv /dev/sdb1 > /tmp/badblocks.txt

# Создание файла с плохими блоками для e2fsck
sudo e2fsck -l /tmp/badblocks.txt /dev/sdb1
```

---

# Лабораторная работа №2. Проверка диска и обслуживание файловых систем

**Цель:** Научиться проверять целостность файловых систем, исправлять ошибки и выполнять профилактическое обслуживание.

**Продолжительность:** 1,5–2 часа.

---

## Теоретическая справка

- **fsck** (File System Check) — основная утилита для проверки и восстановления файловых систем.
- **e2fsck** — версия fsck для файловых систем ext2/ext3/ext4.
- **Резервные суперблоки** в ext4 хранятся в позициях 32768, 98304, 163840 и др.

---

## Скрипт поломки

Создайте скрипт `break_filesystem.sh`:

```bash
#!/bin/bash
# Скрипт повреждает файловую систему на тестовом разделе

echo "=== ПОВРЕЖДЕНИЕ ФАЙЛОВОЙ СИСТЕМЫ ==="

# 1. Создаём тестовый образ
dd if=/dev/zero of=/tmp/fs_test.img bs=1M count=100
LOOP=$(sudo losetup -f --show /tmp/fs_test.img)
echo "Создано устройство: $LOOP"

# 2. Создаём файловую систему ext4
sudo mkfs.ext4 -F $LOOP

# 3. Монтируем и создаём файлы
sudo mkdir -p /mnt/fs_test
sudo mount $LOOP /mnt/fs_test
for i in {1..20}; do
    echo "File $i" | sudo tee /mnt/fs_test/file_$i.txt > /dev/null
done
sudo umount /mnt/fs_test

# 4. Повреждаем суперблок (записываем случайные данные в начало)
sudo dd if=/dev/urandom of=$LOOP bs=4096 count=1 conv=notrunc 2>/dev/null

echo "=== ФАЙЛОВАЯ СИСТЕМА ПОВРЕЖДЕНА ==="
echo "Устройство: $LOOP"
echo "Для восстановления используйте:"
echo "  sudo fsck -y $LOOP"
```

Выполните:

```bash
chmod +x break_filesystem.sh
./break_filesystem.sh
```

---

## Пошаговая диагностика и восстановление

### Шаг 1. Попытка монтирования

```bash
sudo mkdir -p /mnt/fs_test
sudo mount /dev/loop0 /mnt/fs_test
```

**Ожидаемый результат:** Ошибка монтирования с указанием на повреждённый суперблок.

---

### Шаг 2. Проверка файловой системы (только чтение)

```bash
sudo fsck -n /dev/loop0
```

---

### Шаг 3. Поиск резервных суперблоков

```bash
sudo mke2fs -n /dev/loop0 2>/dev/null | grep -i "superblock"
```

**Ожидаемый результат:** Список позиций резервных суперблоков (32768, 98304 и т.д.).

---

### Шаг 4. Восстановление суперблока

```bash
sudo e2fsck -b 32768 /dev/loop0
```

Если не помогло, попробуйте другой номер из списка.

---

### Шаг 5. Автоматическое восстановление

```bash
sudo fsck -y /dev/loop0
```

---

### Шаг 6. Проверка и монтирование

```bash
sudo mount /dev/loop0 /mnt/fs_test
ls -la /mnt/fs_test/
```

---

## Скрипт восстановления

Создайте `fix_filesystem.sh`:

```bash
#!/bin/bash
DEVICE=${1:-/dev/loop0}

echo "=== ВОССТАНОВЛЕНИЕ ФАЙЛОВОЙ СИСТЕМЫ ==="
echo "Устройство: $DEVICE"

# Поиск резервных суперблоков
echo "Поиск резервных суперблоков..."
sudo mke2fs -n $DEVICE 2>/dev/null | grep -i "superblock" | head -5

# Попытка восстановления с резервным суперблоком
echo "Попытка восстановления с резервным суперблоком 32768..."
sudo e2fsck -b 32768 -y $DEVICE

# Если не помогло — полное восстановление
if [ $? -ne 0 ]; then
    echo "Полное восстановление..."
    sudo fsck -y $DEVICE
fi

echo "=== ГОТОВО ==="
```

---

# Лабораторная работа №3. Восстановление потерянных данных и LVM

**Цель:** Научиться восстанавливать удалённые файлы, повреждённые разделы и томы LVM с помощью TestDisk, extundelete и встроенных средств LVM.

**Продолжительность:** 2 часа.

---

## Теоретическая справка

### LVM: базовые понятия

- **PV (Physical Volume)** — физический том (диск или раздел)
- **VG (Volume Group)** — группа томов, объединяющая несколько PV
- **LV (Logical Volume)** — логический том, который используется как блочное устройство

### Восстановление LVM

LVM автоматически создаёт резервные копии метаданных в:
- `/etc/lvm/backup/<vg_name>` — после каждого изменения
- `/etc/lvm/archive/` — архив предыдущих состояний

Для восстановления используются команды:
- `pvcreate --uuid <UUID> --restorefile <file> <device>` — восстановление PV
- `vgcfgrestore -f <file> <vg_name>` — восстановление VG
- `vgchange -ay` — активация томов
- `vgreduce --removemissing --force <vg_name>` — удаление потерянных PV

---

## Подготовка LVM-стенда

Создайте скрипт `setup_lvm.sh` для настройки LVM на четырёх дисках:

```bash
#!/bin/bash
# Настройка LVM на 4 дисках

echo "=== НАСТРОЙКА LVM НА 4 ДИСКАХ ==="

# 1. Создание разделов на каждом диске (используем весь диск)
for disk in /dev/sdb /dev/sdc /dev/sdd /dev/sde; do
    sudo parted -s $disk mklabel gpt
    sudo parted -s $disk mkpart primary ext4 0% 100%
    sudo partprobe $disk
done

# 2. Создание физических томов (PV)
sudo pvcreate /dev/sdb1 /dev/sdc1 /dev/sdd1 /dev/sde1
echo "Физические тома созданы:"
sudo pvs

# 3. Создание группы томов (VG)
sudo vgcreate vg_data /dev/sdb1 /dev/sdc1 /dev/sdd1 /dev/sde1
echo "Группа томов создана:"
sudo vgs

# 4. Создание логических томов (LV)
sudo lvcreate -L 2G -n lv_data vg_data
sudo lvcreate -L 1G -n lv_logs vg_data
echo "Логические тома созданы:"
sudo lvs

# 5. Создание файловых систем
sudo mkfs.ext4 /dev/vg_data/lv_data
sudo mkfs.ext4 /dev/vg_data/lv_logs

# 6. Монтирование и создание тестовых файлов
sudo mkdir -p /mnt/lv_data /mnt/lv_logs
sudo mount /dev/vg_data/lv_data /mnt/lv_data
sudo mount /dev/vg_data/lv_logs /mnt/lv_logs

echo "Тестовые файлы..." | sudo tee /mnt/lv_data/test.txt
for i in {1..20}; do
    echo "File $i" | sudo tee /mnt/lv_data/file_$i.txt > /dev/null
done
for i in {1..10}; do
    echo "Log $i" | sudo tee /mnt/lv_logs/log_$i.log > /dev/null
done

echo "=== LVM НАСТРОЕН ==="
echo "PV: /dev/sdb1, /dev/sdc1, /dev/sdd1, /dev/sde1"
echo "VG: vg_data"
echo "LV: lv_data (2G), lv_logs (1G)"
echo "Точки монтирования: /mnt/lv_data, /mnt/lv_logs"
```

Выполните:

```bash
chmod +x setup_lvm.sh
./setup_lvm.sh
```

---

## Скрипт поломки LVM

Создайте скрипт `break_lvm.sh`, который повреждает метаданные LVM:

```bash
#!/bin/bash
# Повреждение LVM-метаданных

echo "=== ПОВРЕЖДЕНИЕ LVM-МЕТАДАННЫХ ==="

# 1. Создаём резервную копию метаданных VG
sudo vgcfgbackup vg_data
echo "Резервная копия создана в /etc/lvm/backup/"

# 2. Повреждаем метаданные на первом физическом томе
sudo dd if=/dev/urandom of=/dev/sdb1 bs=512 count=100 conv=notrunc 2>/dev/null

echo "=== LVM ПОВРЕЖДЕН ==="
echo "Повреждён PV: /dev/sdb1"
echo "Попробуйте выполнить: sudo vgs"
```

Выполните:

```bash
chmod +x break_lvm.sh
./break_lvm.sh
```

---

## Пошаговая диагностика и восстановление LVM

### Шаг 1. Диагностика состояния LVM

Проверьте состояние томов:

```bash
sudo pvs
sudo vgs
sudo lvs
```

**Ожидаемый результат:** Ошибка при обращении к повреждённому PV или сообщение о недоступности VG.

Проверьте логи:

```bash
sudo journalctl -xb | grep -i lvm
dmesg -T | grep -i lvm
```

---

### Шаг 2. Поиск резервных копий метаданных

```bash
# Просмотр доступных резервных копий
ls -la /etc/lvm/backup/
ls -la /etc/lvm/archive/
```

Найдите файл `/etc/lvm/backup/vg_data`.

---

### Шаг 3. Поиск UUID повреждённого PV

Извлеките UUID повреждённого физического тома из архивного файла:

```bash
# Просмотр содержимого резервной копии
sudo cat /etc/lvm/backup/vg_data | grep -A5 "physical_volumes"
```

Или используйте:

```bash
sudo grep -i "uuid" /etc/lvm/backup/vg_data | head -5
```

Запомните UUID (строка вида `"abc123-def456-..."`).

---

### Шаг 4. Восстановление физического тома

Используйте `pvcreate` с опциями `--uuid` и `--restorefile`:

```bash
sudo pvcreate --uuid "<UUID_из_архива>" --restorefile /etc/lvm/backup/vg_data /dev/sdb1
```

**Пример:**
```bash
sudo pvcreate --uuid "MsfsFe-zW9Y-hdAf-4VgW-xwGO-ZHPk-I4gJqL" --restorefile /etc/lvm/backup/vg_data /dev/sdb1
```

---

### Шаг 5. Восстановление группы томов

Восстановите метаданные VG из резервной копии:

```bash
sudo vgcfgrestore -f /etc/lvm/backup/vg_data vg_data
```

---

### Шаг 6. Активация томов

Активируйте все тома в группе:

```bash
sudo vgchange -ay vg_data
```

---

### Шаг 7. Проверка

```bash
sudo pvs
sudo vgs
sudo lvs
sudo mount /dev/vg_data/lv_data /mnt/lv_data
ls -la /mnt/lv_data/
```

---

### Шаг 8. Удаление потерянных PV (если не восстанавливаются)

Если какой-то PV не подлежит восстановлению, его можно удалить из VG:

```bash
# Активация с пропуском потерянных PV
sudo vgchange -ay --partial vg_data

# Удаление потерянного PV из VG
sudo vgreduce --removemissing --force vg_data
```

---

## Скрипт восстановления LVM

Создайте `fix_lvm.sh`:

```bash
#!/bin/bash
VG_NAME=${1:-vg_data}
PV_DEVICE=${2:-/dev/sdb1}

echo "=== ВОССТАНОВЛЕНИЕ LVM ==="
echo "VG: $VG_NAME"
echo "PV: $PV_DEVICE"

# 1. Поиск UUID повреждённого PV
echo "Поиск UUID повреждённого PV..."
UUID=$(sudo grep -i "uuid" /etc/lvm/backup/${VG_NAME} | head -1 | awk -F'"' '{print $2}')
if [ -z "$UUID" ]; then
    echo "UUID не найден. Попробуйте найти вручную."
    exit 1
fi
echo "Найден UUID: $UUID"

# 2. Восстановление PV
echo "Восстановление PV..."
sudo pvcreate --uuid "$UUID" --restorefile /etc/lvm/backup/${VG_NAME} $PV_DEVICE

# 3. Восстановление VG
echo "Восстановление VG..."
sudo vgcfgrestore -f /etc/lvm/backup/${VG_NAME} $VG_NAME

# 4. Активация томов
echo "Активация томов..."
sudo vgchange -ay $VG_NAME

# 5. Проверка
echo "=== ПРОВЕРКА ==="
sudo pvs
sudo vgs
sudo lvs

echo "=== ГОТОВО ==="
```

---

# Лабораторная работа №4. Работа с BTRFS

**Цель:** Научиться диагностировать и восстанавливать повреждённые файловые системы BTRFS, использовать встроенные инструменты восстановления.

**Продолжительность:** 2 часа.

---

## Теоретическая справка

**Btrfs** — современная файловая система с поддержкой снапшотов, сжатия, проверки целостности и RAID.

**Инструменты восстановления:**
- `btrfs check` — проверка целостности ФС
- `btrfs rescue` — восстановление повреждённой ФС
- `btrfs restore` — извлечение файлов с повреждённой ФС
- `btrfs scrub` — проверка и восстановление данных

---

## Скрипт поломки BTRFS

Создайте скрипт `break_btrfs.sh`:

```bash
#!/bin/bash
# Скрипт создаёт BTRFS-раздел и повреждает его

echo "=== СОЗДАНИЕ BTRFS-РАЗДЕЛА ==="

# 1. Создаём образ
dd if=/dev/zero of=/tmp/btrfs_test.img bs=1M count=500
LOOP=$(sudo losetup -f --show /tmp/btrfs_test.img)

# 2. Создаём BTRFS
sudo mkfs.btrfs -f $LOOP

# 3. Монтируем и создаём файлы
sudo mkdir -p /mnt/btrfs_test
sudo mount $LOOP /mnt/btrfs_test

echo "Создание тестовых файлов и снапшотов..."
for i in {1..20}; do
    echo "File $i" | sudo tee /mnt/btrfs_test/file_$i.txt > /dev/null
done

# Создаём подтом и снапшот
sudo btrfs subvolume create /mnt/btrfs_test/@subvol
echo "Subvolume file" | sudo tee /mnt/btrfs_test/@subvol/subfile.txt > /dev/null
sudo btrfs subvolume snapshot /mnt/btrfs_test/@subvol /mnt/btrfs_test/@snap1

sudo umount /mnt/btrfs_test

# 4. Повреждаем метаданные (записываем случайные данные в начало)
sudo dd if=/dev/urandom of=$LOOP bs=4096 count=10 conv=notrunc 2>/dev/null

echo "=== BTRFS-РАЗДЕЛ ПОВРЕЖДЕН ==="
echo "Устройство: $LOOP"
echo ""
echo "Для диагностики используйте:"
echo "  sudo btrfs check $LOOP"
echo "  sudo btrfs rescue chunk-recover $LOOP"
echo "  sudo btrfs restore $LOOP /tmp/restored/"
```

Выполните:

```bash
chmod +x break_btrfs.sh
./break_btrfs.sh
```

---

## Пошаговая диагностика и восстановление

### Шаг 1. Попытка монтирования в режиме восстановления

```bash
sudo mount -o recovery /dev/loop0 /mnt/btrfs_test
```

---

### Шаг 2. Проверка файловой системы

```bash
sudo btrfs check /dev/loop0
```

---

### Шаг 3. Восстановление суперблока

**Очистка повреждённого лога:**
```bash
sudo btrfs rescue zero-log /dev/loop0
```

**Восстановление chunk-дерева:**
```bash
sudo btrfs rescue chunk-recover -y /dev/loop0
```

---

### Шаг 4. Извлечение данных (btrfs restore)

Если ФС не поддаётся восстановлению:

```bash
# Просмотр содержимого
sudo btrfs restore -l /dev/loop0

# Восстановление всех файлов
sudo mkdir -p /tmp/btrfs_restored
sudo btrfs restore -x /dev/loop0 /tmp/btrfs_restored/
```

---

### Шаг 5. Проверка целостности (scrub)

Если система смонтировалась:

```bash
sudo btrfs scrub start /mnt/btrfs_test
sudo btrfs scrub status /mnt/btrfs_test
```

---

## Скрипт восстановления BTRFS

Создайте `fix_btrfs.sh`:

```bash
#!/bin/bash
DEVICE=${1:-/dev/loop0}
RESTORE_DIR=${2:-/tmp/btrfs_restored}

echo "=== ВОССТАНОВЛЕНИЕ BTRFS ==="
echo "Устройство: $DEVICE"

# 1. Попытка монтирования в recovery
echo "1. Попытка монтирования в recovery mode..."
sudo mkdir -p /mnt/btrfs_recovery
sudo mount -o recovery $DEVICE /mnt/btrfs_recovery 2>/dev/null
if [ $? -eq 0 ]; then
    echo "   УСПЕШНО! Данные доступны в /mnt/btrfs_recovery"
    exit 0
fi

# 2. Очистка лога
echo "2. Очистка повреждённого лога..."
sudo btrfs rescue zero-log $DEVICE

# 3. Попытка монтирования после очистки лога
echo "3. Повторная попытка монтирования..."
sudo mount $DEVICE /mnt/btrfs_recovery 2>/dev/null
if [ $? -eq 0 ]; then
    echo "   УСПЕШНО! Данные доступны в /mnt/btrfs_recovery"
    exit 0
fi

# 4. Восстановление chunk-дерева
echo "4. Восстановление chunk-дерева..."
sudo btrfs rescue chunk-recover -y $DEVICE

# 5. Извлечение данных
echo "5. Извлечение данных с помощью btrfs restore..."
sudo mkdir -p $RESTORE_DIR
sudo btrfs restore -x $DEVICE $RESTORE_DIR/

echo "=== ГОТОВО ==="
echo "Данные восстановлены в $RESTORE_DIR"
```
