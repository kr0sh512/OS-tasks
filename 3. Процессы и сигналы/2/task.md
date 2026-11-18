# Слияние файлов разными процессами с синхронизацией сигналами

Напишите программу для POSIX-системы, которая объединяет два файла в третий, поочерёдно записывая блоки по `N` байт: сначала блок первого файла, затем блок второго, снова блок первого и так далее. Первый и второй входные файлы должны читать **разные процессы**; эти же процессы записывают прочитанные данные в выходной файл. Очерёдность их работы необходимо организовать **сигналами**, без каналов, сокетов, семафоров и разделяемой памяти для передачи данных или управления очередностью.

Командная строка:

```text
./merge input1 input2 output N
```

Здесь `input1` и `input2` — имена входных файлов, `output` — имя файла-результата, а `N` — положительное целое число байт в блоке. Единица измерения — байт, а не символ многобайтовой кодировки.

Требования:

1. Первым записывается блок из `input1`.
2. Последний блок каждого файла может содержать меньше `N` байт. Нужно записать только реально прочитанные байты, без дополнения и без остатков прежнего содержимого буфера.
3. Когда один входной файл закончился, вся ещё не записанная часть другого дописывается в конец результата без дальнейшего чередования. Пустые входные файлы допустимы.
4. Конец файла определяется результатом `read(...) == 0`. Возврат `-1` означает ошибку; короткое положительное чтение само по себе не доказывает конец файла. Частичные записи `write` также необходимо обрабатывать.
5. `output` формируется **с нуля**: если он существует, его прежнее содержимое удаляется. Он не должен обозначать тот же файл, что любой из входных, в том числе через жёсткую или символическую ссылку.
6. Процесс, ожидающий своей очереди, должен спать, а не выполнять активный опрос. Недопустима гонка, при которой сигнал приходит между проверкой флага и вызовом `pause` и процесс затем засыпает навсегда.
7. Родитель должен дождаться дочернего процесса. При успешной работе программа завершается с кодом `0` и ничего не печатает в стандартный вывод; при ошибке выдаёт диагностику в стандартный поток ошибок и завершается с ненулевым кодом.

Восстановленные соглашения: порядок аргументов, начало с первого файла, поведение для пустых файлов и создание результата с нуля уточнены для однозначного запуска. На доске использован `O_APPEND`, но добавление результата к старому содержимому здесь не требуется. Во время работы входные файлы не изменяются другими процессами; служебные сигналы участникам извне не посылаются.

Обычные сигналы могут сливаться, если несколько экземпляров одного сигнала ожидают приёма. Поэтому протокол должен гарантировать, что один процесс не выдаёт несколько не принятых другим процессом разрешений на запись. В частности, нельзя заранее отправить пачку одинаковых сигналов, рассчитывая использовать её как очередь разрешений.

Источники: печатное задание № 5 и досочный набросок, фотографии `photo_2026-11-18_21-12-37.jpg` и `photo_2026-11-18_21-12-26.jpg` от 18.11.2026. Некорректные проверки EOF и ожидания с доски исправлены в решении.

---

# решение

Достаточно двух процессов: родитель читает первый файл, ребёнок — второй. Выходной файл открываем **до `fork`**, поэтому унаследованные дескрипторы ссылаются на одно открытое файловое описание с общей текущей позицией. Сначала проверяем по устройству и inode, что результат не совпадает ни с одним входным файлом, и только после этого обнуляем его через `ftruncate`. `O_APPEND` не используется.

До `fork` блокируем `SIGUSR1`, `SIGUSR2` и `SIGCHLD`. Родитель получает начальную очередь без сигнала, ребёнок сразу ожидает через `sigwait`. Протокол:

- `SIGUSR1` передаёт очередь: отправитель уже полностью записал свой блок и теперь сам ждёт.
- `SIGUSR2` сообщает, что отправитель обнаружил EOF своего файла и больше не будет записывать данные. Получатель копирует весь оставшийся хвост своего файла без ожиданий.
- `SIGCHLD` позволяет родителю заметить аварийное завершение ребёнка даже тогда, когда родитель ждёт передачи очереди. Успешное завершение ребёнка также означает, что его данные уже записаны; оно может наблюдаться раньше ожидающего `SIGUSR2`.

При строгой передаче очереди одновременно существует не больше одного не принятого разрешения на следующий блок, поэтому слияние обычных сигналов протокол не нарушает. Маска устанавливается до создания ребёнка, так что даже сигнал, пришедший до входа получателя в `sigwait`, остаётся ожидающим. Обработчиков и вызовов `pause` нет.

`copy_block` копирует до `N` байт, используя небольшой буфер, а не выделяя память размером `N`. Короткое чтение уменьшает остаток блока; только чтение нуля завершает файл. Если EOF обнаружен после неполного блока, уже записанная часть сохраняется, затем отправляется `SIGUSR2`. Если размер файла кратен `N`, EOF будет обнаружен при следующей попытке чтения; порядок результирующих байтов от этого не меняется.

```c
#define _POSIX_C_SOURCE 200809L

#include <errno.h>
#include <fcntl.h>
#include <inttypes.h>
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>

static int write_all(int fd, const char *data, size_t size)
{
    while (size != 0) {
        ssize_t written = write(fd, data, size);
        if (written == -1) {
            if (errno == EINTR)
                continue;
            return -1;
        }
        if (written == 0) {
            errno = EIO;
            return -1;
        }
        data += (size_t)written;
        size -= (size_t)written;
    }
    return 0;
}

/* 1: полные N байт; 0: EOF, возможно после неполного блока; -1: ошибка. */
static int copy_block(int input, int output, uintmax_t n)
{
    char buffer[8192];
    uintmax_t left = n;

    while (left != 0) {
        size_t wanted = left < sizeof buffer ? (size_t)left : sizeof buffer;
        ssize_t count = read(input, buffer, wanted);
        if (count == -1) {
            if (errno == EINTR)
                continue;
            return -1;
        }
        if (count == 0)
            return 0;
        if (write_all(output, buffer, (size_t)count) == -1)
            return -1;
        left -= (uintmax_t)count;
    }
    return 1;
}

/* 0: очередь; 1: другой вход закончился; -1: ошибка.
   child == 0 у ребёнка; родитель передаёт PID своего ребёнка. */
static int await_turn(const sigset_t *signals, pid_t child,
                      int *reaped, int *status)
{
    for (;;) {
        int received;
        int error = sigwait(signals, &received);
        if (error != 0) {
            errno = error;
            return -1;
        }
        if (received == SIGUSR1)
            return 0;
        if (received == SIGUSR2)
            return 1;
        if (received == SIGCHLD && child > 0) {
            pid_t result;
            do {
                result = waitpid(child, status, WNOHANG);
            } while (result == -1 && errno == EINTR);
            if (result == -1)
                return -1;
            if (result == child) {
                *reaped = 1;
                if (WIFEXITED(*status) && WEXITSTATUS(*status) == 0)
                    return 1;
                errno = EIO;
                return -1;
            }
            /* Например, уведомление об остановке, а не завершении. */
        }
    }
}

static int merge_part(int input, int output, uintmax_t n, pid_t peer,
                      const sigset_t *signals, int first,
                      int *reaped, int *status)
{
    pid_t child = first ? peer : 0;

    for (;;) {
        if (!first) {
            int event = await_turn(signals, child, reaped, status);
            if (event == -1)
                return -1;
            if (event == 1) {
                int result;
                do {
                    result = copy_block(input, output, n);
                } while (result == 1);
                return result == -1 ? -1 : 0;
            }
        }
        first = 0;

        int result = copy_block(input, output, n);
        if (result == -1)
            return -1;
        if (result == 0)
            return kill(peer, SIGUSR2);
        if (kill(peer, SIGUSR1) == -1)
            return -1;
    }
}

int main(int argc, char **argv)
{
    if (argc != 5) {
        fprintf(stderr, "Usage: %s input1 input2 output N\n", argv[0]);
        return EXIT_FAILURE;
    }

    char *end;
    errno = 0;
    uintmax_t n = strtoumax(argv[4], &end, 10);
    if (argv[4][0] < '0' || argv[4][0] > '9' ||
        *end != '\0' || errno == ERANGE || n == 0) {
        fputs("N must be a positive integer representable by uintmax_t\n",
              stderr);
        return EXIT_FAILURE;
    }

    int input1 = open(argv[1], O_RDONLY);
    if (input1 == -1) {
        perror(argv[1]);
        return EXIT_FAILURE;
    }
    int input2 = open(argv[2], O_RDONLY);
    if (input2 == -1) {
        perror(argv[2]);
        return EXIT_FAILURE;
    }
    int output = open(argv[3], O_WRONLY | O_CREAT, 0666);
    if (output == -1) {
        perror(argv[3]);
        return EXIT_FAILURE;
    }

    struct stat first_stat, second_stat, output_stat;
    if (fstat(input1, &first_stat) == -1 ||
        fstat(input2, &second_stat) == -1 ||
        fstat(output, &output_stat) == -1) {
        perror("fstat");
        return EXIT_FAILURE;
    }
    if ((output_stat.st_dev == first_stat.st_dev &&
         output_stat.st_ino == first_stat.st_ino) ||
        (output_stat.st_dev == second_stat.st_dev &&
         output_stat.st_ino == second_stat.st_ino)) {
        fputs("Output must differ from both input files\n", stderr);
        return EXIT_FAILURE;
    }
    if (ftruncate(output, 0) == -1) {
        perror("ftruncate");
        return EXIT_FAILURE;
    }

    sigset_t signals;
    sigemptyset(&signals);
    sigaddset(&signals, SIGUSR1);
    sigaddset(&signals, SIGUSR2);
    sigaddset(&signals, SIGCHLD);
    if (sigprocmask(SIG_BLOCK, &signals, NULL) == -1) {
        perror("sigprocmask");
        return EXIT_FAILURE;
    }

    pid_t parent = getpid();
    pid_t child = fork();
    if (child == -1) {
        perror("fork");
        return EXIT_FAILURE;
    }
    if (child == 0) {
        close(input1);
        int reaped = 0, status = 0;
        int result = merge_part(input2, output, n, parent, &signals, 0,
                                &reaped, &status);
        if (result == 0 && close(output) == -1)
            result = -1;
        if (result == -1)
            perror("child merge");
        _exit(result == 0 ? EXIT_SUCCESS : EXIT_FAILURE);
    }

    close(input2);
    int reaped = 0, status = 0;
    int result = merge_part(input1, output, n, child, &signals, 1,
                            &reaped, &status);
    if (result == -1) {
        perror("parent merge");
        if (!reaped)
            kill(child, SIGTERM);
    }
    if (!reaped) {
        pid_t waited;
        do {
            waited = waitpid(child, &status, 0);
        } while (waited == -1 && errno == EINTR);
        if (waited == -1) {
            perror("waitpid");
            result = -1;
        }
    }
    if (!WIFEXITED(status) || WEXITSTATUS(status) != 0) {
        fputs("Child did not finish successfully\n", stderr);
        result = -1;
    }
    if (close(output) == -1) {
        perror("close output");
        result = -1;
    }
    return result == 0 ? EXIT_SUCCESS : EXIT_FAILURE;
}
```

Данные не передаются между процессами: каждый читает свой вход и сам записывает его байты. Наследование выходного дескриптора даёт общую позицию записи, а сигнальный протокол исключает одновременную запись. При собственной ошибке родитель завершает ребёнка и собирает его статус; при ошибке ребёнка родитель получает `SIGCHLD` и не остаётся в вечном ожидании очереди.

Время работы — `O(|input1| + |input2|)`, дополнительная память каждого процесса — `O(1)` относительно размеров файлов и `N`.
