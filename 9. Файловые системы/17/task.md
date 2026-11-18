# Фрагментация мать её

Источник: `Задачи на код.md`, «Фрагментация мать её».

Система использует непрерывное размещение файлов: каждый файл занимает один непрерывный диапазон байтов. Память доступна через `FILE *memory`, её размер в байтах равен внешнему макросу `N`. Таблица содержит `K` записей, где `K` также задан внешним макросом:

```c
struct table {
    char name[20];
    unsigned start_pos;
    unsigned length;
};
struct table files[K];
```

В исходнике объявления `char[20] name` и `struct table[K] files` синтаксически неверны; выше явно исправлено только расположение размеров массивов.

Напишите функцию

```c
int memory_control(FILE *memory, struct table *files, char *response);
```

Она обрабатывает один запрос:

- `+ имя данные` — создать файл;
- `* имя данные` — дописать данные в конец существующего файла;
- `- имя` — удалить файл.

Имя и данные — непустые слова. Если создать файл невозможно, вернуть внешний код `ENOSPACE`, ничего не меняя. Дописывание сначала выполняется на прежнем месте, если за файлом достаточно непрерывного свободного пространства. Иначе файл перемещается в первый подходящий свободный диапазон. При невозможности дописывания вернуть внешний код `ENOUPD`, ничего не меняя. Удаление существующего файла успешно; для отсутствующего вернуть внешний код `ENOFILE`, ничего не меняя. Успех обозначается `0`.

В исходнике недостаёт ряда соглашений. Для однозначной реализации принимаются следующие:

1. `N` и `K` положительны, `N <= UINT_MAX` и `N <= LONG_MAX`. Поток позиционируемый, открыт для чтения и записи, содержит не менее `N` байтов; операции ввода-вывода успешны. Дополнительного кода ошибки ввода-вывода условие не задаёт.
2. Свободная запись определяется по `name[0] == '\0'`; её остальные поля не учитываются. Занятые записи имеют уникальные имена длиной от 1 до 19 байтов, корректные непересекающиеся диапазоны внутри памяти и положительную длину. `length` измеряется в байтах, завершающий NUL в память файла не записывается.
3. Запрос — корректная NUL-терминированная строка в указанной форме, с одним обычным пробелом между словами и без завершающего перевода строки. При создании имя ещё не существует и есть свободная запись таблицы; при дописывании файл существует. Иначе исходник не определяет нужный код ошибки, поэтому такие запросы не входят в область задачи. Удалять несуществующее имя разрешается.
4. Создание использует первую свободную запись таблицы. Размещение first fit означает свободный диапазон с минимальным начальным адресом. При поиске места для перемещения прежний диапазон этого же файла считается освобождённым: новый диапазон может с ним перекрываться. До нахождения подходящего места никаких изменений не производится.
5. Удаление освобождает запись, но не затирает прежние байты. Перемещение также не затирает байты вне нового диапазона. Состояние памяти не меняется посторонним кодом во время вызова.

---
# решение

Поиск места выполняется по границам занятых диапазонов, а не перебором `N` байтов. Без дополнительного массива и сортировки таблицы он требует `O(K²)` времени и `O(1)` дополнительной памяти. Проверка продолжения на месте требует `O(K)`. Перемещение копирует только содержимое перемещаемого файла небольшими порциями; объём всей памяти на сложность копирования не влияет.

Прежде чем что-либо записывать, проверяем длину и находим место. Поэтому `ENOSPACE` и `ENOUPD` оставляют неизменными и таблицу, и все байты памяти. При перекрывающемся перемещении направление копирования выбирается как у `memmove`: влево — от начала, вправо — от конца. Позиционирование перед каждой операцией чтения и записи также обеспечивает корректное переключение направлений ввода-вывода у update-потока `FILE`.

```c
#include <stdio.h>
#include <string.h>
#include <stddef.h>
#include <limits.h>

/* N, K, ENOSPACE, ENOUPD, ENOFILE заданы извне. */
struct table {
    char name[20];
    unsigned start_pos;
    unsigned length;
};

/* skip == K означает, что исключённой записи нет. */
static int mc_first_fit(const struct table *files, size_t skip,
                        unsigned need, unsigned *result)
{
    unsigned pos = 0;
    for (;;) {
        unsigned next = (unsigned)N;
        unsigned next_end = (unsigned)N;
        for (size_t i = 0; i < (size_t)K; ++i) {
            if (i == skip || files[i].name[0] == '\0')
                continue;
            if (files[i].start_pos >= pos && files[i].start_pos < next) {
                next = files[i].start_pos;
                next_end = next + files[i].length;
            }
        }
        if (need <= next - pos) {
            *result = pos;
            return 1;
        }
        if (next == (unsigned)N)
            return 0;
        pos = next_end;
    }
}

static int mc_can_extend(const struct table *files, size_t index,
                         unsigned extra)
{
    unsigned end = files[index].start_pos + files[index].length;
    if (extra > (unsigned)N - end)
        return 0;
    unsigned new_end = end + extra;
    for (size_t i = 0; i < (size_t)K; ++i) {
        if (i == index || files[i].name[0] == '\0')
            continue;
        unsigned other_end = files[i].start_pos + files[i].length;
        if (files[i].start_pos < new_end && other_end > end)
            return 0;
    }
    return 1;
}

static void mc_move(FILE *memory, unsigned from, unsigned to,
                    unsigned length)
{
    unsigned char buffer[256];
    if (from == to)
        return;
    if (to < from) {
        unsigned done = 0;
        while (done < length) {
            unsigned amount = length - done;
            if (amount > sizeof buffer)
                amount = (unsigned)sizeof buffer;
            fseek(memory, (long)(from + done), SEEK_SET);
            fread(buffer, 1, amount, memory);
            fseek(memory, (long)(to + done), SEEK_SET);
            fwrite(buffer, 1, amount, memory);
            done += amount;
        }
    } else {
        unsigned left = length;
        while (left != 0) {
            unsigned amount = left;
            if (amount > sizeof buffer)
                amount = (unsigned)sizeof buffer;
            left -= amount;
            fseek(memory, (long)(from + left), SEEK_SET);
            fread(buffer, 1, amount, memory);
            fseek(memory, (long)(to + left), SEEK_SET);
            fwrite(buffer, 1, amount, memory);
        }
    }
}

int memory_control(FILE *memory, struct table *files, char *response)
{
    char operation = response[0];
    const char *name = response + 2;
    size_t name_length = strcspn(name, " ");
    size_t index = (size_t)K;
    size_t free_index = (size_t)K;

    for (size_t i = 0; i < (size_t)K; ++i) {
        if (files[i].name[0] == '\0') {
            if (free_index == (size_t)K)
                free_index = i;
        } else if (strlen(files[i].name) == name_length &&
                   memcmp(files[i].name, name, name_length) == 0) {
            index = i;
        }
    }

    if (operation == '-') {
        if (index == (size_t)K)
            return ENOFILE;
        files[index].name[0] = '\0';
        files[index].start_pos = 0;
        files[index].length = 0;
        return 0;
    }

    const char *data = name + name_length + 1;
    size_t data_length = strlen(data);
    unsigned place;

    if (operation == '+') {
        if (data_length > (size_t)N || free_index == (size_t)K)
            return ENOSPACE;
        unsigned length = (unsigned)data_length;
        if (!mc_first_fit(files, (size_t)K, length, &place))
            return ENOSPACE;
        fseek(memory, (long)place, SEEK_SET);
        fwrite(data, 1, length, memory);
        memcpy(files[free_index].name, name, name_length);
        files[free_index].name[name_length] = '\0';
        files[free_index].start_pos = place;
        files[free_index].length = length;
        return 0;
    }

    /* По предусловию это '*', а index обозначает существующий файл. */
    unsigned old_length = files[index].length;
    if (data_length > (size_t)((unsigned)N - old_length))
        return ENOUPD;
    unsigned extra = (unsigned)data_length;
    unsigned new_length = old_length + extra;
    place = files[index].start_pos;
    if (!mc_can_extend(files, index, extra)) {
        if (!mc_first_fit(files, index, new_length, &place))
            return ENOUPD;
        mc_move(memory, files[index].start_pos, place, old_length);
    }
    fseek(memory, (long)(place + old_length), SEEK_SET);
    fwrite(data, 1, extra, memory);
    files[index].start_pos = place;
    files[index].length = new_length;
    return 0;
}
```

Проверка `free_index == K` защитно возвращает уже предусмотренный `ENOSPACE`, хотя по соглашению о запросах такой случай не возникает. Остальные некорректные запросы не получают выдуманных кодов: они исключены явными предусловиями. Численные значения внешних кодов ошибок здесь не определяются.

Для передачи конкретных состояний используется текстовое описание аргументов: строки `N n`, `K k`, `fill c`, `requests r`, затем ровно `r` строк запросов. До первого вызова все записи свободны, а все `N` байтов потока равны однобайтовому символу `c`. Это описание последовательности вызовов функции, а не дополнительный интерфейс программы. Результат содержит возвраты в порядке вызовов (`returns`, затем по одному значению на строке), все `K` строк таблицы в порядке индексов (`index имя start length данные` либо `index FREE`) и строку `memory`, после которой записаны все `N` байтов памяти. Внешние ошибки обозначаются именами констант.
