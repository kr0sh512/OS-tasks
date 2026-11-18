# Три процесса и общий счётчик

Напишите программу на C с использованием разделяемой памяти и семафоров System V.

Родитель создаёт сегмент разделяемой памяти для одного `int`, записывает туда `0`, затем создаёт ровно трёх дочерних процессов. Нумерация детей — `0`, `1`, `2` в порядке создания.

Каждый ребёнок должен ровно один раз увеличить общее значение на единицу и завершиться. Действия выполняются строго в порядке **1 → 0 → 2**, независимо от порядка планирования процессов. Для синхронизации используйте три семафора; задержки и активное ожидание не допускаются.

Родитель дожидается всех детей, печатает итоговое значение с переводом строки и освобождает созданные IPC-объекты. При ошибке завершите программу с ненулевым кодом, не оставляя дочерних процессов и IPC-объектов.

Входных данных и аргументов командной строки нет. Единственная строка стандартного вывода успешного запуска — итоговый счётчик.

Источники: набросок `1.md` и фото `photo_2026-11-18_21-12-41.jpg`. Нумерация и способ синхронизации уточнены по коду на доске.

---

# решение

Начальные значения семафоров: `{0, 1, 0}`. Ребёнок `i` ждёт уменьшения своего семафора, увеличивает счётчик, затем увеличивает семафор `(i + 2) % 3`. Получается цепочка `1 → 0 → 2`; доступ к счётчику последователен, отдельный мьютекс не нужен.

Удалять семафоры должен только родитель после завершения детей: удаление в ребёнке может сорвать ожидание остальных. `SEM_UNDO` здесь не нужен: возврат семафоров при выходе разрушил бы передачу хода.

Сборка на Linux: `cc -std=c11 -Wall -Wextra counter.c -o counter`.

```c
#define _POSIX_C_SOURCE 200809L
#include <errno.h>
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/ipc.h>
#include <sys/sem.h>
#include <sys/shm.h>
#include <sys/wait.h>
#include <unistd.h>

union semun {
    int val;
    struct semid_ds *buf;
    unsigned short *array;
};

static int change(int id, unsigned short number, short delta)
{
    struct sembuf op = {number, delta, 0};
    int r;
    do { r = semop(id, &op, 1); } while (r < 0 && errno == EINTR);
    return r;
}

int main(void)
{
    int result = 1, made = 0;
    pid_t children[3] = {0};
    int sem = -1;
    int shm = shmget(IPC_PRIVATE, sizeof(int), IPC_CREAT | 0600);
    if (shm < 0) { perror("shmget"); return 1; }
    int *value = shmat(shm, NULL, 0);
    if (value == (void *) -1) { perror("shmat"); goto cleanup; }
    *value = 0;
    sem = semget(IPC_PRIVATE, 3, IPC_CREAT | 0600);
    if (sem < 0) { perror("semget"); goto cleanup; }
    unsigned short initial[3] = {0, 1, 0};
    if (semctl(sem, 0, SETALL, (union semun){.array = initial}) < 0) {
        perror("semctl"); goto cleanup;
    }
    for (int i = 0; i < 3; ++i) {
        pid_t pid = fork();
        if (pid < 0) { perror("fork"); goto stop; }
        if (pid == 0) {
            if (change(sem, (unsigned short)i, -1) < 0) _exit(1);
            ++*value;
            if (change(sem, (unsigned short)((i + 2) % 3), 1) < 0) _exit(1);
            shmdt(value);
            _exit(0);
        }
        children[made++] = pid;
    }
    for (int left = made; left > 0; --left) {
        int status;
        pid_t pid;
        do { pid = waitpid(-1, &status, 0); } while (pid < 0 && errno == EINTR);
        if (pid < 0) { perror("waitpid"); goto stop; }
        for (int i = 0; i < made; ++i)
            if (children[i] == pid) children[i] = 0;
        if (!WIFEXITED(status) || WEXITSTATUS(status) != 0) goto stop;
    }
    result = printf("%d\n", *value) < 0 ? 1 : 0;
    goto cleanup;
stop:
    for (int i = 0; i < made; ++i)
        if (children[i] > 0) kill(children[i], SIGKILL);
    for (int i = 0; i < made; ++i) {
        if (children[i] <= 0) continue;
        while (waitpid(children[i], NULL, 0) < 0 && errno == EINTR) {}
    }
cleanup:
    if (sem >= 0 && semctl(sem, 0, IPC_RMID) < 0) result = 1;
    if (value != (void *) -1 && shmdt(value) < 0) result = 1;
    if (shmctl(shm, IPC_RMID, NULL) < 0) result = 1;
    return result;
}
```
