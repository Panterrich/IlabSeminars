# 3. Системы сборки

## Введение

Когда вы пишите свои проекты, то зачем-то разбиваете их на файлы, некоторые из которых являются единицами трансляции (.с), а другие (.h) иклюдятся в другие файлы. Для чего это делается? В первую очередь, чтобы разделить логику и меньше держать контекста в голове. Работая по такому принципу, вы одновременно взаимодействуете с меньшим количеством
сущностей, это значительно облегчает написание кода.

Прежде чем переходить к сборке проекта, вспомним как компилятор узнает об именах.

<figure>
    <img src="images/names.png" alt="Связь имен">
</figure>

Итак, у нас есть файлы, которые объявляют имена, которые определяют имена и те, которые используют имена.

Есть два типа зависимостей, которые должны быть разрешены на разных стадиях сборки. Первый тип - стрелки из синих в черные блоки, должно быть разрешены на этапе компиляции. Второй тип - стрелки из черных черные блоки, должны быть разрешены на этапе линковки, определение должно слинковаться с использованием имени.

Имена могут быть объявлены, определены и затем использованы в рамках одного файла, но более типичная история, это когда нам необходимо переиспользовать функцию из другого модуля. Тогда мы декларацию функции (прототип) выносим в заголовочный файл. Определение функции кладем в какую-то единицу трансляции (.с), при этом в этом же файле указываем декларацию с помощью подключения заголовочного файла. Затем
в единице трансляции, где мы вызываем это функцию, также необходимо прокинуть декларацию функции: подключить соответствующий заголовочный файл.

Итак, мы узнали как у нас выглядит привычный принцип передачи имен компилятору для сборки.

## Project layout

Теперь перейдем к типичному устройству репозитория проектов на С/C++. Как уже отмечалось ранее, все программисты стремятся сделать разделение на модули, поэтому любой проект представляет собой древовидную структуру, состоящую из модулей.

Модуль представляет собой набор исходников (единиц трансляции), которые лежат в директории с типичным названием `src`/`source`, и набор публичных заголовочных файлов, которые лежат в директории `include`. Если модуль содержит тесты, то их также могут положить в директорию `test`.

Если репозиторий не является отдельной библиотекой, т.е. у него существует точка входа (функция main()), то её могут положить как в корень репозитория, так и в папку src корневого модуля.

Некоторые заголовочные файлы могут лежать внутри директории src, если они нужны только в рамках текущего модуля, и имена, хранящиеся в данном файле, не планируются экспортировать.

> [!NOTE]
> Если вы решили поделить файлы по указанной структуре, не забудьте, что можно не указывать самостоятельно относительные пути заголовочных файлов из директории `include`, а добавить флаг `-I` с указанием пути директории `include`, тогда компилятор будет искать заголовочные файлы и там.

## Системы сборки: уровень 0, bash-скрипт

Перейдем к системам сборки. Пусть для простоты у нас есть репозиторий с одним модулем. Нам необходимо собрать исполняемый файл нашей программы. Вы уже умеете компилировать программу с помощью командной строки, но чтобы каждый раз её не прописывать, можно записать эту команду в файл и вызывать скрипт каждый раз по необходимости.

Пример:

```shell
#!/bin/bash

g++ src/arg_parser.cpp src/logger.cpp src/test_library.cpp \
src/buffer_clean.cpp src/main.cpp src/test_solve_quadr.cpp \
src/compare_double.cpp src/show_results.cpp src/get_data.cpp \
src/solve_quadr.cpp -Iinclude/ -std=c++17 -o square

```

> [!NOTE]
> Не забудьте сделать скрипт исполняемым с помощью команды `chmod +x build.sh`

Давайте разобьём эту команду на шаги, которые компилятор сам выполняет в процессе. Позже будет понятно зачем.

```shell
#!/bin/bash

FLAGS=""

g++ $FLAGS -c -I./include/ ./src/arg_parser.cpp -o arg_parser.o
g++ $FLAGS -c -I./include/ ./src/logger.cpp -o logger.o
g++ $FLAGS -c -I./include/ ./src/test_library.cpp -o test_library.o
g++ $FLAGS -c -I./include/ ./src/buffer_clean.cpp -o buffer_clean.o
g++ $FLAGS -c -I./include/ ./src/main.cpp -o main.o
g++ $FLAGS -c -I./include/ ./src/test_solve_quadr.cpp -o test_solve_quadr.o
g++ $FLAGS -c -I./include/ ./src/compare_double.cpp -o compare_double.o
g++ $FLAGS -c -I./include/ ./src/show_results.cpp -o show_results.o
g++ $FLAGS -c -I./include/ ./src/get_data.cpp -o get_data.o
g++ $FLAGS -c -I./include/ ./src/solve_quadr.cpp -o solve_quadr.o

g++ $FLAGS arg_parser.o logger.o test_library.o buffer_clean.o main.o \
 test_solve_quadr.o compare_double.o show_results.o get_data.o solve_quadr.o -o square

```

Т.е. сначала идет компиляция каждой единицы трансляции, затем идёт линковка всех полученных объектников (.o) в конечный исполняемый файл.

## Системы сборки: уровень 1, базовый makefile

Что плохого в написанном выше скрипте? Мы пересобираем каждый раз весь проект. Если проект большой, то его полная пересборка измеряется часами. Часто в процессе разработки меняется только небольшая часть файлов, и соответственно необходимо пересобирать только те файлы, которые вы поменяли, и те файлы, которые зависели от них. Это существенно ускоряет пересборку проекта. Такой принцип называется **раздельной компиляцией**.

Что ещё нам не нравится в предыдущем варианте? Язык bash-скриптов не декларативный: постоянно приходится описывать **как** сделать вместо того, чтобы указать **что** нужно сделать. Хочется просто описать зависимости файлов, а не процесс сборки.

На основе решений этих проблем был придуман Makefile.

Зависимости проекта, который можно собрать, всегда образуют **ациклический** однонаправленный граф. Это значит, что мы всегда можем описать файлы, которые уже есть и которые нужны. Makefile строит этот граф, и запускает сборку в этом топологическом порядке, пересобирая только те цели, зависимости которых изменились.

Пример написания цели (target) следующий:

```make
arg_parser.o: ./src/arg_parser.cpp
    g++ $(FLAGS) -c -I./include/ ./src/arg_parser.cpp -o arg_parser.o
```

Или в общем случае:

```make
<target>: [<requisites>]
    [shell commands]
```

Таким образом задаётся, что будет собирать данная цель и через двоеточие указываются все зависимости, а затем рецепт: как эту цель собрать.

> [!Warning]
> В Makefile обязательно используется табуляция.

Любая цель в Makefile по-умолчанию считается файлом, и make следит за датой обновления этого файла. Цель будет пересобираться, если изменился хотя бы один реквизит, либо если реквизита вообще нет.

Перепишем наш скрипт на makefile:

```make
FLAGS = -O2

all: arg_parser.o logger.o test_library.o buffer_clean.o main.o test_solve_quadr.o compare_double.o \
show_results.o get_data.o solve_quadr.o
	g++ $(FLAGS) arg_parser.o logger.o test_library.o buffer_clean.o main.o \
	test_solve_quadr.o compare_double.o show_results.o get_data.o solve_quadr.o -o square

arg_parser.o: ./src/arg_parser.cpp
	g++ $(FLAGS) -c -I./include/ ./src/arg_parser.cpp -o arg_parser.o

logger.o: ./src/logger.cpp
	g++ $(FLAGS) -c -I./include/ ./src/logger.cpp -o logger.o

test_library.o: ./src/logger.cpp
	g++ $(FLAGS) -c -I./include/ ./src/test_library.cpp -o test_library.o

buffer_clean.o: ./src/buffer_clean.cpp
	g++ $(FLAGS) -c -I./include/ ./src/buffer_clean.cpp -o buffer_clean.o

main.o: ./src/main.cpp
	g++ $(FLAGS) -c -I./include/ ./src/main.cpp -o main.o

test_solve_quadr.o: ./src/test_solve_quadr.cpp
	g++ $(FLAGS) -c -I./include/ ./src/test_solve_quadr.cpp -o test_solve_quadr.o

compare_double.o: ./src/compare_double.cpp
	g++ $(FLAGS) -c -I./include/ ./src/compare_double.cpp -o compare_double.o

show_results.o: ./src/show_results.cpp
	g++ $(FLAGS) -c -I./include/ ./src/show_results.cpp -o show_results.o

get_data.o: ./src/get_data.cpp
	g++ $(FLAGS) -c -I./include/ ./src/get_data.cpp -o get_data.o

solve_quadr.o: ./src/solve_quadr.cpp
	g++ $(FLAGS) -c -I./include/ ./src/solve_quadr.cpp -o solve_quadr.o
```

Отлично, но у нас теперь в нашей рабочей директории куча объектников. Давайте добавим в makefile следующую цель:

```make
clean:
    rm -rf *.o
```

Как уже говорилось ранее, clean будет считаться за файл, но так как его не существует и он не появляется при выполнении данной команды, то эта цель будет отрабатывать каждый раз.

Давайте перечислим некоторые проблемы данного makefile:

1. Компилятор прибит гвоздями к каждой команде. Хочется иметь возможность быстро подменить компилятор на другой.
2. Часто дублируется конструкция `-I./include`, можно вынести в переменную.
3. Если в проекте будет файл с именем `clean`, то таргет `clean` перестанет отрабатывать.
4. Много однотипных целей.

Исправим недоразумение с `clean`:

```make
.PHONY: clean
clean:
    rm -rf *.o
```

`PHONY` таргеты это специальные таргеты, которые не соответствуют никаким результирующим файлам. Обычно такими таргеты становятся `all`, `clean`, `install`, `check` и другие. Рекомендуется помечать специальные таргеты так, чтобы случайный файл с таким именем в директории не помешал сборке.

### Переменные в Makefile

Переменные в Makefile занимают важную позицию.
Есть некоторый список стандартных переменных предопределенных в make, которые необходимо рассмотреть:
1. $(CC) и $(CXX) - компиляторы С и C++
2. $(CFLAGS) и $(CXXFLAGS) - флаги компиляции С и С++
3. $(CPPFLAGS) - флаги препроцессора С
4. $(LDFLAGS) - флаги линковщика

Давайте разберемся со следующим примером

```make
bar = Hello $(baz)
baz = World
quux = $(baz)
qux = Hello $(quux)
quux = Make

.PHONY: all
all:
    @echo "bar = $(bar)"
    @echo "qux = $(qux)"
```

> [!Note]
> По умолчанию make показывает все исполняемые команды, иногда имеет смысл это подавить. Делается это с помощью `@` перед командой.

```
$ make -f lazy.mk
bar = Hello World
qux = Hello Make
```

Почему так? Переменные в makefile вычисляются лениво. Раскрытие переменных происходит в самый последний момент - в момент использования.

### Энергичные присвоения

А что будет выводиться в этом случае?

```make
bar := Hello $(baz)
baz := World

.PHONY: all
all:
    @echo "bar := $(bar)"
```

```
$ make -f energy.mk
bar := Hello
```

Неинициализированная переменная, которая используется в энергичном присвоении, будет пустой строкой.

Если переменная не создана внутри Makefile, то make будет искать это же имя в переменных окружения. Переменные окружения -
это специальные переменные, определенные оболочкой (shell) и используемые программами во время выполнения. Они могут определяться системой и пользователем.

Чтобы записать, дописать или переписать переменную, заданную снаружи, надо указать override.

```make
override CXXFLAGS += -I./include

.PHONY: all
all:
    @echo $(CXXFLAGS)
```

```
$ make CXXFLAGS="-g -O0"
-g -O0 -I./include
```

### Параллельность в makefile
Давайте посмотрим следующий пример:

```make
SUBDIRS = foo bar baz

.PHONY: subdirs
subdirs:
    for dir in $(SUBDIRS); do \
        $(MAKE) -C $dir; \
    done
```

Команда `make -C $dir` запускает рекурсивно makefile в указанной директории. Makefile можно запускать параллельно с помощью флага `-jN`, где `N` - количество воркеров (обычно ставят не больше числа потоков, учитывая количество оперативной памяти). В данном случае параллельность была утеряна, так как make не знает о зависимостях и последовательно запускает рекурсивный make, а также теряется сообщение об ошибках внутри запуска makefile.

Перепишим следующим образом:

```make
SUBDIRS = foo bar baz

.PHONY: subdirs
subdirs: $(SUBDIRS)

.PHONY: $(SUBDIRS)
$(SUBDIRS):
    @$(MAKE) -C $@
```

> [!Note]
> `$@` - автоматическая переменная, которая означает имя цели.

Получаем, что список директорий задаёт несколько псевдоцелей, каждая из которых вызывает соответствующий make, и есть псевдоцель, которая зависит от списка этих псевдоцелей.

> [!Note]
>  Если запустить make c флагом `-k`, то make будет игнорировать ошибки и пытаться собрать всё, что может.

### Автоматические переменные

Мы уже успели познакомиться с автоматической переменной `$@`. Рассмотрим пример, который избавит нас от другой упомянутой проблемы:

```make
arg_parser.o: ./src/arg_parser.cpp
	g++ $(CXXFLAGS) -c $^ -o $@
```

Итого:

1. `$@` - имя таргета
2. `$^` - имена всех реквизитов
3. `$<` - имя первого реквизита
4. `$(@D)` - часть имени относящаяся к директории
5. `$(@F)` - часть имени относящаяся к файлу

## Системы сборки: уровень 2, любительский makefile

Объединяем предыдущие результаты и получаем:

```make
CXX = g++
CXXFLAGS ?= -O2
COMMONOTIC = -I./include

override CXXFLAGS += $(COMMONOTIC)

.PHONY: all
all: square

square: arg_parser.o logger.o test_library.o buffer_clean.o main.o test_solve_quadr.o compare_double.o \
show_results.o get_data.o solve_quadr.o
    $(CXX) $^ -o $@ $(LDFLAGS)

arg_parser.o: ./src/arg_parser.cpp
	$(CXX) $(CXXFLAGS) -c $^ -o $@

logger.o: ./src/logger.cpp
	$(CXX) $(CXXFLAGS) -c $^ -o $@

test_library.o: ./src/logger.cpp
	$(CXX) $(CXXFLAGS) -c $^ -o $@

buffer_clean.o: ./src/buffer_clean.cpp
	$(CXX) $(CXXFLAGS) -c $^ -o $@

main.o: ./src/main.cpp
	$(CXX) $(CXXFLAGS) -c $^ -o $@

test_solve_quadr.o: ./src/test_solve_quadr.cpp
	$(CXX) $(CXXFLAGS) -c $^ -o $@

compare_double.o: ./src/compare_double.cpp
	$(CXX) $(CXXFLAGS) -c $^ -o $@

show_results.o: ./src/show_results.cpp
	$(CXX) $(CXXFLAGS) -c $^ -o $@

get_data.o: ./src/get_data.cpp
	$(CXX) $(CXXFLAGS) -c $^ -o $@

solve_quadr.o: ./src/solve_quadr.cpp
	$(CXX) $(CXXFLAGS) -c $^ -o $@

.PHONY: clean
clean:
    rm -rf *.o
```

### Функции в makefile

Функция вызывается как `$(function [<arguments>])`

```make
cppfiles = main1.cpp foo.cpp main2.cpp bar.cpp
mains = main1.cpp main2.cpp
filtered = $(filter-out $(mains),$(cppfiles))
```

В переменную filtered попадёт `foo.cpp` и `bar.cpp`.

Из всех функций выделим функцию `patsubst`, её аргументы: pattern, replacement, text.

```
objs = $(patsubst %.cpp, %.0, $(cppfiles))
```

Эквивалентно другой конструкции `$(var:pattern=replacement)`
```
ofilt = $(filtered:%.cpp=%.o)
```

Эта конструкция более часто используемая.

## Pattern rules

Напишим общий рецепт для сборки всех объектников

```make

CPPSRC = src/arg_parser.cpp src/logger.cpp src/test_library.cpp src/buffer_clean.cpp \
    src/main.cpp src/test_solve_quadr.cpp src/compare_double.cpp src/show_results.cpp \
    src/get_data.cpp src/solve_quadr.cpp

CPPOBJ = $(CPPSRC:%.cpp=%.o)

%.o : %.cpp
    $(CXX) $(CXXFLAGS) -c $< -o $@

square: $(CPPOBJ)
    $(CXX) $^ -o $@ $(LDFLAGS)

```

Мы написали pattern rule, но ровно такой же предопределен в makefile, поэтому его можно опустить, так как он является implicit. Но хороший тон это использовать для ваших переменных static pattern rules:

```
$(CPPOBJ): %.o: %cpp
     $(CXX) $(CXXFLAGS) -c $< -o $@
```

Проблема: объектные файлы начнут создавать внутри папок. Временно сделаем по другому:

```make

CPPSRC = src/arg_parser.cpp src/logger.cpp src/test_library.cpp src/buffer_clean.cpp \
    src/main.cpp src/test_solve_quadr.cpp src/compare_double.cpp src/show_results.cpp \
    src/get_data.cpp src/solve_quadr.cpp

CPPOBJ = arg_parser.o logger.o test_library.o buffer_clean.o \
    main.o test_solve_quadr.o compare_double.o show_results.o \
    get_data.o solve_quadr.o

$(CPPOBJ): %.o : src/%.cpp
    $(CXX) $(CXXFLAGS) -c $< -o $@

square: $(CPPOBJ)
    $(CXX) $^ -o $@ $(LDFLAGS)

```

> [!CAUTION]
> Есть соблазн сделать так
> ```make
> CPPSRC = $(wildcard src/*.cpp)
> ```
> Само по себе wildcard не плохи, но плохо делать своими таргетами весь мусор, который wildcard найдет в директории `src`.
> Опыт показывает, что списки файлов лучше прибивать намертво простым перечислением.

## Headers

На этом мы победили makefile, но кажется мы не заметили слона. Makefile ничего не знает о зависимостях по заголовочным файлам. Он не будет пересобирать таргет, даже если она зависит от изменившегося хедера. На помощь приходит сам компилятор:

```shell
$ g++ -I./include -E src/arg_parser.cpp -MM -MT arg_parser.o

arg_parser.o: src/arg_parser.cpp include/arg_parser.h include/print_colors.h \
  include/define_consts.h
```

Отлично пропишим включение файлов зависимостей автоматически при сборке объектного файла при помощи чуть других опций компилятора:

```make
$(CPPOB): %.o: src/%.cpp
	$(CXX) $(CXXFLAGS) -MP -MMD -c $< -o $@
```

Данный способ сильно проще других, которые вы можете найти в интернете.

## Советы

1. Используйте стандартные переменные для компиляторов, линкеров и т.д.
2. Используйте также стандартные переменные для флагов.
3. Помечайте PHONY те таргеты, которые не соответствуют файлам.
4. Используйте override, если вы предполагаете, что переменная задаётся извне.
5. Не пишите сложные shell-скрипты внутри makefiles.
6. Используйте автоматические переменные.
7. Используйте pattern rule

## Список рекомендованных флагов

### Windows

```
-Wshadow -Winit-self -Wredundant-decls -Wcast-align -Wundef -Wfloat-equal -Winline -Wunreachable-code -Wmissing-declarations -Wmissing-include-dirs -Wswitch-enum -Wswitch-default -Weffc++ -Wmain -Wextra -Wall -g -pipe -fexceptions -Wcast-qual -Wconversion -Wctor-dtor-privacy -Wempty-body -Wformat-security -Wformat=2 -Wignored-qualifiers -Wlogical-op -Wno-missing-field-initializers -Wnon-virtual-dtor -Woverloaded-virtual -Wpointer-arith -Wsign-promo -Wstack-usage=8192 -Wstrict-aliasing -Wstrict-null-sentinel -Wtype-limits -Wwrite-strings -Werror=vla -D_DEBUG -D_EJUDGE_CLIENT_SIDE
```

### Linux and MacOs (x86-64)

```
-D _DEBUG -ggdb3 -std=c++17 -O0 -Wall -Wextra -Weffc++ -Waggressive-loop-optimizations -Wc++14-compat -Wmissing-declarations -Wcast-align -Wcast-qual -Wchar-subscripts -Wconditionally-supported -Wconversion -Wctor-dtor-privacy -Wempty-body -Wfloat-equal -Wformat-nonliteral -Wformat-security -Wformat-signedness -Wformat=2 -Winline -Wlogical-op -Wnon-virtual-dtor -Wopenmp-simd -Woverloaded-virtual -Wpacked -Wpointer-arith -Winit-self -Wredundant-decls -Wshadow -Wsign-conversion -Wsign-promo -Wstrict-null-sentinel -Wstrict-overflow=2 -Wsuggest-attribute=noreturn -Wsuggest-final-methods -Wsuggest-final-types -Wsuggest-override -Wswitch-default -Wswitch-enum -Wsync-nand -Wundef -Wunreachable-code -Wunused -Wuseless-cast -Wvariadic-macros -Wno-literal-suffix -Wno-missing-field-initializers -Wno-narrowing -Wno-old-style-cast -Wno-varargs -Wstack-protector -fcheck-new -fsized-deallocation -fstack-protector -fstrict-overflow -flto-odr-type-merging -fno-omit-frame-pointer -Wlarger-than=8192 -Wstack-usage=8192 -pie -fPIE -Werror=vla -fsanitize=address,alignment,bool,bounds,enum,float-cast-overflow,float-divide-by-zero,integer-divide-by-zero,leak,nonnull-attribute,null,object-size,return,returns-nonnull-attribute,shift,signed-integer-overflow,undefined,unreachable,vla-bound,vptr
```

### MacOs (ARM)

```
-Wall -std=c++17 -Wall -Wextra -Weffc++ -Wc++14-compat -Wmissing-declarations -Wcast-align -Wcast-qual -Wchar-subscripts -Wconversion -Wctor-dtor-privacy -Wempty-body -Wfloat-equal -Wformat-nonliteral -Wformat-security -Wformat=2 -Winline -Wnon-virtual-dtor -Woverloaded-virtual -Wpacked -Wpointer-arith -Winit-self -Wredundant-decls -Wshadow -Wsign-conversion -Wsign-promo -Wstrict-overflow=2 -Wsuggest-override -Wswitch-default -Wswitch-enum -Wundef -Wunreachable-code -Wunused -Wvariadic-macros -Wno-literal-range -Wno-missing-field-initializers -Wno-narrowing -Wno-old-style-cast -Wno-varargs -Wstack-protector -Wsuggest-override -Wbounds-attributes-redundant -Wlong-long -Wopenmp -fcheck-new -fsized-deallocation -fstack-protector -fstrict-overflow -fno-omit-frame-pointer -Wlarger-than=8192 -Wstack-protector -fPIE -Werror=vla
```

## Самостоятельная работа

1. Пройдите самостоятельно по всем шагам, которые были описаны в этом семинаре.
2. Добавьте в makefile директорию build, которая внутри себя должна повторять древовидную структуру репозитория и собирать там все артефакты сборки.
3. Напишите самостоятельно универсальный makefile, в котором необходимо менять минимальное количество вещей, при переходе на другой проект.
4. Попытайтесь написать makefile для нескольких отдельных модулей и написать обобщающий makefile.
5. Добавьте несколько режимов сборки проектов: Release (`-O2`) и Debug (`-g3 -O0`).
