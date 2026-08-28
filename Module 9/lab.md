# Лабораторная работа. Настройка автоматической блокировки от Brute Force

**Продолжительность:** 2 академических часа

**Оборудование:** Две виртуальные машины:
- **Целевой сервер (Target):** Debian 11/12 или Ubuntu 20.04/22.04 LTS
- **Машина атакующего (Attacker):** Kali Linux или любая Linux-система с установленным Hydra

---

## Цель работы

Научиться настраивать автоматическую блокировку IP-адресов при попытках подбора пароля (brute force) с использованием Fail2Ban, а также тестировать эффективность защиты с помощью инструмента Hydra.

В ходе работы вы освоите:
- установку и базовую настройку Fail2Ban для защиты SSH;
- конфигурирование параметров блокировки (количество попыток, время блокировки, окно наблюдения);
- мониторинг работы Fail2Ban и просмотр заблокированных IP-адресов;
- проведение атаки методом перебора паролей с использованием Hydra;
- верификацию срабатывания механизма блокировки.

---

## Теоретическая справка

### Что такое Brute Force атака?

Атака методом перебора (brute force) — это попытка подобрать пароль к учётной записи путём многократных попыток входа с различными комбинациями логинов и паролей. Любой Linux-сервер с открытым SSH-портом постоянно подвергается таким атакам со стороны автоматизированных ботов.

### Что такое Fail2Ban?

**Fail2Ban** — это инструмент, который сканирует логи на предмет повторяющихся неудачных попыток входа и автоматически блокирует IP-адреса злоумышленников на уровне файрвола. Он работает по следующему принципу:

1. **Мониторинг логов** — Fail2Ban отслеживает файлы логов (например, `/var/log/auth.log` для SSH).
2. **Обнаружение атак** — при превышении заданного количества неудачных попыток (`maxretry`) в течение определённого временного окна (`findtime`) запускается механизм блокировки.
3. **Блокировка** — Fail2Ban добавляет правило в файрвол (iptables, nftables, firewalld), блокирующее IP-адрес нарушителя на заданное время (`bantime`).
4. **Разблокировка** — по истечении `bantime` правило автоматически удаляется.

### Архитектура Fail2Ban

| Компонент | Назначение | Расположение |
|-----------|------------|--------------|
| **jail.conf** | Основной файл конфигурации (не рекомендуется изменять) | `/etc/fail2ban/jail.conf` |
| **jail.local** | Пользовательские настройки (переопределяют jail.conf) | `/etc/fail2ban/jail.local` |
| **jail.d/** | Каталог для дополнительных конфигурационных файлов | `/etc/fail2ban/jail.d/` |
| **filter.d/** | Фильтры для поиска подозрительной активности в логах | `/etc/fail2ban/filter.d/` |
| **action.d/** | Действия при срабатывании (блокировка, уведомления) | `/etc/fail2ban/action.d/` |

### Ключевые параметры Fail2Ban

| Параметр | Описание | Рекомендуемое значение |
|----------|----------|------------------------|
| `maxretry` | Количество неудачных попыток до блокировки | 3–5 |
| `findtime` | Временное окно (в секундах) для подсчёта попыток | 600 (10 минут) |
| `bantime` | Время блокировки (в секундах) | 3600 (1 час) |
| `ignoreip` | IP-адреса, которые никогда не блокируются | 127.0.0.1/8, внутренние IP |
| `action` | Действие при блокировке | `iptables` или `ufw` |

---

## Подготовка стенда

### Шаг 1. Настройка виртуальных машин

Создайте две виртуальные машины в одной сети:

| Роль | ОС | IP-адрес |
|------|-----|----------|
| **Target** | Debian/Ubuntu | 192.168.100.10 |
| **Attacker** | Kali Linux | 192.168.100.20 |

Убедитесь, что:
- Обе машины имеют сетевой доступ друг к другу.
- На целевой машине установлен и запущен SSH-сервер.
- На машине атакующего установлен Hydra.

### Шаг 2. Установка Fail2Ban на целевой сервер

Подключитесь к целевой машине и выполните:

```bash
# Обновление системы
sudo apt update && sudo apt upgrade -y

# Установка Fail2Ban
sudo apt install -y fail2ban iptables

# Проверка статуса
sudo systemctl status fail2ban
```

**Ожидаемый результат:** Fail2Ban должен быть активен (active).

---

## Часть 1. Базовая настройка Fail2Ban (25 минут)

### Шаг 1. Создание конфигурационного файла jail.local

Вместо прямого редактирования `/etc/fail2ban/jail.conf` (который может быть перезаписан при обновлении), создайте файл с пользовательскими настройками:

```bash
sudo nano /etc/fail2ban/jail.local
```

Добавьте следующие настройки:

```ini
[DEFAULT]
# Игнорируемые IP-адреса (не блокировать)
ignoreip = 127.0.0.1/8 ::1

# Время блокировки (в секундах) — 1 час
bantime = 3600

# Временное окно для подсчёта попыток — 10 минут
findtime = 600

# Количество неудачных попыток до блокировки
maxretry = 5

# Действие при блокировке — iptables
banaction = iptables-multiport

[sshd]
# Включение защиты SSH
enabled = true

# Порт SSH (можно указать 22 или ssh)
port = ssh

# Фильтр для SSH (используется /etc/fail2ban/filter.d/sshd.conf)
filter = sshd

# Путь к лог-файлу
logpath = /var/log/auth.log
```

Сохраните файл (`Ctrl+X`, затем `Y`, затем `Enter`).

### Шаг 2. Перезапуск Fail2Ban

```bash
sudo systemctl restart fail2ban
```

### Шаг 3. Проверка статуса

```bash
sudo systemctl status fail2ban
```

**Ожидаемый результат:** Статус `active (running)`.

Проверьте состояние защиты SSH:

```bash
sudo fail2ban-client status sshd
```

**Ожидаемый результат:** Список заблокированных IP-адресов (пока пустой).

---

## Часть 2. Тестирование защиты (Simulated Attack) (25 минут)

### Шаг 1. Установка Hydra на машину атакующего

Подключитесь к машине атакующего и установите Hydra:

```bash
# Для Kali Linux (уже установлена по умолчанию)
# Для других дистрибутивов:
sudo apt update
sudo apt install -y hydra
```

### Шаг 2. Создание словаря для атаки

Создайте файл со списком паролей:

```bash
echo "password" > /tmp/passwords.txt
echo "123456" >> /tmp/passwords.txt
echo "admin" >> /tmp/passwords.txt
echo "root" >> /tmp/passwords.txt
echo "letmein" >> /tmp/passwords.txt
```

### Шаг 3. Проведение brute force атаки

**Важно:** Используйте несуществующего пользователя, чтобы не заблокировать реальные учётные записи!

```bash
hydra -l nonexistent_user -P /tmp/passwords.txt -t 4 192.168.100.10 ssh
```

Где:
- `-l nonexistent_user` — использовать логин `nonexistent_user`
- `-P /tmp/passwords.txt` — использовать словарь паролей из файла
- `-t 4` — количество параллельных потоков
- `192.168.100.10` — IP-адрес целевого сервера
- `ssh` — сервис для атаки

**Ожидаемый результат:** Hydra начнёт перебирать пароли. После 5 неудачных попыток (в соответствии с `maxretry=5`) IP-адрес атакующего будет заблокирован.

### Шаг 4. Проверка блокировки на целевом сервере

На целевой машине выполните:

```bash
# Проверка статуса SSH jail
sudo fail2ban-client status sshd
```

**Ожидаемый результат:** В строке `Banned IP list` должен появиться IP-адрес атакующего.

Проверьте правила iptables:

```bash
sudo iptables -L -n | grep -i fail2ban
```

**Ожидаемый результат:** Правило, блокирующее IP-адрес атакующего.

### Шаг 5. Проверка блокировки с машины атакующего

Попробуйте подключиться по SSH:

```bash
ssh nonexistent_user@192.168.100.10
```

**Ожидаемый результат:** Таймаут или ошибка подключения.

---

## Часть 3. Расширенная настройка (25 минут)

### Шаг 1. Настройка уведомлений по электронной почте

Добавьте в `/etc/fail2ban/jail.local` в секцию `[DEFAULT]`:

```ini
destemail = admin@example.com
sendername = Fail2Ban
mta = sendmail
action = %(action_mwl)s
```

### Шаг 2. Создание собственного фильтра для другого сервиса

Создайте фильтр для защиты веб-сервера Nginx от brute force:

```bash
sudo nano /etc/fail2ban/filter.d/nginx-auth.conf
```

Добавьте:

```ini
[Definition]
failregex = ^<HOST> - .* "POST /login HTTP/1\..*" 401
ignoreregex =
```

Создайте джейл для Nginx в `/etc/fail2ban/jail.local`:

```ini
[nginx-auth]
enabled = true
port = http,https
filter = nginx-auth
logpath = /var/log/nginx/access.log
maxretry = 5
bantime = 600
```

### Шаг 3. Настройка игнорирования IP-адресов

Для добавления IP-адресов в белый список используйте параметр `ignoreip`:

```ini
[DEFAULT]
ignoreip = 127.0.0.1/8 ::1 192.168.100.0/24
```

Это позволит не блокировать IP-адреса из вашей локальной сети.

### Шаг 4. Применение изменений

```bash
sudo systemctl restart fail2ban
```

---

## Часть 4. Анализ логов и мониторинг (20 минут)

### Шаг 1. Просмотр логов Fail2Ban

```bash
# Живой просмотр логов
sudo tail -f /var/log/fail2ban.log

# Просмотр последних событий
sudo tail -n 50 /var/log/fail2ban.log
```

**Пример вывода:**
```
2026-08-28 10:15:23,342 fail2ban.actions [12345]: NOTICE  [sshd] Ban 192.168.100.20
2026-08-28 11:15:23,342 fail2ban.actions [12345]: NOTICE  [sshd] Unban 192.168.100.20
```

### Шаг 2. Просмотр логов аутентификации

```bash
sudo tail -f /var/log/auth.log
```

Вы увидите записи о неудачных попытках входа, которые Fail2Ban отслеживает.

### Шаг 3. Команды управления Fail2Ban

| Команда | Назначение |
|---------|------------|
| `sudo fail2ban-client status` | Показать статус всех джейлов |
| `sudo fail2ban-client status sshd` | Показать статус конкретного джейла |
| `sudo fail2ban-client set sshd banip <IP>` | Вручную заблокировать IP |
| `sudo fail2ban-client set sshd unbanip <IP>` | Разблокировать IP |
| `sudo fail2ban-client reload` | Перезагрузить конфигурацию без перезапуска |

### Шаг 4. Практика: ручная блокировка и разблокировка

Попробуйте вручную заблокировать IP-адрес:

```bash
sudo fail2ban-client set sshd banip 192.168.100.99
```

Проверьте, что IP появился в списке заблокированных:

```bash
sudo fail2ban-client status sshd
```

Затем разблокируйте его:

```bash
sudo fail2ban-client set sshd unbanip 192.168.100.99
```

---

## Часть 5. Скрипты для автоматизации (бонус)

### Скрипт установки и настройки Fail2Ban

Создайте файл `setup_fail2ban.sh`:

```bash
#!/bin/bash
# Скрипт автоматической установки и настройки Fail2Ban

echo "=== УСТАНОВКА FAIL2BAN ==="

# 1. Установка
sudo apt update
sudo apt install -y fail2ban iptables

# 2. Создание конфигурации
sudo tee /etc/fail2ban/jail.local > /dev/null <<EOF
[DEFAULT]
ignoreip = 127.0.0.1/8 ::1
bantime = 3600
findtime = 600
maxretry = 5
banaction = iptables-multiport

[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
EOF

# 3. Запуск
sudo systemctl restart fail2ban
sudo systemctl enable fail2ban

# 4. Проверка
sudo fail2ban-client status

echo "=== УСТАНОВКА ЗАВЕРШЕНА ==="
```

Сделайте скрипт исполняемым и выполните:

```bash
chmod +x setup_fail2ban.sh
./setup_fail2ban.sh
```

### Скрипт проверки защиты

Создайте `check_fail2ban.sh`:

```bash
#!/bin/bash
# Скрипт проверки статуса Fail2Ban

echo "=== ПРОВЕРКА FAIL2BAN ==="

# Статус сервиса
echo "Статус сервиса:"
sudo systemctl status fail2ban --no-pager | grep -E "Active|Loaded"

# Статус джейлов
echo -e "\nСтатус джейлов:"
sudo fail2ban-client status

# Просмотр логов
echo -e "\nПоследние 10 записей в логе:"
sudo tail -n 10 /var/log/fail2ban.log

# Проверка iptables
echo -e "\nПравила iptables:"
sudo iptables -L -n | grep -i fail2ban || echo "Нет правил Fail2Ban в iptables"
```

---

## Часть 6. Альтернативные методы защиты (обзор)

### Метод 1: SSHGuard

SSHGuard — это альтернатива Fail2Ban, которая также защищает от brute force атак, но работает на основе накопления «очков атаки».

```bash
# Установка
sudo apt install -y sshguard

# Конфигурация
sudo nano /etc/sshguard/sshguard.conf
```

### Метод 2: Ограничение с помощью iptables (без Fail2Ban)

```bash
# Ограничение количества новых соединений с одного IP
sudo iptables -A INPUT -p tcp --dport 22 -m state --state NEW -m recent --set
sudo iptables -A INPUT -p tcp --dport 22 -m state --state NEW -m recent --update --seconds 60 --hitcount 4 -j DROP
```
