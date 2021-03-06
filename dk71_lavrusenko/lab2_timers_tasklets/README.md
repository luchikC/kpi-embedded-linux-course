
Лабораторна робота №2
====
Тема: Робота з таймерами та тасклетами
====
Завдання:
===
  - Ознайомитися з таймерами та тасклетами;
  - Написати модуль ядра, який буде приймати на вхід параметри `cnt` та `delay`
    - `cnt` відповідає за кількість циклів відпрацювання тамера
    - `delay` відповідає за затримку між двома відпрацюваннями таймера;
  - При `__init`, записати лог ядра значення `jiffies`
    - після цього викликати тасклет, для запису сого знаечення `jiffies` в лог ядра;
  - Модуль повинен відпрацьовувати при `cnt` та `delay` рівним нулью;    

Хід роботи:
====

Таймери або таймери ядра - дуже важливі для управління потоку часу в коді ядра. Код ядра часто потребує затримок у виконанні деяких функцій.
Наглядний приклд використання таймера - опитування портів на отримування якоїсь інформації. 

Тасклети - це структури відкладеного виконання, які ви можете запланувати на виконання у зареєстрованих функціях.
Обробник переривань виконує невеликий обсяг робіт, а потім планує тасклети, які будуть виконані у нижній половині.

Ідеї, що лежать в основі використання тасклетів і черг робіт, були, в деякому сенсі, успадковані з вбудованих систем. 
У багатьох вбудованих системах немає традиційного планувальника, а робота просто відкладається на більш пізній термін (виконується на рівні введення / виведення або на рівні внутрішньої обробки). 
Через те, що планувальник відсутній, переривання і програми здійснюють відкладання роботи, як засіб планування робіт, які будуть пізніше виконані іншими елементами системи. 

Для виконання завадання потрібно зберігати значення `jiffies` в масиві, для того щоб показати правильну роботу таймера.
Для роботи з масивов потрібно виділити пам'ять, для виділення памяті існує багато функцій, але в нашому випадку підійде `kzalloc` або `kmalloc`.
Ці дві функції призначені для виділення пам'яті не більше "сторінки". 

Для запуску модуля потрібно спочатку скомпілювати його:
`make KBUILDDIR="<KBuild directory>"`

Запускаємо ядро за допомогою QEMU:
`qemu-system-x86_64 -enable-kvm -m 512M -smp 4 -kernel "<bzImage path>" -initrd "<path BusyBox shell utilits .gz>" \
                    -append "console=ttyS0" -nographic \
                    -drive file=fat:rw:./<path to module>,format=raw,media=disk`
                    
Підготовчі дії:
`mkdir mnt`
`mount -t vfat /dev/sda1 /mnt`

Запуск модуля:
`insmode /mnt/<module_name.ko>`

Вигрузка модуля:
`rmmod <module_name>`

Висновки:
===

В ході виконання лабараторної роботи було розроблено простий модуль ядра, який використовує таймери та тасклети. Було випробувано різні режими запуску (у вікні qemu або у власному терміналі)
Було отримано такі дані при не вказуванні параметру:

```
/ # insmod /mnt/lab2_task.ko
[  111.516141] lab2_task: loading out-of-tree module taints kernel.
[  111.517882] Invalid <cnt> <= 0 :(
insmod: can't in[  111.535961] insmod (94) used greatest stack depth: 13928 bytt
sert '/mnt/lab2_task.ko': invalid parameter
```


При вказуванні `cnt=5` та `delay=5`:

```
/ # insmod /mnt/lab2_task.ko cnt=5 delay=5
[  281.930299] 5 msec is 5 jiffies
[  281.931442] Init jiffies is 4294949086
[  281.932361] Tasklet jiffies 4294949087
```

Вимкнення модуля:

```
/ # rmmod lab2_task.ko
[  321.728772] Exit jiffies is 4294988884
[  321.731929] Array[0] = 4294949094
[  321.734517] Array[1] = 4294949100
[  321.736928] Array[2] = 4294949106
[  321.738424] Array[3] = 4294949112
[  321.739529] Array[4] = 4294949118
[  321.740465] Ave Kernel!
```

Як видно із значень `Array`, таймер затримує кожне виконання `task_func` на 5 ms. 

При вказуванні `cnt=10` та `delay=0`:

```
/ # insmod /mnt/lab2_task.ko cnt=10 delay=0
[  467.075056] 0 msec is 0 jiffies
[  467.077406] Init jiffies is 4295134232
[  467.079269] Tasklet jiffies 4295134234

```
Видно, що тасклет після запуску `_init` запускається не атомомарно, це підтверджує різницю в значеннях `jiffies` при різних запусках модуля.

Вигрузка модуля: 

```
/ # rmmod lab2_task.ko
[  473.073528] Exit jiffies is 4295140228
[  473.076852] Array[0] = 4295134237
[  473.078963] Array[1] = 4295134238
[  473.080894] Array[2] = 4295134239
[  473.082232] Array[3] = 4295134240
[  473.083232] Array[4] = 4295134241
[  473.084204] Array[5] = 4295134242
[  473.085062] Array[6] = 4295134243
[  473.086077] Array[7] = 4295134244
[  473.086787] Array[8] = 4295134245
[  473.087507] Array[9] = 4295134246
[  473.088208] Ave Kernel!
```

Вище видно, що при `delay=0` виконання відбувається на наступний такт ядра.

В роботі було використано виділення памя'ті з використанням прапору `GFP_KERNEL`, так як наша програма не  є особливо важливою в роботі, тому вона може засипати. 
Для того, щоб отримувати резервну пам'ять портібно виикористовувати прапор `GFP_ATOMIC`, що виключає можливість засипати. Також можливо поєднувати різні прапори через симовл `|`.

