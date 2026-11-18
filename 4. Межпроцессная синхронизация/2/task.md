# Обедающие философы

Напишите программу на C, моделирующую обедающих философов с помощью процессов, разделяемой памяти и семафоров System V.

Запуск: `./philosophers N`, где `N >= 2` — число философов за круглым столом. Между соседними философами лежит по одной вилке. Философ `i` использует вилки `i` и `(i + 1) % N`.

Каждому философу соответствует отдельный дочерний процесс. Он бесконечно повторяет цикл:

1. Размышляет (`Thinking`) от 1 до 9 интервалов по 0,3 секунды.
2. Переходит в состояние ожидания вилок (`Wanna take forks`).
3. Получает обе вилки и ест (`Eating`) от 1 до 9 таких же интервалов.
4. Возвращает вилки (`Put forks on a table`).

Соседи не могут есть одновременно. Исключите взаимную блокировку: философ не должен держать одну вилку, ожидая вторую. Гарантировать отсутствие голодания отдельного философа не требуется.

Состояния хранятся в разделяемой памяти. Родитель выводит согласованный снимок всех состояний примерно каждые 1,5 секунды. При получении `SIGINT` (например, Ctrl+C) он завершает детей, дожидается их и удаляет IPC-объекты. Дети не должны независимо удалять общие ресурсы.

Входа на стандартном потоке нет. Формат строки снимка: `Philosopher i: состояние`; снимки разделяются пустой строкой. Точная последовательность снимков зависит от планировщика и не фиксируется.

Источник: `readable_philosophers.c`. Условие восстановлено по программе и комментариям. Параметр `N` обобщает зашитые в исходнике размеры IPC-объектов на 5 философов. Согласованный снимок и безопасное завершение уточнены, чтобы убрать чтение неинициализированной строки и утечки ресурсов.

---

# решение

Заведём `N` семафоров вилок со значением `1` и ещё один семафор-мьютекс для массива состояний. Захват двух вилок выполняется одним `semop` с двумя операциями `-1`. System V применяет весь набор атомарно: если обе вилки недоступны, процесс ждёт, не захватив ни одной. Циклическое ожидание с удержанием вилок невозможно.

Состояние `Eating` устанавливается после захвата вилок, а состояние возврата — перед их освобождением. Родитель читает весь массив под мьютексом, поэтому снимок не может показать одновременно едящих соседей. Вывод выполняется после освобождения мьютекса.

`SIGINT` блокируется до `fork`; родитель принимает его через `sigtimedwait`, а дети оставляют заблокированным. Для остановки детей родитель использует `SIGTERM`, затем `waitpid`. Обработчиков с небезопасными вызовами нет.

Сборка на Linux: `cc -std=c11 -Wall -Wextra philosophers.c -o philosophers`.

```c
#define _POSIX_C_SOURCE 200809L
#include <errno.h>
#include <limits.h>
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/ipc.h>
#include <sys/sem.h>
#include <sys/shm.h>
#include <sys/wait.h>
#include <time.h>
#include <unistd.h>

union semun {
    int val;
    struct semid_ds *buf;
    unsigned short *array;
};

static int operations(int id, struct sembuf *ops, size_t count)
{
    int r;
    do { r = semop(id, ops, count); } while (r < 0 && errno == EINTR);
    return r;
}

static int set_state(int sem, char *states, int n, int i, char state)
{
    struct sembuf lock = {(unsigned short)n, -1, 0};
    struct sembuf unlock = {(unsigned short)n, 1, 0};
    if (operations(sem, &lock, 1) < 0) return -1;
    states[i] = state;
    return operations(sem, &unlock, 1);
}

static void delay(void)
{
    int ticks = 1 + rand() % 9;
    struct timespec t = {ticks * 3 / 10, (ticks * 3 % 10) * 100000000L};
    while (nanosleep(&t, &t) < 0 && errno == EINTR) {}
}

static void philosopher(int sem, char *states, int n, int i)
{
    /* SIGTERM должен завершать ребёнка даже при унаследованной маске. */
    sigset_t term;
    sigemptyset(&term);
    sigaddset(&term, SIGTERM);
    if (sigprocmask(SIG_UNBLOCK, &term, NULL) < 0) _exit(1);
    struct sigaction action = {.sa_handler = SIG_DFL};
    sigemptyset(&action.sa_mask);
    if (sigaction(SIGTERM, &action, NULL) < 0) _exit(1);
    srand((unsigned)getpid());
    struct sembuf take[2] = {
        {(unsigned short)i, -1, 0},
        {(unsigned short)((i + 1) % n), -1, 0}
    };
    struct sembuf put[2] = {
        {(unsigned short)i, 1, 0},
        {(unsigned short)((i + 1) % n), 1, 0}
    };
    for (;;) {
        if (set_state(sem, states, n, i, 't') < 0) _exit(1);
        delay();
        if (set_state(sem, states, n, i, 'w') < 0) _exit(1);
        if (operations(sem, take, 2) < 0) _exit(1);
        if (set_state(sem, states, n, i, 'e') < 0) _exit(1);
        delay();
        if (set_state(sem, states, n, i, 'g') < 0) _exit(1);
        if (operations(sem, put, 2) < 0) _exit(1);
    }
}

static const char *name(char state)
{
    switch (state) {
    case 'w': return "Wanna take forks";
    case 'e': return "Eating";
    case 'g': return "Put forks on a table";
    default: return "Thinking";
    }
}

int main(int argc, char **argv)
{
    if (argc != 2) { fprintf(stderr, "usage: %s N\n", argv[0]); return 1; }
    char *end;
    errno = 0;
    long number = strtol(argv[1], &end, 10);
    if (errno || end == argv[1] || *end || number < 2 ||
        number >= USHRT_MAX || number >= INT_MAX) return 1;
    int n = (int)number;
    sigset_t interrupts;
    sigemptyset(&interrupts);
    sigaddset(&interrupts, SIGINT);
    if (sigprocmask(SIG_BLOCK, &interrupts, NULL) < 0) return 1;
    pid_t *pids = calloc((size_t)n, sizeof(*pids));
    char *snapshot = malloc((size_t)n);
    if (!pids || !snapshot) { free(pids); free(snapshot); return 1; }
    int result = 1, made = 0, sem = -1;
    int shm = shmget(IPC_PRIVATE, (size_t)n, IPC_CREAT | 0600);
    char *states = (void *)-1;
    if (shm < 0) { perror("shmget"); goto cleanup; }
    states = shmat(shm, NULL, 0);
    if (states == (void *)-1) { perror("shmat"); goto cleanup; }
    for (int i = 0; i < n; ++i) states[i] = 't';
    sem = semget(IPC_PRIVATE, n + 1, IPC_CREAT | 0600);
    if (sem < 0) { perror("semget"); goto cleanup; }
    for (int i = 0; i <= n; ++i) {
        if (semctl(sem, i, SETVAL, (union semun){.val = 1}) < 0) {
            perror("semctl"); goto cleanup;
        }
    }
    for (int i = 0; i < n; ++i) {
        pid_t pid = fork();
        if (pid < 0) { perror("fork"); goto stop; }
        if (pid == 0) philosopher(sem, states, n, i);
        pids[made++] = pid;
    }
    for (;;) {
        struct timespec interval = {1, 500000000L};
        int sig = sigtimedwait(&interrupts, NULL, &interval);
        if (sig == SIGINT) { result = 0; break; }
        if (sig < 0 && errno == EINTR) continue;
        if (sig < 0 && errno != EAGAIN) { perror("sigtimedwait"); break; }
        struct sembuf lock = {(unsigned short)n, -1, 0};
        struct sembuf unlock = {(unsigned short)n, 1, 0};
        if (operations(sem, &lock, 1) < 0) break;
        for (int i = 0; i < n; ++i) snapshot[i] = states[i];
        if (operations(sem, &unlock, 1) < 0) break;
        for (int i = 0; i < n; ++i)
            printf("Philosopher %d: %s\n", i, name(snapshot[i]));
        putchar('\n');
        if (fflush(stdout) == EOF) break;
    }
stop:
    for (int i = 0; i < made; ++i) kill(pids[i], SIGTERM);
    for (int i = 0; i < made; ++i) {
        pid_t r;
        do { r = waitpid(pids[i], NULL, 0); } while (r < 0 && errno == EINTR);
        if (r < 0) result = 1;
    }
cleanup:
    if (sem >= 0 && semctl(sem, 0, IPC_RMID) < 0) result = 1;
    if (states != (void *)-1 && shmdt(states) < 0) result = 1;
    if (shm >= 0 && shmctl(shm, IPC_RMID, NULL) < 0) result = 1;
    free(snapshot);
    free(pids);
    return result;
}
```

Размеры разделяемой памяти и набора семафоров зависят от `N`, а не равны константе `5`. Поддерживаемый верхний предел числа семафоров также ограничен настройками ядра: если создать набор нельзя, программа сообщает ошибку и освобождает уже созданные ресурсы.
