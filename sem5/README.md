# 5. Вспомогательные инструменты: valgrind, gdb

## Пререквизиты
* [WSL, консоль](../sem1)

## Семинар

Никто не пишет идеальный код, в нём всегда будут случаться ошибки. Приходится отлаживать свою программу. Чаще всего самым удобным и гибким способом оказывается отладками printf-ами или же логами. Если вы заранее подумали об этом и расставили дебажные сообщения в своей программе то, что за ошибка произошла, часто можно понять по этим сообщениям, или по крайней мере сильно локализировать проблему. Это первое с чего нужно начинать отладку. Чем сильнее вы сузите область, где может происходить ошибка тем лучше. Но не все ошибки можно обнаружить с помощью логов, например утечки памяти обнаружить не получится, для этого к нам на помощь приходят другие инструменты.

### Санитайзеры и valgrind

Начнём с `valgrind`, существует он только под Linux (WSL) и платформу x86-64.

Для его установки необходимо ввести команду (на Ubuntu):

```
sudo apt install valgrind
```

Пример запуска valgrind на программе с ошибками:

```
$ valgrind --leak-check=full ./a.out
==10578== Conditional jump or move depends on uninitialised value(s)
==10578==    at 0x484F229: strlen (in /usr/libexec/valgrind/vgpreload_memcheck-amd64-linux.so)
==10578==    by 0x49C8DA7: __printf_buffer (vfprintf-process-arg.c:435)
==10578==    by 0x49EDCC5: __vsnprintf_internal (vsnprintf.c:96)
==10578==    by 0x49C4405: snprintf (snprintf.c:31)
==10578==    by 0x10C65B: GetNodeLabel(Node const*) (graph_dump.cpp:32)
==10578==    by 0x10CD3A: GenerateGraph2(Node*, char*, int*, unsigned long) (graph_dump.cpp:171)
==10578==    by 0x10CE6E: GenerateGraph2(Node*, char*, int*, unsigned long) (graph_dump.cpp:188)
==10578==    by 0x10CE6E: GenerateGraph2(Node*, char*, int*, unsigned long) (graph_dump.cpp:188)
==10578==    by 0x10CF83: TreeDumpDot2(Node*) (graph_dump.cpp:210)
==10578==    by 0x10A71F: main (main.cpp:46)
==10578==
==10578==
==10578== HEAP SUMMARY:
==10578==     in use at exit: 360 bytes in 9 blocks
==10578==   total heap usage: 907 allocs, 898 frees, 177,973 bytes allocated
==10578==
==10578== 360 (40 direct, 320 indirect) bytes in 1 blocks are definitely lost in loss record 9 of 9
==10578==    at 0x484D953: calloc (in /usr/libexec/valgrind/vgpreload_memcheck-amd64-linux.so)
==10578==    by 0x10A8D6: NewNode(NodeType, NodeValue, Node*, Node*) (read_expr.cpp:34)
==10578==    by 0x10E318: GetP(Lexeme*, unsigned long*) (syntaxis_analysis.cpp:100)
==10578==    by 0x10E12D: GetD(Lexeme*, unsigned long*) (syntaxis_analysis.cpp:78)
==10578==    by 0x10DF2B: GetT(Lexeme*, unsigned long*) (syntaxis_analysis.cpp:54)
==10578==    by 0x10DD38: GetE(Lexeme*, unsigned long*) (syntaxis_analysis.cpp:30)
==10578==    by 0x10DB48: GetG(Lexeme*, unsigned long*) (syntaxis_analysis.cpp:15)
==10578==    by 0x10A892: ReadExpression(char const*) (read_expr.cpp:26)
==10578==    by 0x10A637: main (main.cpp:20)
==10578==
==10578== LEAK SUMMARY:
==10578==    definitely lost: 40 bytes in 1 blocks
==10578==    indirectly lost: 320 bytes in 8 blocks
==10578==      possibly lost: 0 bytes in 0 blocks
==10578==    still reachable: 0 bytes in 0 blocks
==10578==         suppressed: 0 bytes in 0 blocks
==10578==
==10578== Use --track-origins=yes to see where uninitialised values come from
==10578== For lists of detected and suppressed errors, rerun with: -s
==10578== ERROR SUMMARY: 2 errors from 2 contexts (suppressed: 0 from 0)
```

Видим, что он поймал две ошибки, это утечку памяти и подсветил, где она была выделена. А также обнаружил неопределенное поведение: чтение из неинициализированной памяти.

Теперь рассмотрим санитайзеры (sanitizers). Для их использования необходимо включить указать следующие флаги компиляции:

Для Linux и MacOs (x86-64)
```
-fsanitize=address,alignment,bool,bounds,enum,float-cast-overflow,float-divide-by-zero,integer-divide-by-zero,leak,nonnull-attribute,null,object-size,return,returns-nonnull-attribute,shift,signed-integer-overflow,undefined,unreachable,vla-bound,vptr
```

Для MacOs (ARM)
```
-fsanitize=address,alignment,bool,bounds,enum,float-cast-overflow,float-divide-by-zero,integer-divide-by-zero,nonnull-attribute,null,return,returns-nonnull-attribute,shift,signed-integer-overflow,undefined,unreachable,vla-bound,vptr
```

Пример использования:

```
$ ./a.out
==15995==ERROR: AddressSanitizer: stack-use-after-return on address 0x7f32269093c0 at pc 0x7f3229aa1a6a bp 0x7ffc11f9d3c0 sp 0x7ffc11f9cb38
READ of size 2 at 0x7f32269093c0 thread T0
    #0 0x7f3229aa1a69 in printf_common ../../../../src/libsanitizer/sanitizer_common/sanitizer_common_interceptors_format.inc:563
    #1 0x7f3229ace5f6 in vsnprintf ../../../../src/libsanitizer/sanitizer_common/sanitizer_common_interceptors.inc:1652
    #2 0x7f3229ad0786 in snprintf ../../../../src/libsanitizer/sanitizer_common/sanitizer_common_interceptors.inc:1723
    #3 0x649da54a6b9e in GetNodeLabel(Node const*) (/home/panterrich/Documents/Differentiator/do+0x29b9e) (BuildId: be4369b0293e5d90043f97d4ab1680aebdd01501)
    #4 0x649da54a8340 in GenerateGraph2(Node*, char*, int*, unsigned long) (/home/panterrich/Documents/Differentiator/do+0x2b340) (BuildId: be4369b0293e5d90043f97d4ab1680aebdd01501)
    #5 0x649da54a89a4 in GenerateGraph2(Node*, char*, int*, unsigned long) (/home/panterrich/Documents/Differentiator/do+0x2b9a4) (BuildId: be4369b0293e5d90043f97d4ab1680aebdd01501)
    #6 0x649da54a89a4 in GenerateGraph2(Node*, char*, int*, unsigned long) (/home/panterrich/Documents/Differentiator/do+0x2b9a4) (BuildId: be4369b0293e5d90043f97d4ab1680aebdd01501)
    #7 0x649da54a8ded in TreeDumpDot2(Node*) (/home/panterrich/Documents/Differentiator/do+0x2bded) (BuildId: be4369b0293e5d90043f97d4ab1680aebdd01501)
    #8 0x649da549dc74 in main (/home/panterrich/Documents/Differentiator/do+0x20c74) (BuildId: be4369b0293e5d90043f97d4ab1680aebdd01501)
    #9 0x7f3228a2a1c9 in __libc_start_call_main ../sysdeps/nptl/libc_start_call_main.h:58
    #10 0x7f3228a2a28a in __libc_start_main_impl ../csu/libc-start.c:360
    #11 0x649da549d844 in _start (/home/panterrich/Documents/Differentiator/do+0x20844) (BuildId: be4369b0293e5d90043f97d4ab1680aebdd01501)

Address 0x7f32269093c0 is located in stack of thread T0 at offset 64 in frame
    #0 0x649da549fd49 in ParseMathExpr(Node, char, Node*) (/home/panterrich/Documents/Differentiator/do+0x22d49) (BuildId: be4369b0293e5d90043f97d4ab1680aebdd01501)

  This frame has 2 object(s):
    [48, 52) 'offset' (line 214)
    [64, 74) 'value_buf' (line 213) <== Memory access at offset 64 is inside this variable
HINT: this may be a false positive if your program uses some custom stack unwind mechanism, swapcontext or vfork
      (longjmp and C++ exceptions *are* supported)
SUMMARY: AddressSanitizer: stack-use-after-return ../../../../src/libsanitizer/sanitizer_common/sanitizer_common_interceptors_format.inc:563 in printf_common
Shadow bytes around the buggy address:
  0x7f3226909100: f5 f5 f5 f5 f5 f5 f5 f5 f5 f5 f5 f5 00 00 00 00
  0x7f3226909180: f5 f5 f5 f5 f5 f5 f5 f5 f5 f5 f5 f5 00 00 00 00
  0x7f3226909200: f5 f5 f5 f5 f5 f5 f5 f5 f5 f5 f5 f5 00 00 00 00
  0x7f3226909280: f5 f5 f5 f5 f5 f5 f5 f5 f5 f5 f5 f5 00 00 00 00
  0x7f3226909300: f5 f5 f5 f5 f5 f5 f5 f5 f5 f5 f5 f5 00 00 00 00
=>0x7f3226909380: f5 f5 f5 f5 f5 f5 f5 f5[f5]f5 f5 f5 00 00 00 00
  0x7f3226909400: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x7f3226909480: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x7f3226909500: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x7f3226909580: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x7f3226909600: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
Shadow byte legend (one shadow byte represents 8 application bytes):
  Addressable:           00
  Partially addressable: 01 02 03 04 05 06 07
  Heap left redzone:       fa
  Freed heap region:       fd
  Stack left redzone:      f1
  Stack mid redzone:       f2
  Stack right redzone:     f3
  Stack after return:      f5
  Stack use after scope:   f8
  Global redzone:          f9
  Global init order:       f6
  Poisoned by user:        f7
  Container overflow:      fc
  Array cookie:            ac
  Intra object redzone:    bb
  ASan internal:           fe
  Left alloca redzone:     ca
  Right alloca redzone:    cb
==15995==ABORTING
```

`Valgrind` использует динамическую инструментацию, в то время как санитайзеры - статическую из-за чего необходимо перекомпилировать проект. Также `valgrind` находит меньше проблем согласно [статье](https://developers.redhat.com/blog/2021/05/05/memory-error-checking-in-c-and-c-comparing-sanitizers-and-valgrind#datarace) и намного сильнее замедляет исполнение программ в сравнении с санитайзерами.

> [!Warning]
> Санитайзеры не совместимы с `valgrind`.

> [!NOTE]
> Помните, чтобы выводы в отчетах были полными и понятными, необходимо подключать дебаг информацию с помощью флага `-g` (`-g3`) во время компиляции.


## Отладчик (debugger) - GDB

Как уже отмечалось, отладка с помощью логов не всегда удобна, особенно, если проект большой и его пересборка и запуск занимает много времени. В некоторых случаях удобным способом оказывается отладчик (debugger). Рассмотрим на примере `gdb`.

Для запуска программы под отладчиком необходимо скомпилировать её без оптимизаций (`-O0`) и добавить дебаг информацию (`-ggdb3`). Оптимизации необходимо выключить, чтобы сохранить соответствие строк кода и машинным инструкциям. Дебаг информация просто необходима отладчику для инструментирования кода.

`Gdb` - это очень мощный инструмент, но в данном семинаре остановимся на базовом функционале.

Для запуска программы в отладчике необходимо написать команду

```
gdb ./a.out
```

Откроется консольное меню дебагера. Чтобы перейти в начало функции `main()` необходимо ввести

```
> start
```

Чтобы открыть классное консольное меню, зажмите `ctrl+x+a` или же впишите команду:

```
> tui enable
```

Список команд:

1. `n` (next) - пройти строку, не заходя в функцию.
2. `s` (step) - шаг с заходом в функцию.
3. `ctrl+l` - очистка вывода программы с экрана отладчика.
4. `p <expr>` - распечатать значение переменной или выражения.
5. `b <n/func>` - поставить breakpoint на строку или номер функции.
6. `info breakpoints` - посмотреть расставленные breakpoint
7. `delete <n>` - удалить объект по номеру (например, breakpoint).
8. `c` (continue) - идти до первого прерывания (breakpoint)
9. `command <n>` - привязать команду по номеру объекта.
10. `set pagination off` - отключить пагинацию

