# Лабораторные работы по отладке программ в Linux (с дополненной установкой пакетов)

---

# Лабораторная работа №1. Отладка программ с GDB

**Цель:** Научиться использовать GNU Debugger (GDB) для пошаговой отладки программ, поиска и исправления ошибок.

**Продолжительность:** 1,5–2 часа.

---

## Теоретическая справка

**GDB** (GNU Debugger) — мощный инструмент для отладки программ на C, C++ и других языках. Он позволяет:

- Запускать программу под контролем отладчика
- Устанавливать **точки останова** (breakpoints) в определённых местах кода
- Выполнять программу **пошагово** (step, next)
- **Просматривать и изменять** значения переменных
- **Исследовать стек вызовов** (backtrace) в любой момент
- Прикрепляться к уже **запущенному процессу** (attach)

**Ключевые команды GDB:**

| Команда | Сокращение | Назначение |
|---------|------------|------------|
| `break` | `b` | Установить точку останова |
| `run` | `r` | Запустить программу |
| `continue` | `c` | Продолжить выполнение |
| `next` | `n` | Шаг с обходом функций |
| `step` | `s` | Шаг с заходом в функции |
| `print` | `p` | Вывести значение переменной |
| `backtrace` | `bt` | Показать стек вызовов |
| `list` | `l` | Показать исходный код |
| `info locals` | — | Показать локальные переменные |
| `quit` | `q` | Выйти из GDB |

---

## Подготовка стенда

### Шаг 1. Установка необходимых пакетов

Выполните установку всех инструментов, необходимых для компиляции и отладки:

```bash
sudo apt update
sudo apt install -y gdb gcc make binutils build-essential
```

**Пояснение пакетов:**
- `gdb` — отладчик
- `gcc` — компилятор C
- `make` — система сборки
- `binutils` — набор утилит для работы с бинарными файлами (objdump, strings, readelf и др.)
- `build-essential` — мета-пакет, включающий компиляторы и базовые библиотеки

Для дополнительного анализа памяти рекомендуется установить `valgrind` (по желанию):

```bash
sudo apt install -y valgrind
```

---

### Шаг 2. Создание рабочей директории

```bash
mkdir -p ~/lab_gdb_debug
cd ~/lab_gdb_debug
```

---

## Пример программы с ошибками

Создайте файл `buggy_program.c`:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

// Функция для вычисления факториала (работает неправильно для n=0)
int factorial(int n) {
    if (n == 0) {
        return 1;  // Здесь всё правильно
    }
    return n * factorial(n - 1);  // Ошибка: рекурсия никогда не достигнет 0
}

// Функция для обработки строки (переполнение буфера)
void process_string(const char *input) {
    char buffer[10];
    strcpy(buffer, input);  // ОПАСНО! Нет проверки длины
    printf("Обработана строка: %s\n", buffer);
}

// Функция с делением на ноль
int divide(int a, int b) {
    return a / b;  // Если b == 0 — ошибка
}

int main(int argc, char *argv[]) {
    printf("=== ДЕМОНСТРАЦИЯ ОШИБОК ===\n");

    // Ошибка 1: бесконечная рекурсия при вычислении факториала
    int n = 5;
    printf("Факториал %d = %d\n", n, factorial(n));

    // Ошибка 2: переполнение буфера
    char long_string[] = "This string is way too long for a 10-byte buffer!";
    process_string(long_string);

    // Ошибка 3: деление на ноль
    int x = 10, y = 0;
    printf("Результат деления: %d\n", divide(x, y));

    return 0;
}
```

---

## Компиляция программы

Скомпилируйте программу **с отладочной информацией** (ключ `-g`) и **без оптимизации** (ключ `-O0`):

```bash
gcc -g -O0 -o buggy_program buggy_program.c
```

Для защиты от переполнения стека и других атак (чтобы увидеть реальные ошибки, а не защитные механизмы), отключите некоторые защиты:

```bash
gcc -g -O0 -fno-stack-protector -z execstack -o buggy_program buggy_program.c
```

---

## Пошаговая отладка

### Шаг 1. Запуск GDB

```bash
gdb ./buggy_program
```

Должно появиться приглашение `(gdb)`.

---

### Шаг 2. Просмотр исходного кода

```bash
(gdb) list
```

Эта команда показывает первые 10 строк исходного кода. Повторный `list` покажет следующие строки.

Чтобы посмотреть конкретную функцию:

```bash
(gdb) list factorial
```

---

### Шаг 3. Запуск программы

```bash
(gdb) run
```

**Ожидаемый результат:** Программа вызовет ошибку сегментации (Segmentation fault) при попытке записи за пределы буфера.

---

### Шаг 4. Анализ места падения

После падения используйте `backtrace` (или `bt`), чтобы увидеть стек вызовов в момент ошибки:

```bash
(gdb) bt
```

Вы увидите цепочку вызовов, которая привела к падению.

---

### Шаг 5. Установка точек останова

Перезапустите GDB и установите точки останова в ключевых местах:

```bash
(gdb) break main
(gdb) break factorial
(gdb) break process_string
(gdb) break divide
```

Проверьте установленные точки:

```bash
(gdb) info breakpoints
```

---

### Шаг 6. Пошаговое выполнение

Запустите программу заново:

```bash
(gdb) run
```

Программа остановится на `main`. Теперь можно выполнять команды пошагово:

- `next` (или `n`) — выполнить следующую строку, не заходя в функции
- `step` (или `s`) — выполнить следующую строку, заходя в функции

Пройдите до вызова `factorial`:

```bash
(gdb) n
(gdb) n
(gdb) n
```

Когда дойдёте до `printf("Факториал %d = %d\n", n, factorial(n));`, используйте `step`, чтобы войти в функцию:

```bash
(gdb) step
```

---

### Шаг 7. Просмотр переменных

Внутри функции `factorial` посмотрите значение аргумента:

```bash
(gdb) print n
```

Используйте `display`, чтобы автоматически показывать значение при остановке:

```bash
(gdb) display n
```

Теперь выполняйте пошагово и следите за значениями:

```bash
(gdb) step
(gdb) step
(gdb) step
```

Вы увидите, что рекурсия никогда не достигает 0 из-за ошибки в условии.

---

### Шаг 8. Отладка переполнения буфера

Продолжите выполнение до `process_string`:

```bash
(gdb) continue
```

Используйте `step`, чтобы войти в `process_string`:

```bash
(gdb) step
```

Посмотрите длину входной строки:

```bash
(gdb) print strlen(input)
```

Вы увидите, что строка значительно длиннее 10 байт.

Перед вызовом `strcpy` можно посмотреть адрес буфера:

```bash
(gdb) print &buffer
(gdb) print sizeof(buffer)
```

---

### Шаг 9. Наблюдение за переменными (watchpoints)

Установите **точку наблюдения** за изменением переменной:

```bash
(gdb) watch buffer
(gdb) continue
```

GDB остановится, когда содержимое `buffer` начнёт меняться.

---

### Шаг 10. Выход из GDB

```bash
(gdb) quit
```

---

## Исправление ошибок

Создайте исправленную версию программы `fixed_program.c`:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

// Исправленный факториал
int factorial(int n) {
    if (n <= 1) {   // Исправлено: условие для 0 и 1
        return 1;
    }
    return n * factorial(n - 1);
}

// Безопасная обработка строки
void process_string(const char *input) {
    char buffer[10];
    // Используем strncpy с ограничением длины
    strncpy(buffer, input, sizeof(buffer) - 1);
    buffer[sizeof(buffer) - 1] = '\0';  // Гарантируем завершающий ноль
    printf("Обработана строка: %s\n", buffer);
}

// Деление с проверкой
int divide(int a, int b) {
    if (b == 0) {
        printf("ОШИБКА: деление на ноль!\n");
        return 0;
    }
    return a / b;
}

int main(int argc, char *argv[]) {
    printf("=== ИСПРАВЛЕННАЯ ПРОГРАММА ===\n");

    int n = 5;
    printf("Факториал %d = %d\n", n, factorial(n));

    char long_string[] = "This string is way too long for a 10-byte buffer!";
    process_string(long_string);

    int x = 10, y = 0;
    printf("Результат деления: %d\n", divide(x, y));

    return 0;
}
```

Скомпилируйте исправленную программу:

```bash
gcc -g -O0 -o fixed_program fixed_program.c
```

Запустите и убедитесь, что ошибки устранены.

---

## Контрольные вопросы

1. Для чего используется ключ `-g` при компиляции?
2. В чём разница между командами `step` и `next` в GDB?
3. Как вывести значение переменной в GDB?
4. Что показывает команда `backtrace`?
5. Как установить точку останова в определённой строке кода?
6. Что такое watchpoint и для чего он нужен?

---

# Лабораторная работа №2. Отладка программ с помощью core-файлов и трассировки

**Цель:** Научиться анализировать core-файлы для диагностики причин падения программ, использовать системные средства трассировки (strace, ltrace) для выявления проблем.

**Продолжительность:** 1,5–2 часа.

---

## Теоретическая справка

**Core-файл** (дамп памяти) — это снимок состояния работающей программы в момент её аварийного завершения (Segmentation Fault, Floating Point Exception и др.). Он содержит содержимое памяти, регистры процессора, стек вызовов и другую отладочную информацию.

**strace** — утилита для трассировки системных вызовов процессов. Позволяет видеть, какие системные вызовы (open, read, write, etc.) выполняет программа.

**ltrace** — утилита для трассировки вызовов библиотечных функций.

---

## Подготовка стенда

### Шаг 1. Установка необходимых пакетов

Выполните установку всех инструментов:

```bash
sudo apt update
sudo apt install -y gdb gcc make binutils build-essential strace ltrace
```

**Пояснение пакетов:**
- `gdb`, `gcc`, `make`, `binutils` — как в первой лабораторной
- `strace` — трассировка системных вызовов
- `ltrace` — трассировка библиотечных вызовов
- `build-essential` — базовые инструменты компиляции

---

### Шаг 2. Настройка генерации core-файлов

По умолчанию core-файлы могут быть отключены. Включите их:

```bash
# Проверка текущего лимита
ulimit -c

# Установка неограниченного размера core-файлов
ulimit -c unlimited

# Для постоянной настройки добавьте в ~/.bashrc или /etc/security/limits.conf
```

**Настройка пути сохранения core-файлов:**

```bash
# Просмотр текущих настроек
cat /proc/sys/kernel/core_pattern

# Установка имен core-файлов с именем программы и PID
echo "core.%e.%p" | sudo tee /proc/sys/kernel/core_pattern
```

Для постоянной настройки отредактируйте `/etc/sysctl.conf`:

```bash
echo "kernel.core_pattern=core.%e.%p" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

---

### Шаг 3. Создание рабочей директории

```bash
mkdir -p ~/lab_core_debug
cd ~/lab_core_debug
```

---

## Пример программы с ошибками (для core-файла)

Создайте файл `crash_program.c`:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

// Ошибка: разыменование нулевого указателя
void cause_null_pointer() {
    int *ptr = NULL;
    *ptr = 42;  // Segmentation Fault
}

// Ошибка: запись за пределы выделенной памяти
void cause_buffer_overflow() {
    char *buffer = malloc(10);
    if (!buffer) return;
    strcpy(buffer, "This is a very long string that will overflow");
    free(buffer);
}

// Ошибка: деление на ноль (SIGFPE)
void cause_divide_by_zero() {
    int a = 10;
    int b = 0;
    printf("Результат: %d\n", a / b);
}

// Ошибка: использование после освобождения
void cause_use_after_free() {
    char *ptr = malloc(20);
    free(ptr);
    strcpy(ptr, "Hello");  // Use-after-free
}

int main(int argc, char *argv[]) {
    printf("=== ПРОГРАММА С ОШИБКАМИ ДЛЯ CORE-ФАЙЛА ===\n");

    int choice = 0;
    if (argc > 1) {
        choice = atoi(argv[1]);
    }

    switch (choice) {
        case 1:
            cause_null_pointer();
            break;
        case 2:
            cause_buffer_overflow();
            break;
        case 3:
            cause_divide_by_zero();
            break;
        case 4:
            cause_use_after_free();
            break;
        default:
            printf("Использование: %s <1|2|3|4>\n", argv[0]);
            printf("  1 - Null pointer\n");
            printf("  2 - Buffer overflow\n");
            printf("  3 - Divide by zero\n");
            printf("  4 - Use after free\n");
    }

    return 0;
}
```

---

## Компиляция программы

Скомпилируйте с отладочной информацией:

```bash
gcc -g -O0 -o crash_program crash_program.c
```

---

## Генерация core-файлов

### Шаг 1. Запуск для каждого типа ошибки

```bash
# Запуск с аргументом 1 (null pointer)
./crash_program 1
```

**Ожидаемый результат:** Segmentation Fault, в текущей директории появится файл `core.crash_program.12345` (номер процесса будет отличаться).

Повторите для других типов ошибок:

```bash
./crash_program 2   # buffer overflow
./crash_program 3   # divide by zero
./crash_program 4   # use after free
```

---

## Анализ core-файлов

### Шаг 2. Загрузка core-файла в GDB

```bash
gdb ./crash_program core.crash_program.*
```

Или укажите конкретный файл:

```bash
gdb ./crash_program core.crash_program.12345
```

---

### Шаг 3. Просмотр стека вызовов

В GDB выполните:

```bash
(gdb) bt
```

Вы увидите стек вызовов в момент падения. Определите, в какой функции произошла ошибка.

---

### Шаг 4. Анализ регистров и переменных

```bash
# Просмотр значений регистров
(gdb) info registers

# Переход к определённому фрейму стека
(gdb) frame 0   # перейти к верхнему фрейму

# Просмотр локальных переменных
(gdb) info locals

# Просмотр значения конкретной переменной
(gdb) print <variable_name>
```

---

### Шаг 5. Определение причины падения

Для каждого типа ошибки GDB покажет конкретную причину:

- **Null pointer dereference:** `Program received signal SIGSEGV, Segmentation fault.`
- **Buffer overflow:** Ошибка в `strcpy` с нарушением границ памяти.
- **Divide by zero:** `Program received signal SIGFPE, Arithmetic exception.`
- **Use after free:** Ошибка доступа к освобождённой памяти.

---

## Использование strace и ltrace

### Шаг 6. Трассировка системных вызовов (strace)

Запустите программу с `strace`, чтобы увидеть все системные вызовы:

```bash
strace ./crash_program 1 2>&1 | head -50
```

Фильтрация только системных вызовов, связанных с файлами:

```bash
strace -e file ./crash_program 1
```

Сохранение вывода в файл:

```bash
strace -o trace.log ./crash_program 1
```

---

### Шаг 7. Трассировка библиотечных вызовов (ltrace)

```bash
ltrace ./crash_program 1 2>&1 | head -30
```

Фильтрация вызовов конкретных функций:

```bash
ltrace -e malloc+free ./crash_program 2
```

---

### Шаг 8. Трассировка уже запущенного процесса

Запустите программу в фоне:

```bash
./crash_program 1 &
PID=$!
```

Подключитесь к процессу с помощью strace:

```bash
sudo strace -p $PID
```

---

## Исправление ошибок

Создайте исправленную версию `fixed_crash_program.c`:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

void cause_null_pointer() {
    int *ptr = NULL;
    // Проверка перед разыменованием
    if (ptr != NULL) {
        *ptr = 42;
    } else {
        printf("Ошибка: нулевой указатель!\n");
    }
}

void cause_buffer_overflow() {
    char *buffer = malloc(10);
    if (!buffer) {
        printf("Ошибка выделения памяти\n");
        return;
    }
    // Используем strncpy с ограничением
    strncpy(buffer, "This is a very long string that will overflow", 9);
    buffer[9] = '\0';
    printf("Буфер: %s\n", buffer);
    free(buffer);
}

void cause_divide_by_zero() {
    int a = 10;
    int b = 0;
    if (b == 0) {
        printf("Ошибка: деление на ноль!\n");
        return;
    }
    printf("Результат: %d\n", a / b);
}

void cause_use_after_free() {
    char *ptr = malloc(20);
    if (!ptr) return;
    free(ptr);
    // Больше не используем ptr после освобождения
    ptr = NULL;
    // Теперь попытка доступа к ptr вызовет ошибку нулевого указателя, которую можно проверить
    if (ptr != NULL) {
        strcpy(ptr, "Hello");
    } else {
        printf("Ошибка: использование после освобождения предотвращено\n");
    }
}

int main(int argc, char *argv[]) {
    printf("=== ИСПРАВЛЕННАЯ ПРОГРАММА ===\n");

    int choice = 0;
    if (argc > 1) {
        choice = atoi(argv[1]);
    }

    switch (choice) {
        case 1: cause_null_pointer(); break;
        case 2: cause_buffer_overflow(); break;
        case 3: cause_divide_by_zero(); break;
        case 4: cause_use_after_free(); break;
        default:
            printf("Использование: %s <1|2|3|4>\n", argv[0]);
    }
    return 0;
}
```

Скомпилируйте и проверьте:

```bash
gcc -g -O0 -o fixed_crash_program fixed_crash_program.c
./fixed_crash_program 1
```

---

## Полезные команды для работы с core-файлами

| Команда | Назначение |
|---------|------------|
| `file core.*` | Определить, какой программе принадлежит core-файл |
| `strings core.* | grep -i error` | Поиск текстовых строк в core-файле |
| `gdb ./program core.*` | Загрузка core-файла в GDB |
| `objdump -d ./program` | Дизассемблирование программы для анализа кода |

---
