# Лабораторная работа №7
## Преобразование и анализ кода с использованием Clang и LLVM

**Автор:** _<ФИО, группа>_
**Вариант индивидуального задания:** 2.10 «Функции (объявление / создание)»
**Среда:** Ubuntu 22.04, Clang/LLVM 14, Graphviz

---

## 1. Постановка задачи

### 1.1 Общая часть

1. Установить Clang, LLVM, `opt` и Graphviz.
2. Скомпилировать простой C-файл и получить:
   - абстрактное синтаксическое дерево (AST);
   - промежуточное представление LLVM IR.
3. Применить базовую комплексную оптимизацию (`-O2`).
4. Построить граф потока управления (CFG) для оптимизированной программы.
5. Проанализировать результат и сделать выводы.

### 1.2 Индивидуальное задание (вариант 2.10)

Исходный код (`main.cpp`):

```cpp
#include <cstdio>

inline int square(int x) {
    return x * x;
}

int main() {
    int a = 5;
    int b = square(a);
    return b;
}
```

Требуется:

1. Получить IR для `-O0`.
2. Получить IR для `-O2`. Встроилась ли функция?
3. Применить `-always-inline` и сравнить с предыдущими оптимизациями.
4. Построить CFG до и после.
5. Сделать вывод об условиях встраивания функций в LLVM.

---

## 2. Общее задание

### 2.1 Установка инструментов

```bash
sudo apt install clang llvm graphviz
```

Проверка версии:

```bash
clang --version
opt --version
dot -V
```

### 2.2 Исходный код для общей части (`main.c`)

```c
#include <stdio.h>

int square(int x) {
    return x * x;
}

int main() {
    int a = 5;
    int b = square(a);
    printf("%d\n", b);
    return 0;
}
```

### 2.3 Получение AST

```bash
clang -Xclang -ast-dump -fsyntax-only main.c
```

В выводе видно `FunctionDecl square 'int (int)'` с параметром `x` и телом `return x * x`,
а также `FunctionDecl main 'int ()'` с объявлением переменных `a`, `b`, вызовом `square` и
вызовом `printf`.

### 2.4 Генерация LLVM IR

```bash
clang -O0 -S -emit-llvm main.c -o main_O0.ll
clang -O2 -S -emit-llvm main.c -o main_O2.ll
```

В файле `main_O0.ll`:
- все переменные размещены в стеке через `alloca`;
- значения читаются и пишутся через `load`/`store`;
- `square` остаётся отдельной функцией и вызывается через `call`.

### 2.5 Оптимизация IR (`-O2`)

При `-O2` применяется более 30 проходов, ключевые из них:
- `mem2reg` — перевод переменных из памяти в виртуальные регистры (SSA-форма);
- `inline` — встраивание небольших функций;
- `sccp` / `constprop` — свёртка констант;
- `instcombine` — упрощение арифметических инструкций;
- `simplifycfg` — упрощение графа потока управления;
- `dce`, `globaldce` — удаление мёртвого кода.

Сравнение IR:

```bash
diff main_O0.ll main_O2.ll
```

После оптимизаций:
- `alloca`/`load`/`store` удалены;
- функция `square` встроена в `main` и затем удалена как мёртвый код;
- от `main` остался только вызов `printf` с константой 25.

### 2.6 Построение CFG

```bash
opt -dot-cfg -disable-output main_O2.ll
dot -Tpng .main.dot -o cfg_main.png
xdg-open cfg_main.png
```

CFG функции `main` после `-O2` состоит из одного базового блока с единственным
вызовом `printf` и `ret i32 0`. CFG функции `square` после `-O2` не строится, так как
её определение удалено оптимизациями.

### 2.7 Выводы по общей части

- Clang даёт прямой доступ к AST и IR, что позволяет видеть все стадии трансляции.
- LLVM предоставляет богатый набор инструментов для анализа и оптимизации, причём
  каждый pass можно применять и наблюдать отдельно.
- Промежуточное представление и CFG удобны для написания компиляторных
  трансформаций, поскольку имеют регулярную форму (SSA, базовые блоки).

---

## 3. Индивидуальное задание (вариант 2.10)

### 3.1 Исходный код

```cpp
#include <cstdio>

inline int square(int x) {
    return x * x;
}

int main() {
    int a = 5;
    int b = square(a);
    return b;
}
```

> В качестве C++ файла, так как `inline` в C требует `static inline` или отдельной
> внешней декларации для одиночного определения.

### 3.2 AST

Команда:
```bash
clang -Xclang -ast-dump -fsyntax-only main.cpp
```

Ключевые узлы (сокращённо):
```
|-FunctionDecl  square 'int (int)' inline
| |-ParmVarDecl used x 'int'
| `-CompoundStmt
|   `-ReturnStmt
|     `-BinaryOperator 'int' '*'
|       |-DeclRefExpr ParmVar 'x'
|       `-DeclRefExpr ParmVar 'x'
`-FunctionDecl main 'int ()'
  `-CompoundStmt
    |-VarDecl used a 'int' cinit
    |   `-IntegerLiteral 'int' 5
    |-VarDecl used b 'int' cinit
    |   `-CallExpr 'int'
    |     |-DeclRefExpr Function 'square'
    |     `-DeclRefExpr Var 'a'
    `-ReturnStmt
      `-DeclRefExpr Var 'b'
```

`inline` сохраняется в AST как атрибут декларации, но это только подсказка
компилятору; реальное решение о встраивании принимается позже на уровне IR.

### 3.3 LLVM IR при `-O0`

```bash
clang -O0 -S -emit-llvm main.cpp -o main_O0.ll
```

```llvm
; Function Attrs: noinline nounwind optnone uwtable
define linkonce_odr dso_local i32 @_Z6squarei(i32 noundef %0) #0 {
  %2 = alloca i32, align 4
  store i32 %0, i32* %2, align 4
  %3 = load i32, i32* %2, align 4
  %4 = load i32, i32* %2, align 4
  %5 = mul nsw i32 %3, %4
  ret i32 %5
}

; Function Attrs: noinline nounwind optnone uwtable
define dso_local i32 @main() #0 {
  %1 = alloca i32, align 4
  %2 = alloca i32, align 4
  %3 = alloca i32, align 4
  store i32 0, i32* %1, align 4
  store i32 5, i32* %2, align 4
  %4 = load i32, i32* %2, align 4
  %5 = call i32 @_Z6squarei(i32 noundef %4)
  store i32 %5, i32* %3, align 4
  %6 = load i32, i32* %3, align 4
  ret i32 %6
}
```

Наблюдения:
- атрибут `optnone noinline` запрещает встраивание;
- `square` имеет linkage `linkonce_odr` (стандартное поведение inline-функций в C++);
- все переменные размещены через `alloca`, обмен данными через `load`/`store`;
- вызов `square` присутствует как полноценный `call`.

### 3.4 LLVM IR при `-O2`

```bash
clang -O2 -S -emit-llvm main.cpp -o main_O2.ll
```

```llvm
; Function Attrs: mustprogress nofree norecurse nosync nounwind
;                 readnone uwtable willreturn
define dso_local i32 @main() local_unnamed_addr #0 {
  ret i32 25
}
```

Что произошло:
- `mem2reg` устранил `alloca`/`load`/`store`, IR переведён в SSA-форму;
- `inline` встроил `square` в `main` (функция короткая, вызывается один раз,
  имеет linkonce_odr linkage и пользовательский `inline`);
- `sccp` + `instcombine` свернули `5 * 5` в константу `25`;
- определение `square` удалено как мёртвый код (`globaldce`);
- `main` получила атрибуты `readnone willreturn` — LLVM формально доказал
  отсутствие побочных эффектов.

**Ответ:** функция `square` встроена при `-O2` и удалена из IR полностью.

### 3.5 Применение `-always-inline`

Изменим объявление `square`:

```cpp
__attribute__((always_inline)) inline int square(int x) {
    return x * x;
}
```

```bash
clang -O0 -S -emit-llvm main.cpp -o main_O0.ll
opt -passes=always-inline -S main_O0.ll -o main_ainl.ll
```

В `main_ainl.ll`:

```llvm
define dso_local i32 @main() #0 {
  %1 = alloca i32, align 4
  %2 = alloca i32, align 4
  %3 = alloca i32, align 4
  store i32 0, i32* %1, align 4
  store i32 5, i32* %2, align 4
  %4 = load i32, i32* %2, align 4
  ; --- встроенное тело square ---
  %5 = alloca i32, align 4
  store i32 %4, i32* %5, align 4
  %6 = load i32, i32* %5, align 4
  %7 = load i32, i32* %5, align 4
  %8 = mul nsw i32 %6, %7
  ; --- конец встроенного тела ---
  store i32 %8, i32* %3, align 4
  %9 = load i32, i32* %3, align 4
  ret i32 %9
}
```

Сравнение трёх вариантов:

| Режим | Встраивание | `alloca`/`load`/`store` | Свёртка `5*5 = 25` |
|---|---|---|---|
| `-O0` | нет | да | нет |
| `-O0 + -always-inline` | да | да | нет |
| `-O2` | да | нет | да |

`-always-inline` выполняет **только** подстановку тела. Для полной оптимизации
нужны сопровождающие проходы (`mem2reg`, `sccp`, `instcombine`, `simplifycfg`, `dce`).

### 3.6 CFG до и после

```bash
clang -O0 -S -emit-llvm main.cpp -o main_O0.ll
opt -dot-cfg -disable-output main_O0.ll
dot -Tpng .main.dot         -o cfg_main_O0.png
dot -Tpng ._Z6squarei.dot   -o cfg_square_O0.png

clang -O2 -S -emit-llvm main.cpp -o main_O2.ll
opt -dot-cfg -disable-output main_O2.ll
dot -Tpng .main.dot -o cfg_main_O2.png
```

CFG функции `square` при `-O0` — один базовый блок (`alloca`, два `load`, `mul`, `ret`).
CFG функции `main` при `-O0` — один базовый блок с `alloca` × 3, серией `store`/`load`,
вызовом `call @_Z6squarei` и `ret`.
CFG функции `main` при `-O2` — один базовый блок с единственной инструкцией
`ret i32 25`. CFG функции `square` при `-O2` не существует (определение удалено).

Структура графа не изменилась (один блок входа = блок выхода), но содержимое
блока радикально сократилось.

### 3.7 Выводы по индивидуальному заданию

1. Ключевое слово `inline` в C/C++ — рекомендация, а не приказ; решение
   принимает inliner на уровне LLVM IR.
2. При `-O0` функции получают атрибуты `noinline optnone`, и встраивание
   полностью отключено.
3. При `-O2` маленькая нерекурсивная функция, вызываемая один или несколько
   раз, почти всегда встраивается, особенно если её linkage допускает удаление
   (`static`, `internal`, `linkonce_odr`).
4. Гарантированное встраивание обеспечивает `__attribute__((always_inline))`;
   соответствующий pass работает даже на `-O0`.
5. `-always-inline` подставляет только тело — без сопутствующих pass-ов
   реальной выгоды (свёртка констант, регистровая форма) не будет.
6. После встраивания и оптимизаций определение исходной функции может быть
   удалено полностью (в нашем примере `square` исчезает из `main_O2.ll`).
7. Встраивание не всегда выгодно: большие, рекурсивные функции и функции с
   виртуальными вызовами inliner оставляет нетронутыми, чтобы не «раздуть»
   код и не ухудшить локальность инструкционного кэша.

---



---

## 5. Приложение — структура проекта

```
.
├── README.md             — этот файл
├── main.c                — исходник общей части
├── main.cpp              — исходник индивидуального задания
├── main_O0.ll            — IR без оптимизаций
├── main_O2.ll            — IR с -O2
├── main_ainl.ll          — IR после -always-inline (без -O2)
├── cfg_main_O0.png       — CFG main при -O0
├── cfg_main_O2.png       — CFG main при -O2
├── cfg_square_O0.png     — CFG square при -O0
└── lab7_2_10_functions.docx — отчёт в Word
```
