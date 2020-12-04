# BytesRef

类名：`org.apache.lucene.util.BytesRef`

这个类是对byte数组简单的封装，提供了byte数组的比较功能，以及与String相互转化的功能。



# ByteBlockPool

包含一个数组，其中的每个元素是一个byte数组，即`byte[][]`，这个二维数组叫做一个pool，其中的每个byte数组叫做一个buffer（或者叫做block），然后包含几个状态变量：

- bufferUpto

  当前正在使用的buffer的在pool中的索引

- byteUpto

  指示向当前buffer写入数据的开始偏移量

- byteOffset

  当前buffer在pool中的字节偏移量

pool中的每个buffer的长度为32K，当一个buffer写满后通过`nextBuffer`创建下一个buffer。

写入数据的流程如下：

1. 首先`ByteBlockPool`中定义了两个数组：

   ```java
   int[] NEXT_LEVEL_ARRAY = {1, 2, 3, 4, 5, 6, 7, 8, 9, 9};
   int[] LEVEL_SIZE_ARRAY = {5, 14, 20, 30, 40, 40, 80, 80, 120, 200};
   ```

   这两个数组和Slice有关，接下来介绍slice的概念。

2. 数据是按slice来组织并存储在pool中的，首次写入数据时调用`int newSlice(final int size)`在pool中初始化一个slice，初始的size为`LEVEL_SIZE_ARRAY[0]`，每个slice的末尾会有一个非0字节，这个字节有两个作用，其一是表示slice的结束，其二是保存下一个slice的Level值在`NEXT_LEVEL_ARRAY`中的索引，首次创建的slice中该值为16，之所以用16是因为这个值只会使用后4位表示的数字，所有索引最大值为15，16就可以间接的表示0，又因为slice的末尾字节不能为0，所以不能直接写0。另外，每个slice只会在一个buffer中，而不会跨越两个buffer。

3. slice分配好后数据会逐个字节的写入到slice对应的区域中。这里slice只是buffer中一个特定区域的逻辑表示，存储数据的地方还是buffer的byte数组。

4. 如果slice写满了，则会在pool中分配一个新的slice，首次创建的slice的大小是固定的`LEVEL_SIZE_ARRAY[0]`，但之后的Slice的大小需要通过`NEXT_LEVEL_ARRAY`和`LEVEL_SIZE_ARRAY`确定，计算过程是首先获取上一个slice末尾字节中记录的值，然后获取这个字节的末4位表示的整数值，并用这个值作为`NEXT_LEVEL_ARRAY`的索引获取新slice的Level值，然后用这个Level值作为`LEVEL_SIZE_ARRAY`的索引获取新Slice的大小，得到大小后就在pool中为slice分配空间，同样的，新slice的最后一个字节会存储当前Level的值作为下一个slice的Level值的索引。最后，上一个slice的最后四个字节会用于存储新的slice在pool中的偏移量，这四个字节中除了最后一个，其它三个字节保存的数据会被移动到新slice的前三个字节中。然后新slice就可以继续写入数据了。

5. 通过1中的两个数组可以看出，slice的Level值最大为9，并且每个slice的大小最大为200。



之所以这么设计，是因为创建倒排索引时需要在一个pool中同时写入两种不同的数据，通过把数据分割成一系列slice，并以链表的方式把它们组织起来（每个slice的末尾会存储下一个slice的偏移量），可以让多种不同的数据共存于一个pool中。

