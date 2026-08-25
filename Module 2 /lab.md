
# Лабораторная работа  
## «Работа с журналами и планировщиком в Linux»  
**(2 академических часа, стенд из 3 машин Debian)**

---

### Подготовка к работе
- У вас есть три виртуальные машины с Debian 11/12.
- Запомните их IP-адреса и имена (можно задать свои, но далее в инструкции используются имена `logserver`, `client1`, `client2`).
- Все действия выполняются от пользователя с правами `sudo` (или от root). Если используете обычного пользователя, добавляйте `sudo` перед каждой командой.
- Убедитесь, что на всех машинах установлены пакеты: `rsyslog`, `logrotate`, `systemd`, `cron` (обычно уже есть). При необходимости установите:  
  `sudo apt update && sudo apt install rsyslog logrotate systemd cron -y`

---

## Часть 1. Централизованный сбор логов (30 минут)

### Шаг 1. Настройка лог-сервера (`logserver`)

1. Откройте конфигурационный файл rsyslog:
   ```
   sudo nano /etc/rsyslog.conf
   ```
2. Найдите строки, отвечающие за модуль UDP-приёма (они закомментированы). Раскомментируйте их:
   ```
   #module(load="imudp")
   #input(type="imudp" port="514")
   ```
   Должно стать:
   ```
   module(load="imudp")
   input(type="imudp" port="514")
   ```
3. Сохраните файл (`Ctrl+X`, затем `Y`, затем `Enter`).
4. Создайте каталог для удалённых логов:
   ```
   sudo mkdir -p /var/log/remote
   ```
5. Создайте дополнительный конфигурационный файл для шаблонов:
   ```
   sudo nano /etc/rsyslog.d/remote.conf
   ```
   Вставьте в него:
   ```
   $template RemoteHost,"/var/log/remote/%HOSTNAME%/%PROGRAMNAME%.log"
   *.* ?RemoteHost
   ```
   Сохраните.
6. Перезапустите rsyslog:
   ```
   sudo systemctl restart rsyslog
   ```
7. Проверьте, что служба слушает порт 514 UDP:
   ```
   sudo ss -lunp | grep 514
   ```
   Вы должны увидеть строку с `rsyslog` и `*:514`.

---

### Шаг 2. Настройка клиентов (`client1` и `client2`)

На каждой клиентской машине выполните:

1. Откройте конфигурационный файл rsyslog:
   ```
   sudo nano /etc/rsyslog.conf
   ```
2. В конец файла добавьте строку (замените IP на реальный IP вашего `logserver`):
   ```
   *.* @192.168.10.10:514
   ```
   Если вы используете TCP, используйте `@@`.
3. Сохраните и закройте файл.
4. Перезапустите rsyslog:
   ```
   sudo systemctl restart rsyslog
   ```

---

### Шаг 3. Проверка централизованного сбора

1. На **клиенте 1** сгенерируйте тестовое сообщение:
   ```
   logger "Test message from client1"
   ```
   На **клиенте 2** аналогично:
   ```
   logger "Test message from client2"
   ```
2. На **лог-сервере** проверьте, появились ли файлы:
   ```
   ls -l /var/log/remote/
   ```
   Вы должны увидеть каталоги `client1` и `client2` (или с IP, если имена не разрешаются – но у нас заданы имена хостов).
3. Просмотрите содержимое логов, например:
   ```
   cat /var/log/remote/client1/logger.log
   ```
   Там должно быть ваше тестовое сообщение. Если его нет – проверьте настройки и перезапустите rsyslog на всех машинах.

---

## Часть 2. Анализ системных журналов (25 минут)

### Шаг 4. Базовый анализ с помощью `journalctl`

На любой из машин (выберите одну, например, `client1`) выполните команды и запишите результаты.

1. Просмотр всех системных логов с начала загрузки:
   ```
   journalctl -b
   ```
   (можно пролистать клавишами `PgUp`/`PgDn`, выход – `q`).

2. Логи только за последние 30 минут:
   ```
   journalctl --since "30 minutes ago"
   ```

3. Вывод только сообщений с уровнем ошибки и выше (приоритет `err`, `crit`, `alert`, `emerg`):
   ```
   journalctl -p err -b
   ```
   Сколько строк выдало? Запомните.

4. Логи, относящиеся к службе SSH (например, неудачные попытки входа):
   ```
   journalctl -u ssh.service --since today
   ```

5. Логи ядра (можно использовать `dmesg`, но в journal тоже есть):
   ```
   journalctl -k -p err
   ```

6. Поиск сообщений о `sudo` (для текущего пользователя):
   ```
   sudo journalctl | grep "sudo"
   ```
   Или без `sudo`, если пользователь не выполнял sudo, можно поискать `authentication failure`:
   ```
   sudo grep "authentication failure" /var/log/auth.log
   ```
   (на Debian этот файл часто ротируется, лучше использовать `journalctl -u sshd`).

---

### Шаг 5. Анализ централизованных логов на сервере

На `logserver` выполните:

1. Найдите все сообщения об ошибках, пришедшие с клиентов (по всем файлам):
   ```
   sudo grep -i "error" /var/log/remote/*/*.log | head -20
   ```

2. Посчитайте количество записей от каждого клиента:
   ```
   for host in client1 client2; do
     echo -n "$host: "
     sudo find /var/log/remote/$host -type f -exec cat {} \; | wc -l
   done
   ```

3. Найдите сообщения, содержащие слово `fail` (можно указать `denied`):
   ```
   sudo grep -i "fail" /var/log/remote/client1/*.log
   ```

**Запишите** в отчёт команды, которые вы выполнили, и примеры вывода (по 3–5 строк из каждого пункта).

---

## Часть 3. Настройка ротации логов (25 минут)

### Шаг 6. Создание тестового лог-файла

На `logserver` (или на любой машине) создайте файл `/var/log/myapp.log`:

1. Создайте файл и запишите в него несколько строк:
   ```
   sudo touch /var/log/myapp.log
   for i in {1..10}; do echo "Log entry number $i" | sudo tee -a /var/log/myapp.log; done
   ```

---

### Шаг 7. Создание конфигурации logrotate

1. Создайте файл `/etc/logrotate.d/myapp`:
   ```
   sudo nano /etc/logrotate.d/myapp
   ```
2. Вставьте следующее содержимое:
   ```
   /var/log/myapp.log {
       daily
       rotate 4
       compress
       missingok
       notifempty
       create 0640 root adm
       postrotate
           systemctl kill -s USR1 rsyslog >/dev/null 2>&1 || true
       endscript
   }
   ```
3. Сохраните и закройте.

---

### Шаг 8. Проверка и запуск ротации

1. Проверьте синтаксис конфигурации без выполнения:
   ```
   sudo logrotate -d /etc/logrotate.d/myapp
   ```
   Если ошибок нет, переходите к реальному запуску.

2. Принудительно выполните ротацию:
   ```
   sudo logrotate -vf /etc/logrotate.d/myapp
   ```
   Опция `-v` – подробный вывод, `-f` – принудительно.

3. Проверьте результат:
   ```
   ls -l /var/log/myapp*
   ```
   Вы должны увидеть:
   - `myapp.log` (новый, пустой или с новыми записями)
   - `myapp.log.1.gz` – сжатый предыдущий файл (если он существовал)
   - и т.д. до `myapp.log.4.gz` (если ротация была повторена несколько раз, то может быть несколько).

4. Проверьте, что исходный файл `myapp.log` пересоздан с правами `0640` и владельцем `root:adm`.

---

### Шаг 9. (Дополнительно) Настройка ротации для удалённых логов

На `logserver` создайте правило для каталога `/var/log/remote/*/*.log`:
   ```
   sudo nano /etc/logrotate.d/remote-logs
   ```
   Содержимое:
   ```
   /var/log/remote/*/*.log {
       weekly
       rotate 5
       compress
       missingok
       notifempty
       create 0644 root root
       sharedscripts
       postrotate
           systemctl kill -s USR1 rsyslog >/dev/null 2>&1 || true
       endscript
   }
   ```
   Проверьте и выполните вручную (по желанию).

---

## Часть 4. Планировщик задач (30 минут)

### Шаг 10. Создание скрипта архивации

На `logserver` создайте скрипт `/usr/local/bin/archive_logs.sh`:

1. Откройте файл:
   ```
   sudo nano /usr/local/bin/archive_logs.sh
   ```
2. Вставьте следующий код:
   ```bash
   #!/bin/bash
   BACKUP_DIR="/var/backups/logs"
   DATE=$(date +%Y%m%d_%H%M%S)
   mkdir -p $BACKUP_DIR
   tar -czf $BACKUP_DIR/logs_$DATE.tar.gz /var/log/remote/ 2>/dev/null
   find /var/log/remote/ -type f -name "*.log" -mtime +7 -delete
   echo "Archived at $DATE" >> /var/log/archive_logs.log
   ```
3. Сделайте скрипт исполняемым:
   ```
   sudo chmod +x /usr/local/bin/archive_logs.sh
   ```
4. Выполните скрипт вручную для проверки:
   ```
   sudo /usr/local/bin/archive_logs.sh
   ```
5. Проверьте, что архив создался:
   ```
   ls -l /var/backups/logs/
   ```

---

### Шаг 11. Настройка cron (основной вариант)

1. Откройте crontab для root:
   ```
   sudo crontab -e
   ```
   (при первом запуске выберите редактор, например, nano).

2. Добавьте в конец файла строку:
   ```
   0 * * * * /usr/local/bin/archive_logs.sh
   ```
   Это будет запускать скрипт каждый час в 0 минут.

3. Сохраните и закройте.

4. Проверьте, что задание добавлено:
   ```
   sudo crontab -l
   ```

5. Чтобы протестировать, можно изменить время (например, на каждую минуту для проверки):
   ```
   * * * * * /usr/local/bin/archive_logs.sh
   ```
   Подождите минуту, затем проверьте логи архивации:
   ```
   cat /var/log/archive_logs.log
   ```
   И проверьте, что создаются новые архивы в `/var/backups/logs/`.

6. **Важно:** после тестирования верните строку обратно на `0 * * * *` или удалите тестовую.

---

### Шаг 12. (Альтернатива) Настройка systemd timer

Если вы хотите попробовать современный способ, создайте таймер вместо cron (это задание на дополнительный балл).

1. Создайте служебный файл `/etc/systemd/system/archive-logs.service`:
   ```
   sudo nano /etc/systemd/system/archive-logs.service
   ```
   Содержимое:
   ```
   [Unit]
   Description=Archive remote logs

   [Service]
   Type=oneshot
   ExecStart=/usr/local/bin/archive_logs.sh
   ```
2. Создайте таймер `/etc/systemd/system/archive-logs.timer`:
   ```
   sudo nano /etc/systemd/system/archive-logs.timer
   ```
   Содержимое:
   ```
   [Unit]
   Description=Run archive logs hourly

   [Timer]
   OnCalendar=hourly
   Persistent=true

   [Install]
   WantedBy=timers.target
   ```
3. Перезагрузите systemd, включите и запустите таймер:
   ```
   sudo systemctl daemon-reload
   sudo systemctl enable --now archive-logs.timer
   ```
4. Проверьте список таймеров:
   ```
   systemctl list-timers --all | grep archive
   ```
5. Запустите сервис вручную для проверки:
   ```
   sudo systemctl start archive-logs.service
   ```
6. Посмотрите логи:
   ```
   sudo journalctl -u archive-logs.service
   ```

---

### Шаг 13. Проверка работы планировщика

- Убедитесь, что скрипт выполняется автоматически (подождите час или измените интервал для теста).
- Проверьте, что в `/var/backups/logs/` регулярно появляются новые архивы с меткой времени.
- Проверьте, что старые логи (старше 7 дней) удаляются из `/var/log/remote/` – для этого можно создать файлы с датой изменения старше 7 дней с помощью `touch -t` (это для демонстрации).

---

