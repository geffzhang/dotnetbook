## Фаза планирования

> [In Progress] адаптация курса

Следующая фаза, о которой хочется поговорить - это фаза планирования. Самая интересная фаза из всех. В ее ходе происходит виртуальный GC, без физического. А физический это просто коммитмент результатов вычислений на фазе планирования.

После фазы маркировки у нас нет никакой ясности, какой тип GC должен сработать: sweep или collection (?compacting). Чтобы понять, будет ли результирующая фрагментация слишком высокой, надо, во-первых, сделать и sweep, и сжатие кучи, но в виртуальном режиме. Иначе не понять, какой процесс будет лучше. Если мы можем примерно прикинуть, то механизмы пиннинг а могут помешать этим прикидкам и испортить конечный результат GC.  Во время фазы планирования собирается информация, которая одновременно подходит для обоих алгоритмов, чтобы все сделать максимально быстро и выбрать лучший путь. После того, как GC понимает, что compacting, например, - это лучшее, что можно сделать, он его и делает. Но алгоритма, на самом деле, два. Я не слышал, чтобы были разговоры, чтобы в SOH работали оба алгоритма. До сих пор в моем представлении все выглядело так: в SOH идет сборка мусора путем сжатия куча, а в LOH - классический C++ алгоритм со свободными участками. 

В реальности в SOH работают оба алгоритма, как и в LOH. 

SOH.

Используя информацию о размерах объектов мы идем друг за другом и собираем группы свободных объектов - это plugs, и группы пропусков - gaps. То есть, когда мы пробегаем по хипу, то можем легко итерироваться. В самом начале хипа лежит первый объект. Мы переходим в таблицу виртуальных методов, проходим по указателю и по нему, среди прочих полей, лежит размер reference type. Я не очень понимаю, почему sizeof() работает только для value type, потому что чтобы сделать sizeof() reference type достаточно просто пройти по указателю и считать этот размер. Есть только одна разница - для массивов и для строк. Потому что для строк массивов надо этот sizeof посчитать, просто так не угадаешь. А для всех остальных обычных типов проблем никаких нет. 

И вот так, друг за другом, мы по всем объектам пробегаемся. Понимаем, какие из них промаркированы, поэтому легко можем понять, что является plug, занятым куском, что является gap, пропуском. Это вычисляется элементарно, так как на фазе маркировки мы там галочку поставили. Все друг за другом идущие пропуски мы маркируем  как едины пропуск - группу. Все друг за другом идущие занятые участки мы маркируем как один занятый участок. 

Размер и положение каждого пропуска могут быть сохранены. Дл каждого заполненного блока может быть сохранено его положение и финальное смещение. Это значит, что для каждого gap мы считаем его размер: напрмиер первый 32 байта. Для plug считаем offset, это то значение, которое было бы при запуске сжатия. Если бы мы сжимали, то offset - это то место, куда текущий plug переместился бы, насколько байт назад. Gap - это gap, тут понятно. И уже на этой стадии, если мы говорим о двух алгоритмах - sweep и collection (compacting?), можно увидеть всю информацию, которая нам нужна. В случае sweep нам нужны размеры пропусков, так как мы будем формировать списки свободных участков. Если мы делаем сжатие кучи, нам нужны только офсеты. На сладйде 07:10 это видно: вот он офсет -32, потом 32+64-96, следующий офсет -96+64-160. Так легко это все считается, поэтому все работает быстро. Чтобы это все произошло используется некий виртуальный внутренний аллокатор. Он итерирует по объектам и наращивает эту виртуальную дельту, сразу же делая виртуальный compaction этой кучи. 

Полученную информацию GC хранит в последних байтах предшествующего заполненной части gap. 

Был какой-то объект, на него исчезла последняя ссылка, прошел GC, информация в gap нам больше не нужна, значит мы его можем спокойно портить - писать туда то, что хотим, в том числе и результаты вычислений. Если у нас есть некий plug, то информацию мы запишем в предыдущий gap. 

На слайде 08:33 есть некие plugs и gaps и некий диапазон по центру. Мы его увеличили. Получилось, что там три процессорных слова. Дальше несколько слов - это plug. И последние три слова свободные. И перед plug, перед занятым участком, этот gap в три байта мы опять увеличили. Первый байт - это gapsize, второй - relocation offset. Третий разделен на две части, первую мы не рассматриваем, а вторая - это left и right offset, которые мы рассмотрим чуть позже. 

Размеры, которые мы считали, кладутся в gapsize. А relocation offset укладываются вторым байтом. Вся эта информация, которая размещена над куском памяти, вписана на места в нашем графике. 

Что же такое left и right? С помощью этих двух полей образуется двоичное дерево поиска. У нас есть plugs и gaps. Если рассматривать их как единую сущность, как на слайде 10:30, то  left/right offset - это offset до левой и правой ноды в двоичном дереве поиска нужных plugs. 

Двоичное дерево поиска строится для того, чтобы после фазы планирования, в ходе фазы сжатия нам все адреса поменять: куча сжимается, адреса объектов меняются. У всех объектов, которые ссылались на наши, надо заменить адреса в исходящих полях, локальных переменных, во всех рутах - везде, где были ссылки на группу, которая сжимается. Чтобы это сделать быстро и строится бинарное дерево поиска сбалансированное. 

Куча может быть огромная, поэтому используется структура, называемая Brick Table. Чтобы не искать по всему дереву, используется структура, схожая с карточным столом. Весь участок памяти приложения делится на большие регионы. Ячейка Brick Table - это ссылка на корень дерева двоичного поиска, которая описывает plugs и gaps внутри этого диапазона памяти. 

Если у нас на слайде 13:00  диапазон памяти от 1000 до 2000, вторая ячейка Brick Table отвечает именно за него. Она указывает на некий plug в центре, который является корнем двоичного дерева поиска других plugs. 

Если значение ноль, то в этом диапазоне нет никаких данных. Если значение положительное - то это адрес некоего корня. Если значение отрицательное, то этот диапазон является продолжением предыдущего диапазона. В процессе поиска нужного адреса мы делим на 0x1000, чтобы получить номер ячейки. Встаем на ячейку, дальше смотрим, что там за значение. И если ноль - то нет данных, если больше единицы - корень, меньше нуля - то там офсет на ту ячейку Brick Table, в которой записан корень. 

Pinned Objects.

Чем плохи запиненые объекты? 

Посомтрим на слайд 14:34. Тут есть кусок памяти, на нем - plugs, бледные участки, а яркий кусок - это запиненый объект. Что произойдет во время GC? Plug сожмется и все. А запиненый кусок останется на месте. Что произойдет, если слева от запиненого куска есть участок памяти, на который есть ссылка? 

Давайте вспомним, что же делает фаза планирования? Она вычисляет plugs и gaps. И в зоне, которая предшествует plug в gap планирование записывает размер gap и смещение. Вот здесь возникает неприятная ситуация. У нас два занятых участка. Первый - обычный plug, а второй - запиненый. Мы не можем рассматривать их как единый кусок. Потому что первый будет двигаться, а второй нет. Поэтому выделяют plugs и pinned plugs. 

В данном случае можно выкрутиться. Поскольку у нас все потоки стоят, мы можем испортить данные, а потом вернуть все назад. Память перед запиненым участком интерпретируется как кусок, в который можно что-то записать. Но перед тем, как этот участок портить, данные нужно куда-то сохранить, чтобы вернуть все обратно. 

Этот участок в 4 байта сохраняется в pinned plug queue. Это очередь на возврат. Мы сохраняем участок, чтобы потом его вернуть обратно. Участок называется saved_pre_plug. Туда сохраняется информация о типе информации и к чему она относится. После этого все сжимается и этот участок возвращается на свое место.То есть появляется дополнительное действие.

А что, если наоборот? Сначала идет запиненый объект, а потом обычный объект, на который есть ссылка. С запиненым объектом все хорошо, он может свои данные хранить в gap. Где же хранить данные обычному объекту? Перед ним запиненный. 

Почему пинят? Потому что буфер уходит в unmanaged память. Если GC, перед тем как начать работу, саспендит потоки - managed. То про обычные потоки он ничего не знает. Он не может их засапендить. А если буфер ушел в unmanaged память, то любой может с ним в параллели работать из unmanaged кода. Туда нельзя ничего записывать, в том числе и информаций plugs&gaps. В этом случае весь этот кусок помечается как pinned plug: и запиненый объект, и тот, который за ним следует. Они оба пинуются.  После GC они в такой же последовательности и остаются. 

Еще одна ситуация. Сначала у нас идет запиненый, потом два незапиненых объекта. Перед третьим объектом есть второй обычный объект. То есть, когда pinned plug объединяется для GC, в дальнейшем все равно воспринимается раздельно. Последний plug знает, что предшествующий объект не запинен и туда можно писать информацию о gaps и plugs. Но такой кусок тоже уходит в очередь pinned plug queue как saved_post_plug объект. 

И совсем нездоровая ситуация возникает, когда у нас идет несколько объектов подряд и один из них запиненый. Это одна из частых ситуаций.  В этом случае происходит совмещение обоих объектов. Для того, чтобы записать gaps и offsets, нам необходимо испортить и предыдущий объект, и следующий. А чтобы восстановить, мы должны эти куски уложить в очередь на восстановление. 

Demotion. 

Есть такие понятия, как Promotion и Demotion.

Promotion: GC дернули на нулевом поколении и после этого номер поколения вырос. Demotion работает наоборот. Когда у нас было первое поколение, мы сделали GC, и по какой-то причине оно стало нулевым. 

После GC запиненый объект остался в поколении ноль - это суть Demotion. 

Другая ситуация: есть поколение один и поколение ноль. На слайде 21:52 после GC слева осталось мало места, а под gen_0  - походит. И они оба перескочили на gen_0. Этому удивляться не стоит. Если такое произошло, надо просто искать, кто запинил объект и освобождать его.

Еще одна ситуация на слайде 22:56. Объект может свалится во второе поколение, если слева места мало и там нет смысла размещать что-то еще. С запинеными объектами может происходить все, что угодно. 

LOH.

Фаза планирования в LOH имеет смысл только для compacting, чтобы понять дальнейшие действия. В обычных сценариях сборки мусора у нас только sweep, а compacting нет. И фаза планирования в таких случаях отсутствует. Sweep не использует plug&gap, планирование ему не нужно. Он просто идет и составляет списки свободных участков. В этом плане sweep гораздо приятнее compacting. 

В обычном режиме LOH не осуществляет compacting, только по принуждению. Но в этом случае тоже без планирования. Потому что вы попросили конкретную и у него вся информация будет построена на месте, со всеми офсетами. Будут построены офсеты, plugs&gaps только для того, чтобы понять, может ли он обойтись обычным sweep. И если это возможно, то он его и сделает в хипе маленьких объектов. А в куче больших вычислять ничего не надо: что его попросят, то он и сделает. Поэтому фазы планирования там нет.

Так как LOH сделан исключительно для хранения больших данных, это позволяет упростить некоторые вопросы. Нет необходимости группировать объекты в plugs. В SOH это происходит исключительно ради производительности. Если у нас куча объектов по 32 байта, то нет смысла их рассматривать отдельно. С точки зрения GC их надо рассматривать как единое целое. Поэтому каждый объект в LOH - это plug. 

Плотность объектов в нем намного ниже, значит эффективнее трансляция адресов - эффективнее работает дерево. Легче перенести большой объект, чем plug, группу больших объектов. Чтобы обеспечить хранение plugs&gaps информации между объектами в LOH резервируется дополнительное место. Мы не можем себе позволить резервировать дополнительное место под информацию по размерам gap и relocation offset в куче маленьких объектов. Это будет слишком жирно. Поскольку у нас адреса всех объектов должны быть выровнены по размеру процессорного слова, по 8 или по 4 байтам, а информация по plugs и relocation offset занимает всего 3 байта, то мы не можем этими тремя байтами манипулировать. Мы либо 4 будем использовать, либо 8. Поэтому в LOH используем оптимизацию, храним эту информацию в gap. А в LOH мы можем себе позволить дополнительную информацию слева положить, не перетирая при пининге предыдущие объекты. 

Когда строится compacting в LOH, мы просто дополнительно резервируем между всеми объектами немного места для того, чтобы положить туда офсет. Когда мы запрашиваем compacting вручную, мы просчитываем эти офсеты и все сжимается. 

Почему в SOH вместо sweep может быть выбран compacting. 

Например, это последний GC перед OutOfMEmoryException. Последняя надежда, что получится найти немного памяти. 

Compacting может быть попрошен программистом намерено. Или было выбрано все место в эфемерном сегменте. Если у нас в эфемерном сегменте нулевого и первого поколения выбрано вес место и все закоммичено, то необходимо аллоцировать новый сегмент. Это дорого по времени. Поэтому первое, что пытается GC сделать, это сжать. 

Высокая степень фрагментации поколения так же может спровоцировать compacing. Это один из основных случаев. Если фрагментация слишком высокая и нет никакой возможности разместить allocation context в каких-то свободных промежутках памяти, то запускается сжатие. 

Еще один вариант - процесс занял слишком много памяти. Это, в целом, понятно. 

Рассмотрим фрагментацию. 

Это понятие достаточно виртуальное. Что значит "слишком фрагментировано"? Ответ на вопрос простой. Если взять некий total fragmentation, который собирается во время фазы планирования (это суммарное количество памяти, которое занимают gaps) и поделить на размер поколения, то мы получим отношение. При fragmentation size в 40 тысяч байт на нулевом поколении fragmentation ratio это 50%. Это триггер. Если фрагментация вырастает, то мы делаем сжатие кучи. 

Мы получили новую информацию, что если у нас есть фрагментация более 50% при определенном размере хипа, то запускается не очень приятная процедура сжатия, которого мы хотим избегать. 

Избегать процесса сжатия относительно просто. У нас часто возникает ситуация, когда мы создаем объекты одного типа, при этом создаем мы их не сразу друг за другом, а на всем протяжении жизни программы, зная о том, что эти объекты будут удалены вместе. Это приведет к фрагментации. Чтобы так не случилось, можно, например, сделать пул этих объектов. В нем объекты создаются друг за другом, группой. И когда будет потеряна ссылка на пул, они вместе уничтожаться, тем самым создав единый gap. Я в целом приходил к выводам, что пулы использовать очень хорошо. Они могут помочь во многих ситуациях. Спасают от фрагментации, от попадания на карточный стол (вы достаете объект из пула уже второго поколения). Пул - метод инициализации. Туда же можно отнести боксинг, который можно решить через эмуляцию боксинга или даже пул эмуляции боксинга. Если есть какой-то метод, который принимает object, вы изменить этого не можете и вынуждены боксить, то можно этого избежать. Фаза планирования на этом завершена.