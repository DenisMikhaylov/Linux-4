# Лабораторная работа. Устранение неполадок сетевого взаимодействия в Linux

**Продолжительность:** 2 академических часа

**Оборудование:** Виртуальная машина с Ubuntu 20.04/22.04 LTS (или Debian 12 с установленным Netplan), сетевое подключение к интернету

---

## Цель работы

Научиться диагностировать и устранять различные неполадки сетевого взаимодействия в Linux. В ходе работы вы освоите:

- диагностику сетевых подключений, маршрутизации и разрешения имён;
- унификацию управления сетевыми интерфейсами с помощью Netplan;
- настройку режима и скорости канала, MTU, VLAN;
- мониторинг сетевой подсистемы.

---

## Теоретическая справка

### Уровни диагностики сетевых проблем

При диагностике сетевых неполадок рекомендуется применять **послойный подход** (Rule-Out Method):

| Уровень | Что проверяем | Инструменты |
|---------|---------------|-------------|
| **Физический** | Наличие соединения, скорость, дуплекс | `ethtool`, индикаторы на сетевой карте |
| **Драйвер/прошивка** | Загрузка драйвера, ошибки в журнале | `lspci -nnk`, `lsmod`, `dmesg` |
| **Конфигурация IP/L2-L3** | IP-адрес, маска, VLAN, MTU | `ip addr`, `ip route`, `netplan apply` |
| **Маршрутизация** | Доступность шлюза, маршруты | `ip route`, `traceroute` |
| **Разрешение имён** | Работа DNS | `dig`, `nslookup`, `/etc/resolv.conf` |
| **Брандмауэр** | Правила фильтрации | `iptables -L`, `ufw status` |
| **Производительность** | Пропускная способность, задержки | `iperf3`, `ss`, `sar -n DEV` |

### Netplan — унифицированное управление сетью

**Netplan** — это утилита для декларативного управления сетевыми интерфейсами с использованием YAML-конфигураций. Она поддерживает два бэкенда-рендерера:
- `networkd` — для серверных сред (рекомендуется)
- `NetworkManager` — для рабочих станций

Конфигурационные файлы Netplan хранятся в `/etc/netplan/` с расширением `.yaml`.

### Ключевые инструменты диагностики

| Инструмент | Назначение |
|------------|------------|
| `ping` | Проверка базовой连通ности и измерение задержек |
| `traceroute` / `mtr` | Отслеживание маршрута пакетов |
| `ss` | Просмотр сетевых соединений и сокетов (замена `netstat`) |
| `ethtool` | Просмотр и настройка параметров сетевых интерфейсов |
| `dig` / `nslookup` | Диагностика DNS |
| `ip` | Управление сетевыми интерфейсами, адресами и маршрутами |

---

## Подготовка стенда

### Шаг 1. Установка необходимых пакетов

```bash
sudo apt update
sudo apt install -y netplan.io ethtool traceroute mtr dnsutils \
    tcpdump iftop nload bmon vnstat nethogs iperf3
```

### Шаг 2. Создание рабочей директории

```bash
mkdir -p ~/lab_network_troubleshooting
cd ~/lab_network_troubleshooting
```

### Шаг 3. Определение имени сетевого интерфейса

```bash
ip link show | grep -E "^[0-9]+:" | grep -v lo | awk -F: '{print $2}'
```

Запомните имя интерфейса (например, `ens3`, `enp0s3` или `eth0`). В дальнейшем будем обозначать его как `$IFACE`.

---

## Часть 1. Диагностика сетевых подключений и маршрутизации (25 минут)

### Теоретическая справка

Первичная диагностика начинается с проверки физического уровня, затем переходят к IP-конфигурации, маршрутизации и разрешению имён.

---

### Скрипт поломки

Создайте скрипт `break_connectivity.sh`:

```bash
#!/bin/bash
echo "=== СОЗДАНИЕ ПРОБЛЕМ С СЕТЕВОЙ СВЯЗНОСТЬЮ ==="

# Определяем интерфейс
IFACE=$(ip link show | grep -E "^[0-9]+:" | grep -v lo | awk -F: '{print $2}' | head -1 | xargs)
echo "Используется интерфейс: $IFACE"

# 1. Удаляем IP-адрес с интерфейса
sudo ip addr flush dev $IFACE

# 2. Добавляем неправильный IP-адрес
sudo ip addr add 192.168.200.100/24 dev $IFACE

# 3. Удаляем маршрут по умолчанию
sudo ip route del default 2>/dev/null

echo "=== ПРОБЛЕМЫ СОЗДАНЫ ==="
echo "1. IP-адрес изменён на 192.168.200.100/24"
echo "2. Маршрут по умолчанию удалён"
echo ""
echo "Попробуйте выполнить диагностику:"
echo "  ip addr show $IFACE"
echo "  ip route show"
echo "  ping 8.8.8.8"
```

Выполните скрипт:

```bash
chmod +x break_connectivity.sh
./break_connectivity.sh
```

---

### Диагностика

**Шаг 1. Проверка состояния интерфейса**

```bash
ip addr show $IFACE
```

**Ожидаемый результат:** IP-адрес 192.168.200.100/24 (не соответствует вашей сети).

**Шаг 2. Проверка таблицы маршрутизации**

```bash
ip route show
```

**Ожидаемый результат:** Отсутствует маршрут по умолчанию (default).

**Шаг 3. Проверка связности**

```bash
ping -c 3 8.8.8.8
```

**Ожидаемый результат:** Нет ответа (нет маршрута).

**Шаг 4. Проверка маршрута**

```bash
traceroute -n 8.8.8.8
```

---

### Восстановление

**Шаг 1. Восстановление IP-адреса (через DHCP)**

```bash
sudo dhclient -v $IFACE
```

**Шаг 2. Если DHCP недоступен — ручная настройка**

Определите правильную сеть (например, через `ip route show` на соседней машине или `arp -a`):

```bash
# Пример для сети 192.168.1.0/24 с шлюзом 192.168.1.1
sudo ip addr flush dev $IFACE
sudo ip addr add 192.168.1.100/24 dev $IFACE
sudo ip link set $IFACE up
sudo ip route add default via 192.168.1.1
```

**Шаг 3. Проверка**

```bash
ping -c 3 8.8.8.8
ping -c 3 google.com
```

---

## Часть 2. Проблемы с разрешением имён (DNS) (20 минут)

### Теоретическая справка

Проблемы с DNS часто выглядят как потеря сетевой связности: `ping` по IP работает, а по доменному имени — нет. Конфигурация DNS хранится в `/etc/resolv.conf` (обычно управляется через systemd-resolved).

---

### Скрипт поломки

Создайте `break_dns.sh`:

```bash
#!/bin/bash
echo "=== СОЗДАНИЕ ПРОБЛЕМ С DNS ==="

# Сохраняем оригинальный resolv.conf
sudo cp /etc/resolv.conf /etc/resolv.conf.backup

# Удаляем корректные DNS-серверы
echo "nameserver 1.1.1.1" | sudo tee /etc/resolv.conf > /dev/null
echo "nameserver 8.8.8.8" | sudo tee -a /etc/resolv.conf > /dev/null

# Добавляем несуществующий DNS-сервер (будет таймаут)
echo "nameserver 192.168.200.1" | sudo tee -a /etc/resolv.conf > /dev/null

echo "=== ПРОБЛЕМЫ С DNS СОЗДАНЫ ==="
echo "В /etc/resolv.conf добавлен неработающий DNS-сервер"
echo ""
echo "Попробуйте выполнить диагностику:"
echo "  ping 8.8.8.8"
echo "  ping google.com"
echo "  dig google.com"
```

Выполните:

```bash
chmod +x break_dns.sh
./break_dns.sh
```

---

### Диагностика

**Шаг 1. Проверка связности по IP**

```bash
ping -c 3 8.8.8.8
```

**Ожидаемый результат:** IP работает (если маршрутизация настроена).

**Шаг 2. Проверка разрешения имён**

```bash
ping -c 3 google.com
```

**Ожидаемый результат:** Таймаут или ошибка `ping: google.com: Temporary failure in name resolution`.

**Шаг 3. Проверка DNS с помощью dig**

```bash
dig google.com
```

**Ожидаемый результат:** Таймаут или ошибка.

**Шаг 4. Просмотр текущей конфигурации DNS**

```bash
cat /etc/resolv.conf
systemd-resolve --status 2>/dev/null || resolvectl status 2>/dev/null
```

---

### Восстановление

**Шаг 1. Восстановление из резервной копии**

```bash
sudo cp /etc/resolv.conf.backup /etc/resolv.conf
```

**Шаг 2. Если резервной копии нет — ручная настройка**

Для систем с systemd-resolved:

```bash
sudo systemd-resolve --set-dns=8.8.8.8 --set-dns=1.1.1.1
```

Или через Netplan (см. Часть 3).

**Шаг 3. Проверка**

```bash
ping -c 3 google.com
dig google.com
```

---

## Часть 3. Унификация управления сетевыми интерфейсами. Netplan (25 минут)

### Теоретическая справка

Netplan — это высокоуровневый инструмент конфигурации сети, использующий YAML-файлы. Основные параметры:

| Параметр | Описание |
|----------|----------|
| `dhcp4: true/false` | Включение/отключение DHCPv4 |
| `addresses: [IP/маска]` | Статический IP-адрес |
| `routes:` | Статические маршруты |
| `nameservers:` | DNS-серверы |
| `mtu: N` | MTU интерфейса |

---

### Скрипт поломки

Создайте `break_netplan.sh`:

```bash
#!/bin/bash
echo "=== ПОВРЕЖДЕНИЕ КОНФИГУРАЦИИ NETPLAN ==="

# Создаём резервную копию существующих конфигов
sudo mkdir -p /etc/netplan/backup
sudo cp /etc/netplan/*.yaml /etc/netplan/backup/ 2>/dev/null

# Создаём некорректный конфигурационный файл
sudo tee /etc/netplan/99-broken.yaml > /dev/null <<EOF
network:
  version: 2
  ethernets:
    $IFACE:
      dhcp4: false
      addresses:
        - 10.0.0.250/24
      routes:
        - to: default
          via: 10.0.0.1
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
      # Синтаксическая ошибка — неверный отступ
        mtu: 9000
EOF

echo "=== КОНФИГУРАЦИЯ NETPLAN ПОВРЕЖДЕНА ==="
echo "Создан файл /etc/netplan/99-broken.yaml с синтаксической ошибкой"
echo ""
echo "Попробуйте выполнить:"
echo "  sudo netplan try"
echo "  sudo netplan apply"
```

Выполните:

```bash
chmod +x break_netplan.sh
./break_netplan.sh
```

---

### Диагностика

**Шаг 1. Попытка применения конфигурации**

```bash
sudo netplan try
```

**Ожидаемый результат:** Ошибка парсинга YAML с указанием строки и позиции.

**Шаг 2. Проверка синтаксиса**

```bash
sudo netplan generate
```

**Ожидаемый результат:** Сообщение об ошибке в файле `99-broken.yaml`.

**Шаг 3. Просмотр содержимого файла**

```bash
cat /etc/netplan/99-broken.yaml
```

---

### Восстановление

**Шаг 1. Удаление проблемного файла**

```bash
sudo rm /etc/netplan/99-broken.yaml
```

**Шаг 2. Проверка и применение**

```bash
sudo netplan generate
sudo netplan apply
```

**Шаг 3. Если нужно создать корректный файл**

Создайте `/etc/netplan/00-installer-config.yaml`:

```yaml
network:
  version: 2
  ethernets:
    $IFACE:
      dhcp4: true
```

Примените:

```bash
sudo netplan apply
```

---

## Часть 4. Настройка режима и скорости канала, MTU, VLAN (25 минут)

### Теоретическая справка

**ethtool** позволяет просматривать и изменять параметры сетевых интерфейсов:

- `ethtool $IFACE` — просмотр текущих параметров
- `ethtool -s $IFACE speed 1000 duplex full autoneg off` — изменение скорости и дуплекса
- MTU можно настроить через Netplan (`mtu: 9000`) или через `ip link set $IFACE mtu 9000`
- VLAN создаётся командой `ip link add link $IFACE name $IFACE.100 type vlan id 100`

---

### Скрипт поломки

Создайте `break_ethtool.sh`:

```bash
#!/bin/bash
echo "=== СОЗДАНИЕ ПРОБЛЕМ С ПАРАМЕТРАМИ ИНТЕРФЕЙСА ==="

# 1. Устанавливаем некорректный MTU (слишком маленький)
sudo ip link set dev $IFACE mtu 68

# 2. Отключаем автосогласование и устанавливаем низкую скорость (если поддерживается)
sudo ethtool -s $IFACE autoneg off speed 10 duplex half 2>/dev/null || \
    echo "Примечание: изменение скорости не поддерживается для этого интерфейса"

echo "=== ПРОБЛЕМЫ СОЗДАНЫ ==="
echo "1. MTU установлен на 68 (минимально возможный)"
echo "2. Автосогласование отключено, скорость 10 Мбит/с, полудуплекс"
echo ""
echo "Попробуйте выполнить диагностику:"
echo "  sudo ethtool $IFACE"
echo "  ip link show $IFACE"
```

Выполните:

```bash
chmod +x break_ethtool.sh
./break_ethtool.sh
```

---

### Диагностика

**Шаг 1. Просмотр параметров интерфейса**

```bash
sudo ethtool $IFACE
```

**Ожидаемый результат:** 
- `Speed: 10Mb/s`
- `Duplex: Half`
- `Auto-negotiation: off`

**Шаг 2. Проверка MTU**

```bash
ip link show $IFACE
```

**Ожидаемый результат:** `mtu 68`

**Шаг 3. Проверка влияния MTU**

```bash
ping -M do -s 1000 8.8.8.8
```

**Ожидаемый результат:** Ошибка `Frag needed` или `Message too long`.

---

### Восстановление

**Шаг 1. Восстановление MTU**

```bash
sudo ip link set dev $IFACE mtu 1500
```

**Шаг 2. Восстановление автосогласования**

```bash
sudo ethtool -s $IFACE autoneg on
```

**Шаг 3. Проверка**

```bash
sudo ethtool $IFACE
ip link show $IFACE
ping -c 3 8.8.8.8
```

---

## Часть 5. Мониторинг сетевой подсистемы (20 минут)

### Теоретическая справка

Для мониторинга сети используются следующие инструменты:

| Инструмент | Назначение |
|------------|------------|
| `nload` | Общая пропускная способность интерфейса |
| `iftop` | Трафик по соединениям/хостам |
| `nethogs` | Трафик по процессам |
| `vnstat` | Историческая статистика трафика |
| `ss` | Активные соединения и сокеты |

---

### Задание (без скрипта поломки — практика мониторинга)

**Шаг 1. Установка инструментов мониторинга** (если ещё не установлены)

```bash
sudo apt install -y nload iftop nethogs vnstat
```

**Шаг 2. Мониторинг общей пропускной способности**

```bash
nload
```

Нажмите стрелки влево/вправо для переключения между интерфейсами. Выход — `q`.

**Шаг 3. Мониторинг трафика по соединениям**

```bash
sudo iftop -i $IFACE
```

Выход — `q`.

**Шаг 4. Мониторинг трафика по процессам**

```bash
sudo nethogs $IFACE
```

Выход — `q`.

**Шаг 5. Просмотр активных соединений**

```bash
ss -tulpn
```

**Шаг 6. Настройка исторического мониторинга (vnstat)**

```bash
sudo systemctl enable --now vnstat
vnstat -i $IFACE
```

**Шаг 7. Имитация нагрузки для тестирования мониторинга**

В отдельном терминале выполните:

```bash
# Генерация HTTP-трафика
for i in {1..100}; do curl -s http://speedtest.tele2.net/100MB.zip > /dev/null & done
wait
```

Наблюдайте за изменениями в `nload`, `iftop` и `nethogs`.

---

## Часть 6. Создание VLAN (дополнительное задание)

### Теоретическая справка

VLAN (Virtual LAN) позволяет разделять трафик на логические сегменты. Создаётся командой:

```bash
ip link add link $IFACE name $IFACE.100 type vlan id 100
ip addr add 192.168.100.1/24 dev $IFACE.100
ip link set $IFACE.100 up
```

В Netplan VLAN настраивается так:

```yaml
network:
  version: 2
  ethernets:
    $IFACE:
      dhcp4: false
  vlans:
    vlan100:
      id: 100
      link: $IFACE
      addresses: [192.168.100.1/24]
```

---

### Задание

**Шаг 1. Создание VLAN-интерфейса**

```bash
sudo ip link add link $IFACE name $IFACE.100 type vlan id 100
sudo ip addr add 192.168.100.1/24 dev $IFACE.100
sudo ip link set $IFACE.100 up
```

**Шаг 2. Проверка**

```bash
ip link show $IFACE.100
ip addr show $IFACE.100
```

**Шаг 3. Удаление VLAN-интерфейса**

```bash
sudo ip link delete $IFACE.100
```

---

## Часть 7. Скрипт общего восстановления

Создайте универсальный скрипт `fix_network.sh`:

```bash
#!/bin/bash
echo "=== УНИВЕРСАЛЬНОЕ ВОССТАНОВЛЕНИЕ СЕТИ ==="

IFACE=$(ip link show | grep -E "^[0-9]+:" | grep -v lo | awk -F: '{print $2}' | head -1 | xargs)

# 1. Восстановление интерфейса
echo "[1/5] Сброс интерфейса..."
sudo ip link set $IFACE down
sudo ip addr flush dev $IFACE
sudo ip link set $IFACE up

# 2. Получение IP через DHCP
echo "[2/5] Получение IP через DHCP..."
sudo dhclient -v $IFACE 2>/dev/null || echo "DHCP не доступен"

# 3. Восстановление MTU
echo "[3/5] Восстановление MTU..."
sudo ip link set dev $IFACE mtu 1500

# 4. Восстановление автосогласования
echo "[4/5] Восстановление автосогласования..."
sudo ethtool -s $IFACE autoneg on 2>/dev/null

# 5. Восстановление DNS
echo "[5/5] Восстановление DNS..."
sudo systemctl restart systemd-resolved 2>/dev/null || sudo service networking restart

echo "=== ВОССТАНОВЛЕНИЕ ЗАВЕРШЕНО ==="
echo "Проверка: ping -c 3 8.8.8.8"
```

Сделайте скрипт исполняемым:

```bash
chmod +x fix_network.sh
```
