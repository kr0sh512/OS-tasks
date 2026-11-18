# Чтение 16-битных слов из виртуальной памяти

Рассмотрим модельный 16-битный процессор с 16-битным виртуальным и физическим адресным пространством. Память адресуется побайтно. Используется одноуровневая страничная адресация с размером страницы **512 байт**. Целые числа хранятся в формате **Little-Endian**: младший байт расположен по меньшему адресу.

Напишите программу, эмулирующую чтение 16-битных слов по виртуальным адресам.

- Первый аргумент командной строки — имя бинарного файла размером ровно **64 KiB = 65536 байт**, содержащего образ физической памяти.
- Второй аргумент — физический адрес начала таблицы страниц в шестнадцатеричной записи.
- Таблица страниц находится внутри этого образа памяти и содержит **128 записей по 16 бит**, то есть занимает **256 байт**. В каждой записи старшие 7 бит задают номер физической страницы, младшие 9 бит равны нулю. Поэтому сама запись численно равна физическому адресу начала соответствующей страницы.
- На стандартном вводе до конца файла подаются виртуальные адреса 16-битных ячеек в шестнадцатеричной записи, разделённые пробельными символами. Все адреса выровнены по границе двух байт, то есть чётные, и находятся в диапазоне от `0x0000` до `0xFFFE`.
- Для каждого адреса напечатайте значение слова в **беззнаковой десятичной записи**, по одному значению на строку, в порядке поступления адресов. Не печатайте приглашений к вводу.

Таблица должна целиком помещаться в физическую память: адрес её начала не больше `0xFF00`. Дополнительное выравнивание начала таблицы не требуется. Все записи описывают отображение страниц; отдельных флагов присутствия в этой модели нет. Шестнадцатеричные числа допускают префикс `0x`/`0X`; знаки не используются.

Порядок байтов и размер слова определяются моделью, а не компьютером, на котором работает программа: нельзя просто читать байты образа в переменную `unsigned short` или разыменовывать указатель на неё. Некорректные аргументы, образ или адрес ввода программа из решения диагностирует в `stderr` и завершает работу с ненулевым кодом; для корректного ввода диагностик нет.

Источник: «Задачи_вирт_память.docx», задача о модельном 16-битном процессоре; фотография 21-11-49.

---

# решение

Виртуальный адрес разбивается на 7-битный номер страницы и 9-битное смещение:

```text
page   = virtual_address >> 9
offset = virtual_address & 0x1FF
entry_address = table_address + 2 * page
physical_address = entry + offset
```

Последняя формула верна, потому что младшие девять бит `entry` равны нулю: запись уже содержит физическую базу страницы. Обе 16-битные величины — запись таблицы и результат чтения — собираются из двух байтов как `low | (high << 8)`.

Чётное смещение не превосходит 510, поэтому два байта слова всегда лежат внутри одной страницы. Даже для последней физической страницы максимальный адрес начала слова равен `0xFFFE`, и второй байт помещается в память. Полная таблица из 128 записей помещается в образ благодаря проверке её начального адреса.

Ни выравнивание указателей, ни порядок байтов хоста не используются. Все вычисления адресов ведутся в `uint32_t`, чтобы число 65536 и промежуточные суммы были представимы даже на машине с 16-битным `int`. Размер слова задан явно — два восьмибитных байта. Образ хранится в статическом массиве, без динамического выделения памяти. Код — полная программа C11.

```c
#include <ctype.h>
#include <limits.h>
#include <inttypes.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>

_Static_assert(CHAR_BIT == 8, "This emulator requires 8-bit bytes");

static unsigned char memory[UINT32_C(65536)];

static int hex_digit(int c)
{
    if (c >= '0' && c <= '9') {
        return c - '0';
    }
    if (c >= 'a' && c <= 'f') {
        return c - 'a' + 10;
    }
    if (c >= 'A' && c <= 'F') {
        return c - 'A' + 10;
    }
    return -1;
}

static int parse_hex16(const char *s, uint32_t *result)
{
    if (s[0] == '0' && (s[1] == 'x' || s[1] == 'X')) {
        s += 2;
    }
    if (*s == '\0') {
        return 0;
    }

    uint32_t value = 0;
    for (; *s != '\0'; ++s) {
        int digit = hex_digit((unsigned char)*s);
        if (digit < 0 ||
            value > (UINT32_C(0xffff) - (uint32_t)digit) / 16) {
            return 0;
        }
        value = value * 16 + (uint32_t)digit;
    }
    *result = value;
    return 1;
}

/* 1: прочитан адрес; 0: чистый EOF; -1: ошибка ввода или записи. */
static int read_address(uint32_t *result)
{
    int c;
    do {
        c = getchar();
    } while (c != EOF && isspace((unsigned char)c));

    if (c == EOF) {
        return ferror(stdin) ? -1 : 0;
    }

    uint32_t value = 0;
    int has_digits = 0;
    int valid = 1;
    if (c == '0') {
        has_digits = 1;
        c = getchar();
        if (c == 'x' || c == 'X') {
            has_digits = 0;
            c = getchar();
        }
    }

    while (c != EOF && !isspace((unsigned char)c)) {
        int digit = hex_digit(c);
        if (digit < 0) {
            valid = 0;
        } else if (valid) {
            if (value > (UINT32_C(0xffff) - (uint32_t)digit) / 16) {
                valid = 0;
            } else {
                value = value * 16 + (uint32_t)digit;
            }
        }
        has_digits = 1;
        c = getchar();
    }

    if (!valid || !has_digits || ferror(stdin)) {
        return -1;
    }
    *result = value;
    return 1;
}

static uint32_t read_le16(uint32_t address)
{
    return (uint32_t)memory[address] |
           ((uint32_t)memory[address + 1] << 8);
}

int main(int argc, char **argv)
{
    if (argc != 3) {
        fprintf(stderr, "Usage: %s memory.bin table_address_hex\n", argv[0]);
        return EXIT_FAILURE;
    }

    uint32_t table_address;
    if (!parse_hex16(argv[2], &table_address) ||
        table_address > UINT32_C(0xff00)) {
        fputs("Invalid page table address\n", stderr);
        return EXIT_FAILURE;
    }

    FILE *input = fopen(argv[1], "rb");
    if (input == NULL) {
        perror(argv[1]);
        return EXIT_FAILURE;
    }
    size_t count = fread(memory, 1, sizeof memory, input);
    int extra = fgetc(input);
    int read_error = ferror(input);
    int close_error = fclose(input);
    if (count != sizeof memory || extra != EOF ||
        read_error || close_error != 0) {
        fputs("Expected a readable 65536-byte memory image\n", stderr);
        return EXIT_FAILURE;
    }

    uint32_t virtual_address;
    int status;
    while ((status = read_address(&virtual_address)) == 1) {
        if ((virtual_address & 1) != 0) {
            fputs("Virtual address is not word-aligned\n", stderr);
            return EXIT_FAILURE;
        }
        uint32_t page = virtual_address >> 9;
        uint32_t offset = virtual_address & UINT32_C(0x1ff);
        uint32_t entry = read_le16(table_address + 2 * page);
        if ((entry & UINT32_C(0x1ff)) != 0) {
            fputs("Page table entry has nonzero low bits\n", stderr);
            return EXIT_FAILURE;
        }
        uint32_t value = read_le16(entry + offset);
        if (printf("%" PRIu32 "\n", value) < 0) {
            return EXIT_FAILURE;
        }
    }
    if (status < 0) {
        fputs("Invalid hexadecimal virtual address or input error\n", stderr);
        return EXIT_FAILURE;
    }
    return fflush(stdout) == EOF ? EXIT_FAILURE : EXIT_SUCCESS;
}
```
