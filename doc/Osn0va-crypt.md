---
order: 1
title: V:Alpha-0.5aw — Полная криптографическая спецификация
---

Автор: Ольховников Л. А.\
Версия: 0.5aw

Дата: май 2025

---

## 🔐 Описание

V:Alpha-0.5aw -- это самогенерирующийся криптографический протокол, использующий:

-  Изменяющийся ключ (в RAM)

-  Временные сектора длительностью 3 секунды

-  Префиксные управляющие символы и маскировку

-  Автоматическое приближение к эталонному значению

---

## 📦 Ключ

Ключ представляет собой строку цифр произвольной длины.\
Каждая тройка цифр в ключе управляет 1 сектором:

Пример:

```
KEY = 352624254616436126554...
```

Каждая тройка:

-  K = прямой порядок (пример: 352)

-  P = обратный порядок (пример: 253)

-  Размер сектора = `K - P` (в примере: 99 строк)

Ключ:

-  Хранится исключительно в оперативной памяти (RAM)

-  Никогда не записывается на диск

---

## ⏱ Временная структура

Каждые 3 секунды активируется новая тройка из ключа, создавая новый сектор.

Сектор:

-  Живёт строго 3 секунды

-  Может содержать максимум `K - P` строк

-  После 3 секунд сектор закрывается, вне зависимости от количества строк

---

## 🧩 Префикс и семантика символов

Каждая строка начинается с 4-символьного префикса, где первый символ -- управляющий:

Допустимые символы и их значения:

| Символ | Назначение                                                       |
|--------|------------------------------------------------------------------|
| `x`    | Указатель на сервер (ядро протокола)                             |
| `0`    | Маскировка, шум                                                  |
| `a`    | Событие: рождение/смерть сектора, конец строк, начало новых      |
| `b`    | Обращение к `x` для выполнения действия `a`                      |
| `c`    | Резкое завершение всего протокола                                |
| `d`    | Работа с большими объёмами данных                                |
| `e`    | Выполнение действия из набора (`x`, `a`, `b`, `c`, `d`, `f`)     |
| `f`    | Взаимодействие с другими службами (`x`, `a`, `b`, `c`, `d`, `e`) |

## 🧭 Смещение позиции ведущего символа в префиксе

В протоколе V:Alpha-0.5aw управляющий (семантически значимый) символ в 4-символьном префиксе может находиться в любой позиции: в начале, середине или в конце. Его положение смещается динамически, на основе троек текущего блока ключа.

### Алгоритм смещения:

1. Из ключа берётся активная тройка, например:

```
Тройка = 3, 5, 2
```

1. Вычисляется её обратный порядок:

```
Обратная тройка = 2, 5, 3
```

1. Каждое значение в обратной тройке определяет позицию (или сдвиг) управляющего символа в префиксе. Правила: •	Если значение ≤ 4 --> используется напрямую •	Если > 4 --> нормализуется: position = (s % 4) + 2

2. Ведущий символ (например, x, a, b, c, d, e, f) вставляется в рассчитанную позицию внутри 4-символьного префикса, остальные 3 символа -- случайные/маскирующие.

### Пример:

Тройка из ключа: 3, 5, 2 --> Обратный порядок: 2, 5, 3 --> Последовательные позиции: 2, (5 % 4) + 2 = 3, 3

Варианты префиксов: •	fxae (x на позиции 2) •	afex (x на позиции 3) •	x0cf (x на позиции 0) •	и т. д.

⚠️ При обработке строки парсер протокола должен уметь определить, где находится управляющий символ, среди допустимых: x, a, b, c, d, e, f

⸻

## 📄 Формат строки

Каждая строка в протоколе строится строго по шаблону:

\<префикс из 4 символов>:\<значение A> => \<значение B> => (0x) \<значение приближения>

📘 Пример строки:

```
afex:cbd0a2e4c => bdfc:adecb01fx => (0x) 0x0c0x0d0x0e0x0f0x0b0x0c0x0d0x0e0x0f0x0b0x0e
```

🔍 Пояснение к структуре: •	Префикс: afex •	Ведущий символ x находится в позиции 3 (смещён согласно обратной тройке) •	Остальные символы (a, f, e) являются шумом •	Значение A: cbd0a2e4c •	Исходное состояние строки (в пределах сектора) •	Использует только допустимые символы: 0, a–f, x •	Значение B: adecb01fx •	Результат смещения A с учётом активной тройки ключа •	(0x): •	Фиксированный, неизменяемый маркер приближения •	Служит как якорь между основными данными и приближением •	Значение приближения: •	Представляет, насколько строка приближена к эталонному значению:



---

## 🎯 Эталон приближения

Все строки стремятся к одной неизменной цели:

```
(0x) 0x0x0x0x0x0x0x0x0x0x0x0x
```

Правила:

-  `(0x)` -- всегда один и тот же маркер

-  Последние 8 символов приближения = идентификатор текущего сектора

-  Если приближение достигнуто:

   -  Сектор считается завершённым

   -  Telegram уведомление отправляется

   -  Генерируется новый ключ (или продолжение)

⚠️ Если приближение не достигнуто, сектор просто завершается и Telegram не используется.

---

## 🔄 Жизненный цикл сектора (ASCII-диаграмма)

```
+---------------------+
|   Start сектора     |
+---------------------+
					|
			[3 секунды]
					|
			+--------------------+
			| Генерация строк N  |
			+--------------------+
					|
			[Проверка приближения]
					/         \
			[да]          [нет]
			/                \
[Telegram]      [Просто переход]
[новый KEY]        [след. тройка]
```

---

## 🧪 Пример кода (Python)

```python
def parse_triplet(triplet):
		K = int(''.join(triplet))
		P = int(''.join(reversed(triplet)))
		return K, P, K - P

# Пример
key_block = ['3', '5', '2']
K, P, N = parse_triplet(key_block)
print(f"Сектор: {N} строк, длительность: 3 секунды")
```

---

## ✅ Преимущества протокола

-  ⏱ Высокая временная безопасность (каждый блок живёт 3 секунды)

-  🔁 Самообновляемая структура

-  🔐 Нет хранения ключа -- только в RAM

-  💠 Маскировка данных через шумовые символы

-  🧠 Структура приближается к цели без логирования

---