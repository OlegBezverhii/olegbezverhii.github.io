---
layout: post
title: Реверс-инжиниринг EsmiLicense
---

# Реверс-инжиниринг EsmiLicense: от .esl к расшифровке и генератору

## Входные данные

Декомпилированные IDA-шные файлы приложения `EsmiLicense.exe` и библиотеки
`EsmiLicense.dll` в форматах `.asm` и `.c` (Hex-Rays псевдокод). Но файлы библиотеки нам не понадобились.
Плюс набор файлов лицензий: `.esl` (зашифрованная), `.elk` (ключ ПК), `.reg` (экспорт реестра с демо-лицензией, которая идёт при установке).

## Шаг 1. Поиск точки входа

В коде около 550 функций. Нужно найти главную функцию расшифровки. Ищем строковые константы, содержащие `license`.

В файле `EsmiLicense.exe.asm` находится упоминание:

```asm
aAvcblowfish db '.?AVCBlowFish@@',0  ; type descriptor name
```

Подтверждение: используется класс `CBlowFish`. Дальше ищем, кто его вызывает.
Функция `sub_4115A0` ссылается на Blowfish-инициализацию и содержит строки:

```
"ESMILIC: Correct key, license decrypted"
"ESMILIC: Wrong key, trying another"
"ESMILIC: License could not be decrypted"
```

Это главная функция расшифровки. Начинаем анализ с неё.

---

## Шаг 2. Анализ sub_4115A0 — главный цикл расшифровки

### 2.1. Структура зашифрованного файла

```c
if ( Size < 0x16 )  // минимум 22 байта
    error;

memmove(v8,  v16, 2u);         // [0:2]   — type/version (00 01)
memmove(Buf2, v16 + 2, 0x10u); // [2:18]  — encrypted MD5 hash (16 байт)
memmove(&v14, v16 + 18, 4u);   // [18:22] — data size, LE uint32
// v14 должно равняться Size - 22
// [22:] — encrypted license data
```

### 2.2. Ключи: 3 попытки

Программа генерирует до трёх ключей и пробует расшифровать по очереди:

| # | Если есть CryptKey | Если нет CryptKey |
|---|---|-------------------|
| 0 | CryptKey из .elk | V2(flags=1) — SID + Workstation |
| 1 | V2(flags=-1) — всё железо | V2(flags=-1) |
| 2 | V1(flags=-1) — MAC + том | V1(flags=-1) |

CryptKey из .elk обрабатывается через `sub_4035F3`:
1. Проверка формата: 39 символов, дефисы на позициях 5,10,15,20,25,30,35
2. Удаление дефисов → 32 hex-символа
3. `sub_403A96` → **_mbslwr_s** → приведение к **lowercase**
4. Результат: 32 строчных ASCII-символа → ключ Blowfish

V1/V2 ключи генерируются через цепочку `MD5(hex(MD5(данные WMI)))`.

### 2.3. Процесс для каждого ключа

```c
sub_401000(bf_ctx, 0, -1);               // init Blowfish (empty key)
sub_401C63(bf_ctx, key_str, key_len);     // set key
sub_40109B(bf_ctx, Buf2, 16);             // decrypt hash
sub_40109B(bf_ctx, v16 + 22, v14);        // decrypt data

sub_40CE96(md5_ctx);                      // MD5 init
sub_40D842(md5_ctx);                      // MD5 init (повторно)
sub_40CE9E(md5_ctx, v16 + 22, v14);       // MD5 update(decrypted_data)
sub_40CF47(md5_ctx);                      // MD5 finalize
sub_40CFF9(md5_ctx, Buf1);                // get 16-byte hash

if (memcmp(Buf1, Buf2, 16) == 0)          // хеш совпал?
    SUCCESS
```

После успеха — обрезка хвостовых `0x01` байт.

---

## Шаг 3. Разбор Blowfish (sub_401000, sub_401C63, sub_401A14)

### 3.1. Инициализация P-box и S-box

```c
memmove(a1,      &unk_5ADB40, 0x48u);  // P-box  (18 × 4 = 72 байта)
memmove(a1 + 72,  &unk_5AD740, 0x400u); // S-box0 (256 × 4 = 1024)
memmove(a1 + 1096, &unk_5AD340, 0x400u); // S-box1
memmove(a1 + 2120, &unk_5ACF40, 0x400u); // S-box2
memmove(a1 + 3144, &unk_5ACB40, 0x400u); // S-box3
```

Проверка по дампу: первые байты P-box = `88 6A 3F 24` → 0x243F6A88.
Совпадает со стандартным Blowfish (hex-цифры π). **P-box и S-box —
стандартные.**

### 3.2. Key schedule: LE-упаковка (НЕ стандартная!)

```c
v5 = 0;
do {
    v5 = key_byte | (v5 << 8);   // ← байты пакуются в little-endian!
    key_pos = (key_pos + 1) % key_len;
} while (--count);
P[i] ^= v5;
```

**Критическое отличие от референсного Blowfish:** Шнайеровский референс пакует
байты ключа как `d = (d << 8) | byte` (big-endian). Здесь `d = byte | (d << 8)`
— байт 0 идёт в младшие биты. Это **little-endian упаковка**, характерная для x86.

Если бы мы использовали стандартную BE-упаковку — расшифровка бы не сработала.

### 3.3. Режим: CBC с нулевым IV

```c
memset(iv, 0, 8);                // IV = {0,0,0,0,0,0,0,0}
sub_401BDE(ctx+1, iv, data, len);
```

Внутри `sub_401BDE`:
```c
memmove(saved, block, 8);        // сохранить шифротекст
sub_40158C(ctx, block);           // Blowfish DECRYPT блока
*block++ ^= block[iv_offset];     // XOR с предыдущим шифротекстом (IV)
memmove(iv, saved, 8);            // IV = сохранённый шифротекст
```

Стандартный CBC decrypt: `P_i = D(C_i) XOR C_{i-1}`, feedback = `C_i`.

---

## Шаг 4. Проверка хеша: MD5

`sub_40D0A6` инициализирует константы:
```
0x67452301, 0xEFCDAB89, 0x98BADCFE, 0x10325476
```
Это стандартные MD5 initialization vectors. Алгоритм — **обычный MD5**,
никаких модификаций.

---

## Шаг 5. Загрузка файла и 418-байтная обёртка

### 5.1. sub_410560 — чтение .esl файла

```c
sub_410350(filepath, &size, 0, 32);  // читает файл
if (sub_410470(data, size))           // проверка обёртки
    sub_4104C0(&data, &size);         // срезать обёртку
```

### 5.2. sub_410470 — проверка обёртки

```c
return size >= 418 && size - 418 == *(uint32*)(data + 414);
```

Если по смещению 414 лежит `filesize - 418` — это файл с обёрткой, первые
418 байт нужно отрезать.

### 5.3. sub_40869B — разбор обёртки (второй проход)

При установке лицензии программа повторно читает `.esl` через `sub_40869B`:

```c
ReadFile(f, &type, 2);           // [0:2]   — тип
ReadFile(f, buf, 128);           // [2:130] — имя продукта ("Esgraf...")
ReadFile(f, &year, 2);           // [130:132] — год (LE uint16)
ReadFile(f, &month, 1);          // [132] — месяц
ReadFile(f, &day, 1);            // [133] — день
// + 3 фиксированных байта (0x0B, 0x32, 0x1D)
ReadFile(f, &cryptkey, 16);      // [137:153] — CryptKey (16 байт)
ReadFile(f, &a6, 1);             // [153] — 0x00
ReadFile(f, &a7, 4);             // [154:158] — 0x80000002
ReadFile(f, regpath, 256);       // [158:414] — путь реестра
ReadFile(f, &size, 4);           // [414:418] — размер данных
```

**Важно:** даты в обёртке (год, месяц, день) должны совпадать с IssueDate
в тексте лицензии. Путь реестра — `Software\Esmi\Esgraf\License\License`.
Байты `a6=0x00, a7=0x80000002` — фиксированные константы.

---

## Шаг 6. Пошаговая отладка и поиск багов

### 6.1. Первая ошибка: регистр ключа

Расшифровка не работала с ключом в верхнем регистре. Причина: `sub_403A96`
вызывает `_mbslwr_s` — приведение к **lowercase**.

**Исправлено:** ключ → `.lower()` перед использованием.

### 6.2. Вторая ошибка: лишний байт в обёртке

При установке сгенерированного `.esl` появлялась ошибка
*«License installation type is unknown»*.

**Отладка через x32dbg:**
1. `bp MessageBoxA` — ловим диалог с ошибкой
2. Call stack показывает путь: `sub_407843` → `sub_41758A` → `MessageBoxA`
3. Выше по стеку: `sub_40869B` — разбор обёртки

**Причина:** в нашей обёртке было 6 байт после CryptKey вместо 5:

```
Оригинал: [CryptKey 16 байт] 00 02 00 00 80 [Software\...]
У нас:    [CryptKey 16 байт] 1E 00 02 00 00 80 [Software\...]
                              ^^ лишний байт!
```

`1E` — это последний байт CryptKey, по ошибке включённый в `WRAPPER_POST_KEY`.
Из-за сдвига на 1 байт путь реестра читался как `\x80Software\...` вместо
`Software\...` — программа не находила нужный ключ → ошибка установки. При этом на расшифровку файла это не влияло.

**Исправлено:** `WRAPPER_POST_KEY = bytes([0x00, 0x02, 0x00, 0x00, 0x80])`.

---

## Шаг 7. Итоговый алгоритм

### Расшифровка

```
1. Открыть .esl
2. Если *(uint32*)(offset 414) == filesize - 418 → отрезать 418 байт обёртки
3. Прочитать заголовок: type(2) + enc_hash(16) + data_size(4)
4. Взять CryptKey из .elk → убрать дефисы → lowercase → 32 ASCII байта
5. Blowfish(CBC, IV=0, LE key schedule): расшифровать hash, расшифровать data
6. MD5(decrypted_data) == decrypted_hash ?
7. Обрезать хвостовые 0x01
```

### Генерация

```
1. Прочитать CryptKey и Workstation из .elk
2. Сформировать текст лицензии (cp1251)
3. Паддинг 0x01 до границы 8 байт
4. MD5(padded_text)
5. Blowfish(CBC, IV=0): зашифровать hash, зашифровать padded_data
6. Собрать inner block: type + enc_hash + data_size + enc_data
7. Собрать обёртку 418 байт:
   - type (00 01)
   - "Esgraf" + padding до 130 байт
   - год (LE uint16), месяц, день, 0x0B, 0x32, 0x1D
   - CryptKey (16 байт binary)
   - 00 02 00 00 80
   - "Software\Esmi\Esgraf\License\License\0"
   - zeros до offset 414
   - inner_data_size (LE uint32)
8. Записать обёртку + inner data
```

### Ключевые константы

| Параметр | Значение |
|----------|---------|
| P-box, S-box | Стандартные Blowfish (hex-цифры π) |
| Key schedule | **Little-endian** упаковка байт |
| Режим | CBC, IV = `\x00` × 8 |
| Хеш | Стандартный MD5 |
| Ключ | 32 строчных ASCII hex-символа |
| Кодировка текста | **cp1251** (Windows-1251) |
| Размер обёртки | 418 байт |
| Паддинг данных | `0x01` до границы 8 байт |


Ссылка на репозиторий с расшифровкой - [FireSuite-Viewer-lic](https://github.com/OlegBezverhii/FireSuite-Viewer-lic).