# CAOS_cheatsheet

## Лекция 1 (29/10/2021): вычисления на регистрах

### История: как устроены современные компьютеры

Устроены в соотвествии с системой Джона фон Неймана. 

1. *Цифровое кодирование:* двоичное кодирование
2. У компьютера есть *память*: конечное количество одинаковых ячеек
Память адресуемая: у каждой ячейки есть номер
Всё, чем оперирует компьютер, лежит в ячейках памяти: инструкция или файл 1
3. *Арифметическо-логическое устройство и контрольное устройство*, которое за ним следит
У КУ есть какое-то ограниченное иструкций: складывать, записывать в память, ...
Инструкции закодированы в той же памяти, в том же двоичном виде, выполняются последовательно
4. Исполнение программ должно зависеть от данных: в зависимости от наших данных у нас изменяется порядок инструкций (даем команду перейти ен на следующую инструкцию, а на какую-то произвольную)

### Как у нас могут быть закодированы инструкции?

Посмотрим, что у нас внутри у программы **cat** 
```bash
objdump -d /bin/cat | less
```
Там мы сможем проглядеть, как программа превращает байты, означающие некоторые записи для процессора, в более привычный, текстовой вид команд и аргументов

*Замечание.* Аргументов у команды не более двух: бывают двухадресные, одноадресные, безадресные

Попробуем представить себе, как у нас могут выглядеть команды в процессоре (например, арифметические и логические), которые принимают на вход адреса ячеек памяти:
```
add
mul
div
and
or 
xor
...

jmp

.....  
```
Зашифруем все операции (инструкции) 1 байтом 

```asm
add 01
sub 02
mul 03

.... ff
```

Наша инструкция двухадресная и имеет вид `z += y` (при чем в порядке: *что прибавить, куда прибавить*)
```
add 01 address1 address2
add 01 y z
```
Однако заметим, что использовать обычную оперативную память -- нерационально, так как она очень медленная и операции с 32-битным адресом достаточно дорогие

Именно поэтому у нас есть, кроме обычной медленной оперативной памяти, которой у нас много, ещё одна память, которой супер мало, но зато она быстрая. Это ограниченное количество ячеек памяти прямо внутри процессора, которые называются *регистры*. Именно на регистрах мы производим вычисления.

### Архитектура x86

Наши регистры с уникальными названиями:
```
eax ebx ecx edx esi edi

epi  # интересный отдельный регистр, про него поговорим позже
```

Одинаковые ячейки памяти, каждая по 32 бита. 

Посмотрим на некоторые операции:

- копирование данных: `mov %eax, %ebx` -- копировать из `eax` в `ebx`
- сложение: `add %eax, %ebx`  -- `ebx += eax` 
- вычитание: `sub %eax, %ebx` -- `ebx -= eax` 
- побитовое И: `and %r1, %r2`  -- `r2 &= r1`
- побитовое ИЛИ: `or %r1, %r2`  -- `r2 |= r1`
- XOR: `xor %r1, %r2`
- NOT: `not %r1`

### Напишем первую программу на ассемблере (они имеют расширение .S):

Магические функции (как работают, узнаем потом):
```asm
call readi32  # считывает число из стандартного ввода и записывает в eax 
call writei32  # выводит число из eax на стандартный вывод
call finish  # завершает исполнение программы
```

**Задача: считать число, удвоить, вывести**

```asm
.global main  # метку main надо не выбрасывать, а показать операционной системе
main:  # метка, которое означает адрес, по которому эта команда останется в памяти
    call readi32
    add %eax, %eax
    call writei32
    call finish
```

**Как запустить?**

Например, можно использовать `as`, но удобнее пользоваться фронтэндом `gcc`, который смотрит на файлы-исходники и вызывает правильные программы, чтобы их запустить. Файл `simpleio_i686.S` (лежит на вики) нужен, чтобы ввод-вывод работал корректно.
```bash
gcc -m32 program.S simpleio_i686.S
```
Появится файлик с расширением .o

Можно посмотреть на disassembly через
```bash
objdump -d program.o
```

Написать Makefile (и потом запускать через `make`)
```bash
%: %.S 
gcc -m32 -o $@ $< simpleio_i686.S$
```

Полезно смотреть под отладчиком (вспоминаем курс missing semester):

- Добавляем в Makefile флаг `-g`
```bash
make program
gdb ./program 
```
- смотрим на всякое `info registers`


### А если переполнится?

Для начала вспомним, как в компьютере представлены отрицательные числа

Используем *дополнительный код*. Как у нас устроены числа:
```vim
0000000011 -- 3
0000000010 -- 2
0000000001 -- 1
0000000000 -- 0
1111111111 -- -1 (2**32 - 1)
1111111110 -- -2 (2**32 - 2)
1111111101 -- -3 (2**32 - 3)
```
Заметим, что в старшем бите положительных чисел у нас *0*, а в старшем бите отрицательных чисел -- *1*.
Тогда наибольшее положительное число `0111111111 -- 2**31 - 1`, а наименьшее отрицательное `100000000 -- -2**32`.

Заметим, что вообще говоря у нас есть бит переноса в знаковых числах, однако он не шибко помогает. Например, если мы складываем -1 и 1, то бит переноса меняется, но результат верный. Однако если мы складываем два положительных больших числа, то ответ может получиться отрицательным.

### Как ассемблер работает с переполнением?

У нас добавляется ещё один регистр `eflags`, который состоит из нескольких флагов, каждый по 1 биту. Эти флаги выставляются после каждой арифметической операции в зависимости от результата.

В частности, сейчас рассмотрим следующие флаги:

1. `ZF` -- выставляет 1, если в результате арифметической операции получился 0
2. `CF` -- carry bit (бит переноса; заметим, что он же работает для беззнакового переполнения)
3. `OF` -- signed overflow, выставляется когда случается знаковое переполнение
4. `SF` -- sign flag, верхний бит числа (получилось ли отрицательное число)

Посмотрим на примерах, как это работает. В частности, для этого нам понадобится ещё несколько инструкций:
```asm
jmp LABEL  # безусловный переход

# условные переходы:
jz LABEL  # jump if zero flag (ZF) is 1
jc LABEL # jump if CF == 1
jo LABEL # jump if OF == 1
js LABEL # jump if SF == 1

jnz LABEL  # jump if ZF == 0
jnc LABEL  # jump if CF == 0
jno LABEL  # jump if OF == 0
jns LABEL  # jump if SF == 0

cmp r1, r2  # вычислить r2-r1, результат никуда не класть
jl LABEL  # jump if less (r2 < r1)
jg LABEL  # jump if greater (r2 > r1)
jle LABEL  # jump if less or equal (r2 <= r1)
jge LABEL  # jump if greater or equal (r2 >= r1)
```

**Примеры программ с переходами**

Бесконечный цикл:
```asm
.global main

main:
    mov $0, %eax  # better: xor %eax, %eax
loop:
    add $1, %eax
    call writei32
    jmp loop
```

Считаем до 1000:
```asm
.global main

main:
    mov $0, %eax  # better: xor %eax, %eax
loop:
    add $1, %eax
    call writei32
    mov %eax, %ebx
    sub $1000, %ebx
    jz exit
    jmp loop
exit:
    call finish
```
Считаем до 1000, но по-другому:
```asm
.global main

main:
    mov $0, %eax  # better: xor %eax, %eax
loop:
    add $1, %eax
    call writei32
    cmp $1000, %eax
    jz exit
    jmp loop
exit:
    call finish
```


Считаем до 1000, но по-другому:
```asm
.global main

main:
    mov $0, %eax  # better: xor %eax, %eax
loop:
    add $1, %eax
    call writei32
    cmp $1000, %eax
    jnc exit  # jump if eax >= 1000 (unsigned)
    jmp loop
exit:
    call finish
```


### Переводим программы с C++ на asm 

Условный оператор `if`:
```
if (a > b && c > d) {
    # body
}

    cmp %ebx, %eax
    jle ...
    cm %edx, %ecx
    jle ...
    # body
endif:
    ...
```

Цикл `while`:
```
while (condition) {
    # body
}

label:
    # calculate condition
    j.. endloop
    # body
    jmp label
endloop:
```

### Хотим работать с 64битными числами

Воспользуемся следующей функцией (складывает нижние 32 бита в eax, а верхние в edx):
```asm
    call readi64   # -> edx:eax
    call writei64   # <- edx:eax
```

Попробуем сделать какие-то арифметические действия. **Складываем:**
```asm
    # Два числа: edx:eax ; ecx:ebx
    # Результат в ecx:ebx

    add %eax, %ebx
    adc %edx, %ecx  # ecx = ecx + edx + CF
```

**Вычитаем:**
```asm
    sub %eax, %ebx
    sbb %edx, %ecx  # subtract borrowed bit
```

**Сдвиги (умножение и деление на степени двойки и не только)**

Логический сдвиг:
```asm
    shl $2, %eax  # left, то же самое sll
    shr $2, %eax  # right, то же самое slr
```

Ещё сдвиги:
```asm
    rol n, reg  # is similar to shl except the shifted bits are rotated to the other end
    ror n, reg

    rcl n, reg  #  The CF is included in the rotation
    rcr n, reg

    sal n, reg  # left, арифметический сдвиг, знак числа сохранится
    sar n, reg  # right, арифметический сдвиг, знак числа сохранится
```