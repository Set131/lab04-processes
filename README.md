# Лабораторна робота №4: Процеси, системні виклики та fork() / exec() / wait()

## 1. Мета лабораторної роботи

У Lab 3 ми простежили шлях: Source Code → Object File → ELF Executable → Loader/OS → Process. Тепер дослідимо, що відбувається **після** запуску executable.

**Центральні питання:**
- Як операційна система представляє та керує процесом?
- Як програма взаємодіє з kernel через system calls?
- Як один process створює інший process і запускає в ньому іншу програму?

**Досліджувані концепції:** Program vs Process, PID/PPID, System Calls, Process States, fork(), Parent/Child, Process Memory, exec(), wait(), Exit Status, Zombie Process.

---

## 2. Середовище

- **WSL 2**, Ubuntu 24.04 LTS (resolute)
- **gcc 15.2.0**
- **binutils 2.46**, **strace**
- Архітектура: **x86-64 (AMD64)**
- Хост: Windows 11 Pro

---

## 3. Завдання 1: Program та Process

### Програма `process_info.c`

```c
#include <stdio.h>
#include <unistd.h>

int main(void)
{
    printf("PID:  %d\n", getpid());
    printf("PPID: %d\n", getppid());
    sleep(30);
    return 0;
}
```

### Виконані команди

```bash
gcc -Wall -Wextra process_info.c -o process_info
./process_info
# PID:  1672
# PPID: 347
```

### Перевірка в другому терміналі

```bash
ps -p 1672
#     PID TTY          TIME CMD
#    1672 pts/0    00:00:00 process_info

file process_info
# ELF 64-bit LSB pie executable, x86-64, dynamically linked, ...
```

### Відповіді на питання

**1. Де знаходиться executable до запуску?**
На диску, у файловій системі, як звичайний файл `process_info`. Це ELF executable.

**2. Що таке PID?**
Process ID — унікальний числовий ідентифікатор процесу в системі, який видає kernel при створенні процесу.

**3. Чому PID не є властивістю самого executable?**
Executable — це статичний файл на диску. PID виникає лише тоді, коли ОС завантажує executable в пам'ять і створює process. Один executable може мати багато PID (при багатьох запусках).

**4. Що таке PPID?**
Parent PID — PID процесу, який створив даний процес (через fork() або shell).

**5. Який process є parent для вашої програми?**
Зазвичай це shell (bash), з якого ви запустили `./process_info`. PPID = PID цього shell (347).

---

## 4. Завдання 2: Один program — декілька processes

### Запуск трьох instances

| Instance | PID | PPID |
|----------|-----|------|
| Process 1 | 1372 | 347 |
| Process 2 | 1392 | 1019 |
| Process 3 | 1412 | 1249 |

(PPID різні, бо запускалися з різних терміналів; якщо запустити з одного — PPID однаковий.)

### Пояснення

Один executable `process_info` створив **три різні PID**, бо кожен запуск — це окремий process з власним address space, PID, file descriptors тощо.

**Program vs Process:**
- **Program** — статичний файл з інструкціями на диску (executable).
- **Process** — запущений екземпляр програми в пам'яті з власним станом, PID, ресурсами.

---

## 5. Завдання 3: Спостереження через /proc

### Виконані команди

```bash
ls /proc/1672
tr '\0' ' ' < /proc/1672/cmdline; echo
cat /proc/1672/status
readlink /proc/1672/exe
head -n 5 /proc/1672/maps
```

### Результати

```
Name:   process_info
Umask:  0022
State:  S (sleeping)
Tgid:   1672
Pid:    1672
PPid:   347
...

readlink: /home/user/labs/lab04-processes/process_info
cmdline:  ./process_info

maps (перші 5 рядків):
62d1ec3b8000-62d1ec3b9000 r--p 00000000 ... /home/user/labs/lab04-processes/process_info
62d1ec3b9000-62d1ec3ba000 r-xp 00001000 ... /home/user/labs/lab04-processes/process_info
62d1ec3ba000-62d1ec3bb000 r--p 00002000 ... /home/user/labs/lab04-processes/process_info
62d1ec3bb000-62d1ec3bc000 r--p 00002000 ... /home/user/labs/lab04-processes/process_info
62d1ec3bc000-62d1ec3bd000 rw-p 00003000 ... /home/user/labs/lab04-processes/process_info
```

### Заповнена таблиця

| Інформація | Результат |
|------------|-----------|
| Process name | `process_info` |
| PID | `1672` |
| PPID | `347` |
| State | `S (sleeping)` |
| Executable | `/home/user/labs/lab04-processes/process_info` |

### Аналіз `/proc/PID/maps`

| Адреса | Права | Що це |
|--------|-------|-------|
| `...b8000-b9000` | `r--p` | `.rodata` (read-only) |
| `...b9000-ba000` | `r-xp` | `.text` (read + execute) |
| `...ba000-bb000` | `r--p` | `.rodata` (релокації) |
| `...bb000-bc000` | `r--p` | `.rodata` (GOT) |
| `...bc000-bd000` | `rw-p` | `.data` + `.bss` (read + write) |

Це прямо підтверджує матеріал Lab 3 про ELF-секції — тепер вони видні у **живому** process через `/proc/PID/maps`.

### Відповіді на питання

**1. Чи `/proc/PID` є самим process?**
Ні. `/proc/PID` — це віртуальна файлова система, через яку kernel **експортує інформацію** про process. Це не сам process, а "вікно" в його стан.

**2. Що відбувається з `/proc/PID`, коли process завершується?**
Директорія `/proc/PID` зникає — process більше не існує, і kernel прибирає запис.

**3. Чому `/proc` корисний для дослідження OS concepts?**
Дозволяє спостерігати внутрішній стан process (PID, PPID, state, memory maps, file descriptors, cmdline) без спеціальних інструментів — це "прозорість" kernel.

---

## 6. Завдання 4: System Calls — `hello.c`

### Програма `hello.c`

```c
#include <stdio.h>

int main(void)
{
    printf("Hello from process!\n");
    return 0;
}
```

### Виконані команди

```bash
gcc -Wall -Wextra hello.c -o hello
./hello
strace -e write ./hello
```

### Результат

```
write(1, "Hello from process!\n", 20) = 20
```

### Інтерпретація `write`

```
write(
    1,                          // file descriptor stdout
    "Hello from process!\n",    // buffer (адреса даних)
    20                          // кількість байт
) = 20                          // реально записано байт
```

### Шлях від `printf()` до kernel

```
printf()
   ↓
C Library (stdio, buffering)
   ↓
write
   ↓
System Call (syscall instruction)
   ↓
══════════════════
   ↓
Kernel
   ↓
stdout (terminal)
```

---

## 7. Завдання 5: Library function vs System Call

### Програма `output.c`

```c
#include <stdio.h>
#include <unistd.h>

int main(void)
{
    printf("Output using printf\n");

    const char message[] = "Output using write\n";

    write(STDOUT_FILENO, message, sizeof(message) - 1);

    return 0;
}
```

### Виконані команди

```bash
gcc -Wall -Wextra output.c -o output
strace -e write ./output
```

### Результат

```
write(1, "Output using printf\n", 20) = 20
write(1, "Output using write\n", 19)  = 19
```

### Відповіді на питання

**1. Які write calls бачите?**
Два: `write(1, "Output using printf\n", 20)` і `write(1, "Output using write\n", 19)`.

**2. Чим концептуально відрізняються `printf()` і `write()`?**
- `printf()` — **library function** (C standard library). Виконується в user space, використовує buffering, форматування. Під капотом викликає `write()`.
- `write()` — **system call** (POSIX). Переходить у kernel space, записує байти безпосередньо у file descriptor. Без buffering.

**Важливо:** `printf()` не завжди породжує рівно один `write()` — все залежить від buffering (line-buffered для terminal, fully-buffered для file).

---

## 8. Завдання 6: Перший `fork()`

### Програма `fork_basic.c`

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(void)
{
    printf("Before fork: PID=%d\n", getpid());
    fflush(stdout);

    pid_t pid = fork();

    if (pid < 0) {
        perror("fork");
        return EXIT_FAILURE;
    }

    if (pid == 0) {
        printf("CHILD:  PID=%d, PPID=%d, fork returned=%d\n",
               getpid(), getppid(), pid);
    } else {
        printf("PARENT: PID=%d, child PID=%d, fork returned=%d\n",
               getpid(), pid, pid);
    }

    return EXIT_SUCCESS;
}
```

### Виконані команди

```bash
gcc -Wall -Wextra fork_basic.c -o fork_basic
./fork_basic
./fork_basic
./fork_basic
```

### Результати (3 запуски)

| Запуск | Parent PID | Child PID | Child PPID | `fork()` у parent | `fork()` у child |
|--------|-----------|-----------|------------|-------------------|------------------|
| 1 | 1904 | 1905 | 1904 | 1905 (> 0) | 0 |
| 2 | 1911 | 1912 | **343** | 1912 (> 0) | 0 |
| 3 | 1918 | 1919 | 1918 | 1919 (> 0) | 0 |

**Зверніть увагу на запуск 2:** Child PPID = **343**, а не 1911. Це тому, що parent (1911) завершився **раніше**, ніж child встиг викликати `getppid()`. Після завершення parent child "усиновлюється" init-процесом (у WSL — PID 343). Це нормальна поведінка Unix.

### Відповіді на питання

**1. Скільки processes існувало до fork()?**
Один (parent).

**2. Скільки існує після успішного fork()?**
Два (parent + child).

**3. Яке значення fork() отримав parent?**
PID child (> 0), наприклад 1905.

**4. Яке значення отримав child?**
0.

**5. Чому одна й та сама строка `pid_t pid = fork();` призводить до двох різних continuation paths?**
`fork()` — це системний виклик, який повертає **різні значення** в parent і child. Kernel повертає PID child у parent (щоб parent знав, кого чекати) і 0 у child (щоб child знав, що він — child). Обидва процеси продовжують виконання з однієї точки, але з різними значеннями `pid`, тому `if (pid == 0)` розгалужує потік.

### Модель fork()

```
             Process
                │
              fork()
                │
          ┌─────┴─────┐
          ▼           ▼
       Parent       Child
       pid > 0      pid == 0
```

---

## 9. Завдання 7: Чи можна передбачити порядок?

### Запуск кілька разів

```bash
./fork_basic
./fork_basic
./fork_basic
```

Порядок `PARENT`/`CHILD` може змінюватися.

### Відповідь

**Чи можна гарантувати, який process надрукує повідомлення першим?**
**Ні.** Після `fork()` parent та child — окремі schedulable processes. Порядок виконання визначається scheduler ядра, а не порядком у коді. Це перший практичний контакт із concurrency.

---

## 10. Завдання 8: `fork()` та незалежність пам'яті

### Програма `fork_memory.c`

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int value = 100;

int main(void)
{
    printf("Before fork: value=%d, address=%p\n", value, (void *)&value);
    fflush(stdout);

    pid_t pid = fork();

    if (pid < 0) {
        perror("fork");
        return EXIT_FAILURE;
    }

    if (pid == 0) {
        value = 200;
        printf("CHILD:  PID=%d value=%d address=%p\n",
               getpid(), value, (void *)&value);
    } else {
        value = 300;
        printf("PARENT: PID=%d value=%d address=%p\n",
               getpid(), value, (void *)&value);
    }

    return EXIT_SUCCESS;
}
```

### Виконані команди

```bash
gcc -Wall -Wextra fork_memory.c -o fork_memory
./fork_memory
```

### Результат

```
Before fork: value=100, address=0x5ae1abe1d010
PARENT: PID=1998 value=300 address=0x5ae1abe1d010
CHILD:  PID=1999 value=200 address=0x5ae1abe1d010
```

### Заповнена таблиця

| Процес | PID | `value` | `&value` |
|--------|-----|---------|----------|
| Before fork | — | 100 | `0x5ae1abe1d010` |
| Parent | 1998 | 300 | `0x5ae1abe1d010` |
| Child | 1999 | 200 | `0x5ae1abe1d010` |

### Висновок

`&value` має **однакову** віртуальну адресу в обох процесах, але `value` **різний**: 300 у parent, 200 у child. Parent і child **не бачать** змін один одного.

**Чому?** Після `fork()` kernel створює **копію** address space (copy-on-write). Однакова віртуальна адреса в двох процесах відображається на **різні фізичні сторінки**. Це важливий перехід до теми virtual memory: однакова virtual address ≠ спільна змінювана пам'ять.

---

## 11. Завдання 9: `strace` для fork()

### Виконана команда

```bash
strace -f -e trace=process ./fork_basic
```

### Ключові рядки

```
execve("./fork_basic", ["./fork_basic"], ...) = 0
Before fork: PID=1940
clone(child_stack=NULL, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0x7ec518b1da10) = 1941
PARENT: PID=1940, child PID=1941, fork returned=1941
[pid 1940] exit_group(0) = ?
[pid 1940] +++ exited with 0 +++
CHILD:  PID=1941, PPID=1940, fork returned=0
exit_group(0) = ?
+++ exited with 0 +++
```

### Аналіз

- C API `fork()` під капотом викликає kernel interface **`clone`** (або `clone3` на новіших kernel), а не буквальний `fork`
- `= 1941` — PID child, який повернувся в parent
- `SIGCHLD` — сигнал, який kernel надішле parent, коли child завершиться
- `[pid 1940]` та `[pid 1941]` — `strace -f` слідкує за обома процесами окремо
- Порядок: parent завершився (`[pid 1940] exit_group(0)`) **раніше**, ніж child надрукував своє повідомлення — це ілюструє scheduling

**Важливе спостереження:** library/API (`fork()`) та конкретний kernel interface (`clone`) не завжди мають ідентичні назви.

---

## 12. Завдання 10: `wait()`

### Програма `wait_basic.c`

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/wait.h>
#include <unistd.h>

int main(void)
{
    pid_t pid = fork();

    if (pid < 0) {
        perror("fork");
        return EXIT_FAILURE;
    }

    if (pid == 0) {
        printf("CHILD: working...\n");
        sleep(2);
        printf("CHILD: finished\n");
        return 42;
    }

    printf("PARENT: waiting for child %d\n", pid);

    int status;
    if (wait(&status) < 0) {
        perror("wait");
        return EXIT_FAILURE;
    }

    printf("PARENT: child finished\n");

    if (WIFEXITED(status)) {
        printf("PARENT: child exit status = %d\n", WEXITSTATUS(status));
    }

    return EXIT_SUCCESS;
}
```

### Виконані команди

```bash
gcc -Wall -Wextra wait_basic.c -o wait_basic
./wait_basic
```

### Результат

```
PARENT: waiting for child 2055
CHILD: working...
CHILD: finished
PARENT: child finished
PARENT: child exit status = 42
```

### Відповіді на питання

**1. Що робить parent під час `wait()`?**
Блокується (переходить у стан waiting/sleeping), поки child не завершиться.

**2. Чому `PARENT: child finished` не з'являється раніше?**
Бо `wait()` не повертається, поки child не завершиться.

**3. Що означає 42?**
Це exit status child (значення, яке child повернув з `main`).

**4. Чи 42 є PID?**
Ні, це код повернення (exit status), а не PID. PID child — 2055.

**5. Навіщо `WIFEXITED` та `WEXITSTATUS`?**
`WIFEXITED` перевіряє, що child завершився нормально (а не через сигнал). `WEXITSTATUS` витягує код повернення з `status`. Без них не можна коректно інтерпретувати `status`.

---

## 13. Завдання 11: `exec()`

### Програма `exec_basic.c`

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(void)
{
    printf("Before exec: PID=%d\n", getpid());
    fflush(stdout);

    execlp("echo", "echo", "Hello from exec!", NULL);

    perror("execlp");
    return EXIT_FAILURE;
}
```

### Виконані команди

```bash
gcc -Wall -Wextra exec_basic.c -o exec_basic
./exec_basic
```

### Результат

```
Before exec: PID=2110
Hello from exec!
```

`perror("execlp")` **не виконався** — exec успішний, program image замінено на `echo`.

### Перевірка PID після exec

Створено `new_program.c` (виводить свій PID) і `exec_pid.c` (запускає `new_program` через `execl`):

```
Before exec: PID=2217
New program PID=2217
```

**PID однаковий (2217) до і після exec** — це головна властивість `exec()`.

### Висновок

| Питання | Відповідь |
|---------|-----------|
| New process? | **NO** (PID той самий) |
| New program? | **YES** (код замінено) |
| Same PID? | **YES** |

`exec()` замінює program image поточного process новою програмою. Process (PID 2217) **залишається**, але тепер виконує код `new_program`, а не `exec_pid`.

---

## 14. Завдання 12: `fork() + exec()`

### Програма `fork_exec.c`

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/wait.h>
#include <unistd.h>

int main(void)
{
    printf("PARENT: PID=%d\n", getpid());
    fflush(stdout);

    pid_t pid = fork();

    if (pid < 0) {
        perror("fork");
        return EXIT_FAILURE;
    }

    if (pid == 0) {
        printf("CHILD before exec: PID=%d\n", getpid());
        fflush(stdout);

        execlp("echo", "echo", "Program started by child", NULL);

        perror("execlp");
        _exit(EXIT_FAILURE);
    }

    printf("PARENT: created child PID=%d\n", pid);
    wait(NULL);
    printf("PARENT: child completed\n");

    return EXIT_SUCCESS;
}
```

### Виконані команди

```bash
gcc -Wall -Wextra fork_exec.c -o fork_exec
./fork_exec
```

### Результат

```
PARENT: PID=2272
PARENT: created child PID=2273
CHILD before exec: PID=2273
Program started by child
PARENT: child completed
```

### Схема fork → exec → wait

```
Parent (2272)
   │
 fork()
   │
   ├──► Parent (2272) ──► wait() ──► "child completed"
   │
   └──► Child (2273) ──► exec("echo") ──► "Program started by child" ──► exit
```

### Пояснення етапів

- **`fork()`** — створює child process (PID 2273), який є копією parent
- **`exec()`** — замінює program image child на `echo` (PID 2273 залишається)
- **`wait()`** — parent блокується, поки child не завершиться

---

## 15. Завдання 13: `launcher.c`

### Програма `launcher.c`

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/wait.h>
#include <unistd.h>

int main(int argc, char *argv[])
{
    if (argc != 2) {
        fprintf(stderr, "Usage: %s <program>\n", argv[0]);
        return EXIT_FAILURE;
    }

    pid_t pid = fork();

    if (pid < 0) {
        perror("fork");
        return EXIT_FAILURE;
    }

    if (pid == 0) {
        execlp(argv[1], argv[1], NULL);
        perror("execlp");
        _exit(EXIT_FAILURE);
    }

    printf("Started child PID=%d\n", pid);

    int status;
    if (waitpid(pid, &status, 0) < 0) {
        perror("waitpid");
        return EXIT_FAILURE;
    }

    if (WIFEXITED(status)) {
        printf("Child exit status: %d\n", WEXITSTATUS(status));
    }

    return EXIT_SUCCESS;
}
```

### Виконані команди

```bash
gcc -Wall -Wextra launcher.c -o launcher
./launcher date
./launcher whoami
./launcher uname
```

### Результати

| Команда | Started child PID | Вивід | Exit status |
|---------|-------------------|-------|-------------|
| `./launcher date` | 2359 | `Sat Nov 21 16:41:58 UTC 2026` | 0 |
| `./launcher whoami` | 2366 | `user` | 0 |
| `./launcher uname` | 2373 | `Linux` | 0 |

### Що це показує

`launcher` реалізує фрагмент поведінки shell: `fork() → child → exec(program) → parent → wait() → show result`. Кожен запуск створює новий child з унікальним PID. `waitpid()` дочекався завершення child і повернув exit status (0 = успіх).

---

## 16. Завдання 14: Process States

### Програма `states.c`

```c
#include <stdio.h>
#include <unistd.h>

int main(void)
{
    printf("PID=%d\n", getpid());
    fflush(stdout);
    sleep(60);
    return 0;
}
```

### Виконані команди

```bash
gcc -Wall -Wextra states.c -o states
./states
# PID=2428

# У другому терміналі:
cat /proc/2428/status | grep State
ps -o pid,ppid,state,cmd -p 2428
```

### Результат

```
State:  S (sleeping)
    PID    PPID S CMD
   2428     347 S ./states
```

### Відповіді на питання

**1. Під час `sleep()` process активно використовує CPU?**
Ні — він у стані `S` (sleeping/interruptible), CPU не витрачається.

**2. Running?**
Ні — `R` був би, якби процес виконував інструкції.

**3. Waiting/sleeping?**
Так — `S` означає очікування на timer або подію.

**4. Чому OS не витрачає CPU на process, який лише очікує?**
Scheduler переключає CPU на інші готові процеси; коли timer спрацює, kernel "розбудить" process і поверне його в чергу готових.

---

## 17. Завдання 15: Zombie Process

### Програма `zombie.c` (без `wait()`)

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(void)
{
    pid_t pid = fork();

    if (pid < 0) {
        perror("fork");
        return EXIT_FAILURE;
    }

    if (pid == 0) {
        printf("CHILD: PID=%d, exiting now\n", getpid());
        return 0;
    }

    printf("PARENT: PID=%d, child=%d\n", getpid(), pid);
    printf("PARENT: sleeping without wait()\n");
    sleep(60);
    return 0;
}
```

### Виконані команди

```bash
gcc -Wall -Wextra zombie.c -o zombie
./zombie
# PARENT: PID=2556, child=2557
# PARENT: sleeping without wait()
# CHILD: PID=2557, exiting now

# У другому терміналі:
ps -o pid,ppid,state,cmd -p 2557
cat /proc/2557/status | grep State
```

### Результат

```
    PID    PPID S CMD
   2557    2556 Z [zombie] <defunct>
State:  Z (zombie)
```

### Пояснення

- `State: Z (zombie)` — child завершився, але exit status ще не зібрано parent
- `[zombie] <defunct>` — запис про завершення залишається в таблиці процесів
- Zombie **не виконує інструкції** і **не використовує CPU** — це лише запис у таблиці
- Parent (2556) спить без `wait()`, тому zombie не зникає

### Виправлення: `zombie_fixed.c`

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/wait.h>
#include <unistd.h>

int main(void)
{
    pid_t pid = fork();

    if (pid < 0) {
        perror("fork");
        return EXIT_FAILURE;
    }

    if (pid == 0) {
        printf("CHILD: PID=%d, exiting now\n", getpid());
        return 0;
    }

    printf("PARENT: PID=%d, child=%d\n", getpid(), pid);
    wait(NULL);
    printf("PARENT: child reaped\n");
    sleep(10);
    return 0;
}
```

### Результат `zombie_fixed`

```
PARENT: PID=2669, child=2670
CHILD: PID=2670, exiting now
PARENT: child reaped
```

Zombie **зник** — `wait(NULL)` зібрав exit status.

### Порівняння

| Версія | Що робить | Результат |
|--------|-----------|-----------|
| `zombie.c` | `sleep(60)` без `wait()` | `State: Z (zombie)` |
| `zombie_fixed.c` | `wait(NULL)` | Zombie зникає |

---

## 18. Завдання 16: Два `fork()` — process tree

### Програма `fork_tree.c`

```c
#include <stdio.h>
#include <unistd.h>

int main(void)
{
    fork();
    fork();
    printf("PID=%d\n", getpid());
    return 0;
}
```

### Прогноз

Скільки разів виконається `printf()`? **4 рази** (2² = 4 процеси).

### Виконані команди

```bash
gcc -Wall -Wextra fork_tree.c -o fork_tree
./fork_tree
./fork_tree | sort | uniq -c
```

### Результати

**Перший запуск:**
```
PID=2735
PID=2737
PID=2736
PID=2738
```

**Другий запуск:**
```
      1 PID=2744
      1 PID=2747
      1 PID=2748
      1 PID=2749
```

### Process tree

```
main (P0)
 ├── fork() → P1
 │    └── fork() → P3
 └── fork() → P2
      └── fork() → P4
```

4 процеси, кожен виконує `printf()` рівно один раз.

### Формула

Для n послідовних `fork()`, які виконуються всіма створеними процесами, кількість процесів = **2ⁿ**:
- 1 fork → 2 processes
- 2 forks → 4 processes
- 3 forks → 8 processes

---

## 19. Завдання 17: Фінальна програма `process_runner.c`

### Програма `process_runner.c`

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/wait.h>
#include <unistd.h>

int main(int argc, char *argv[])
{
    if (argc != 2) {
        fprintf(stderr, "Usage: %s PROGRAM\n", argv[0]);
        return EXIT_FAILURE;
    }

    printf("Parent PID: %d\n", getpid());
    printf("Creating child...\n");
    fflush(stdout);

    pid_t pid = fork();

    if (pid < 0) {
        perror("fork");
        return EXIT_FAILURE;
    }

    if (pid == 0) {
        printf("Child PID: %d\n", getpid());
        printf("Child PPID: %d\n", getppid());
        printf("Executing: %s\n", argv[1]);
        fflush(stdout);

        execlp(argv[1], argv[1], NULL);

        perror("execlp");
        _exit(EXIT_FAILURE);
    }

    printf("\nParent: child PID = %d\n", pid);
    fflush(stdout);

    int status;
    if (waitpid(pid, &status, 0) < 0) {
        perror("waitpid");
        return EXIT_FAILURE;
    }

    printf("\nChild %d completed\n", pid);

    if (WIFEXITED(status)) {
        printf("Exit status: %d\n", WEXITSTATUS(status));
    } else if (WIFSIGNALED(status)) {
        printf("Child terminated by signal %d\n", WTERMSIG(status));
    }

    printf("\nParent continues\n");
    return EXIT_SUCCESS;
}
```

### Виконані команди

```bash
gcc -Wall -Wextra process_runner.c -o process_runner
./process_runner date
./process_runner whoami
```

### Результат `./process_runner date`

```
Parent PID: 2807
Creating child...

Parent: child PID = 2808
Child PID: 2808
Child PPID: 2807
Executing: date
Sat Nov 21 16:47:54 UTC 2026

Child 2808 completed
Exit status: 0

Parent continues
```

### Результат `./process_runner whoami`

```
Parent PID: 2814
Creating child...

Parent: child PID = 2815
Child PID: 2815
Child PPID: 2814
Executing: whoami
user

Child 2815 completed
Exit status: 0

Parent continues
```

### Що це показує

- Parent (2807) створив child (2808) через `fork()`
- Child виконав `exec("date")` — PID 2808 залишився, програма замінена на `date`
- Parent дочекався через `waitpid()` і отримав exit status 0
- `Parent continues` — parent продовжив після завершення child

---

## 20. Завдання 18: `strace` для фінальної програми

### Виконана команда

```bash
strace -f -e trace=process ./process_runner date
```

### Три фрагменти trace

#### 1. Створення process (`clone`)

```
clone(child_stack=NULL, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0x7f359c86ba10) = 2909
```

C API `fork()` під капотом викликає kernel interface **`clone`** (не буквальний `fork`). Повертає PID child (2909) у parent. `SIGCHLD` — сигнал, який kernel надішле parent, коли child завершиться.

#### 2. Запуск нового program image (`execve`)

```
[pid 2909] execve("/home/user/.local/share/mise/shims/date", ["date"], ...) = -1 ENOENT
[pid 2909] execve("/home/user/.local/bin/date", ["date"], ...) = -1 ENOENT
[pid 2909] execve("/usr/local/sbin/date", ["date"], ...) = -1 ENOENT
[pid 2909] execve("/usr/local/bin/date", ["date"], ...) = -1 ENOENT
[pid 2909] execve("/usr/sbin/date", ["date"], ...) = -1 ENOENT
[pid 2909] execve("/usr/bin/date", ["date"], ...) = 0
```

Це `execlp` шукає `date` у `PATH`. Перші шість спроб — `ENOENT` (файл не знайдено), сьома (`/usr/bin/date`) — успіх (`= 0`). PID 2909 залишився тим самим — `exec` не створив новий process.

#### 3. Очікування/завершення child (`wait4`)

```
[pid 2908] wait4(2909, <unfinished ...>
[pid 2909] exit_group(0) = ?
[pid 2909] +++ exited with 0 +++
<... wait4 resumed>, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], 0, NULL) = 2909
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=2909, si_status=0, ...} ---
```

Parent (2908) викликав `wait4(2909, ...)`, який **блокувався** (`<unfinished ...>`), поки child не завершився (`exit_group(0)`). Потім `wait4` повернув PID child (2909) і status `WIFEXITED(s) && WEXITSTATUS(s) == 0`. `SIGCHLD` — сигнал, який kernel надіслав parent.

### Зіставлення C-коду з kernel-операціями

```
Ваш C-код
      ↓
fork / exec / wait
      ↓
library / POSIX API
      ↓
kernel-facing operations (clone, execve, wait4)
      ↓
Operating System
```

---

## 21. Відповіді на контрольні питання

1. **Чим program відрізняється від process?**
   Program — статичний файл з інструкціями на диску. Process — запущений екземпляр програми в пам'яті з власним станом, PID, ресурсами.

2. **Що таке PID?**
   Process ID — унікальний числовий ідентифікатор процесу, який видає kernel.

3. **Що таке PPID?**
   Parent PID — PID процесу, який створив даний процес.

4. **Чому один executable може мати декілька PID?**
   Кожен запуск executable створює новий process з власним PID. Executable — це "шаблон", а process — екземпляр.

5. **Яку роль виконує `/proc/PID`?**
   Це віртуальна файлова система, через яку kernel експортує інформацію про process (status, maps, cmdline, exe, fd тощо).

6. **Що таке system call?**
   Це механізм, через який програма в user space звертається до kernel для виконання привілейованих операцій (I/O, створення процесів, виділення пам'яті).

7. **Чим function call відрізняється від system call?**
   Function call — виклик у user space (швидкий, без переходу в kernel). System call — перехід у kernel space (повільніший, привілейований).

8. **Яку роль виконує strace?**
   `strace` перехоплює та показує system calls, які виконує process — це "трасування" взаємодії з kernel.

9. **Що робить fork()?**
   Створює копію поточного process (child), який продовжує виконання з тієї ж точки, але з іншим PID.

10. **Скільки processes існує після одного успішного fork()?**
    Два (parent + child).

11. **Чому fork() повертає 0 у child?**
    Щоб child знав, що він — child. Це дозволяє розгалузити код через `if (pid == 0)`.

12. **Що отримує parent як результат fork()?**
    PID child (> 0), щоб parent знав, кого чекати через `wait()`.

13. **Чи мають parent і child одну спільну змінну value після fork()?**
    Ні. Після fork() address spaces незалежні (copy-on-write). Кожен процес має свою копію змінної.

14. **Чому однакові virtual addresses у parent та child не означають спільну змінювану пам'ять?**
    Однакова virtual address відображається на різні фізичні сторінки (copy-on-write). Зміни в одному процесі не видно в іншому.

15. **Що робить exec()?**
    Замінює program image поточного process новою програмою. Process (PID) залишається, але код і дані замінюються.

16. **Чи створює exec() новий process?**
    Ні. Exec замінює program image в існуючому process.

17. **Що відбувається з PID після успішного exec()?**
    PID залишається тим самим.

18. **Для чого потрібен wait()?**
    Щоб parent дочекався завершення child і зібрав його exit status (запобігає zombie).

19. **Що таке exit status?**
    Код повернення child (значення з `return` у `main` або `exit()`), який parent отримує через `wait()`.

20. **Що таке zombie process?**
    Процес, який завершився, але його exit status ще не зібрано parent. Запис залишається в таблиці процесів.

21. **Чи zombie продовжує виконувати instructions?**
    Ні. Zombie не виконує інструкції і не використовує CPU — це лише запис у таблиці.

22. **У чому різниця між Ready, Running та Waiting?**
    - Ready — готовий до виконання, чекає на CPU
    - Running — виконується на CPU
    - Waiting — чекає на подію (I/O, timer, сигнал)

23. **Чому порядок виконання parent та child після fork() не можна просто гарантувати?**
    Бо scheduler ядра вирішує, який процес отримає CPU наступним. Порядок залежить від багатьох факторів (пріоритет, завантаження, тощо).

24. **Як fork → exec → wait пов'язаний із роботою shell?**
    Shell реалізує запуск команд саме через цю модель: `fork()` → child → `exec(program)` → parent → `wait()` → показати результат.

---

## 22. Висновок

Лабораторна робота експериментально підтвердила:

- **Program vs Process:** executable — статичний файл, process — запущений екземпляр з PID, address space, ресурсами
- **PID/PPID:** kernel видає унікальний PID кожному process; PPID вказує на parent
- **`/proc/PID`:** віртуальна файлова система, через яку kernel експортує стан process (status, maps, cmdline, exe)
- **System calls:** `write()` — приклад переходу з user space в kernel; `printf()` — library function, що викликає `write()`
- **`fork()`:** створює копію process; повертає PID child у parent і 0 у child; address spaces незалежні (copy-on-write)
- **`exec()`:** замінює program image, PID залишається; не створює новий process
- **`wait()`:** блокує parent, поки child не завершиться; збирає exit status (`WIFEXITED`, `WEXITSTATUS`)
- **Zombie:** process, що завершився, але exit status не зібрано; виникає без `wait()`; зникає після `wait()`
- **Process tree:** n послідовних `fork()` → 2ⁿ процесів
- **`strace`:** показує реальні system calls (`clone`, `execve`, `wait4`), які не завжди збігаються з назвами C API (`fork`, `exec`, `wait`)
- **`process_runner`:** реалізує фрагмент поведінки shell — `fork → exec → wait`

Модель `fork → exec → wait` є фундаментальною для Unix-систем і лежить в основі роботи shell, запуску програм, обробки команд.
