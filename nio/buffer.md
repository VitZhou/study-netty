# Buffer(缓冲区)
全限定名:java.nio.Buffer
一个用于特定的原始数据类型的容器

> 缓冲区实际上是一个数组。通常它是一个字节数组(ByteBuffer),也可以使用其他种类的数组。但是一个缓冲区不仅仅是一个数组,缓冲区提供了对数据的结构化访问以及维护读写位置(limit)等信息. ---《netty 权威指南》

除了Boolean之外,java的基础类型都有一个与之对应的缓冲区:

- ByteBuffer: 字节缓冲区
- CharBuffer: 字符缓冲区
- ShortBuffer: 短整型缓冲区
- IntBuffer: 整形缓冲区
- LongBuffer: 长整型缓冲区
- FloatBuffer: 浮点型缓冲区
- DoubleBuffer: 双精度型缓冲区

## 缓冲区要点
1. 	缓冲器是特定原始类型的元素的线性有限序列。 除了其内容(content)，缓冲区的基本属性是容量(capacity)，限制(limit)和位置(position)：
	- 缓冲区的capacity是它包含的元素数量。 缓冲区的容量从不可为负数，不可改变。
	- 缓冲区的limit是不可读取或写入的第一个元素的索引。 缓冲区的限制永远不会是负数，永远不会超过其容(即:0<=limit<=capacity)
	- 缓冲区的position是要读取或写入的下一个元素的索引。 缓冲区的位置永远不会为负，并且不会超过其(即:0<=position<=capacity)

1. 	数据传输
	该类的每个子类定义了get()和put()操作的两个类别：

    - 相对: 从当前位置的读取或者写入一个或者多个元素,都会根据传输的元素个数改变其position，如果所请求的传输元素个数超出了limit(position >=limit)，那么相对的get操作会抛出BufferUnderflowException异常,put操作会抛出BufferOverflowException(length>(limit - position))；而且再其任一异常情况下数据都不会传输
    - 绝对: 使用显示的索引操作Buffer,不会改变其position.如果参数索引Index超出了limit，其get,put操作都会抛出IndexOutOfBoundsException异常

    ```java
    @Test
    public void testPut() {
        IntBuffer buffer = IntBuffer.allocate(array.length);
        buffer.put(array);
        //指定读取下标为1的元素
        int b = buffer.get(1);
        assertThat(b).isEqualTo(2);
        //buffer的容量为数组的大小
        assertThat(buffer.capacity()).isEqualTo(array.length);
    }

    @Test
    public void testPut_fail() {
        IntBuffer buffer = IntBuffer.allocate(array.length);
        //将当前位置迁移,使得put的数据大于limit
        buffer.position(2);
        //由于传入的数据大小length,大于当前容量(limit-position),所以抛出BufferOverflowException
        assertThatThrownBy(() -> buffer.put(array))
                .isInstanceOf(BufferOverflowException.class);
    }

    @Test
    public void testGet() {
        IntBuffer buffer = IntBuffer.allocate(array.length);
        buffer.put(array);

        //指定从哪个位置开始读取
        buffer.position(0);
        //指定下标,不会改变position
        int absolute = buffer.get(1);
        assertThat(absolute).isEqualTo(2);
        assertThat(buffer.position()).isEqualTo(0);

        int relative = buffer.get();
        assertThat(relative).isEqualTo(1);
        assertThat(buffer.position()).isEqualTo(1);
    }

    @Test
    public void testGet_fail() {
        IntBuffer buffer = IntBuffer.allocate(array.length);
        buffer.put(array);
        //如果不指定读取的位置,那么position默认是capacity,position必须小于limit,所以抛出BufferUnderflowException
        assertThatThrownBy(buffer::get)
                .isInstanceOf(BufferUnderflowException.class);
    }
	```
	当然,数据也可以通过相对于当前位置的I/O Channel传入或者传出缓冲区

1. 	标记与重置
	当调用reset()方法时，缓冲区的mark是其position将被重置的索引。 mark并不总是被定义，但是当它被定义时它不会是负数，并且永远不会大于position(0<=mark<=position)。 如果mark被定义，则当position或limit被调整为小于mark的值时，它会被丢弃。 如果未定义标记，则调用reset()方法会导致InvalidMarkException抛出。

	```java
    @Test
    public void testMark() {
        IntBuffer buffer = IntBuffer.allocate(array.length);
        buffer.put(array);
        //指定从哪个位置开始读取
        buffer.position(0);
        int i = buffer.get();
        Buffer mark = buffer.mark();
        //下一次读取的位置为1
        assertThat(mark.position()).isEqualTo(1);
        assertThat(mark.limit()).isEqualTo(10);
        assertThat(mark.capacity()).isEqualTo(10);
        //当前读取出来的元素
        assertThat(i).isEqualTo(1);
    }

    @Test
    public void testRest() {
        IntBuffer buffer = IntBuffer.allocate(array.length);
        buffer.put(array);
        //指定从哪个位置开始读取
        buffer.position(0);
        int i = buffer.get();
        Buffer mark = buffer.mark();
        //下一次读取的位置为1
        assertThat(mark.position()).isEqualTo(1);
        assertThat(mark.limit()).isEqualTo(10);
        assertThat(mark.capacity()).isEqualTo(10);
        //当前读取出来的元素
        assertThat(i).isEqualTo(1);

        buffer.get();
        buffer.get();
        //此时position为3
        assertThat(buffer.position()).isEqualTo(3);

        //reset使得position恢复到之前标记的位置(mark=1)
        Buffer reset = buffer.reset();
        assertThat(reset.position()).isEqualTo(1);
    }
    ```

1.  不可变关系
	position,mark,limit,capacity的不可变关系如下:

    0 <= mark <= position <= limit <= capacity

    新创建的缓冲区position为0,mark为-1(未定义的情况下为-1)。 初始limit可以为零，或者可以是取决于缓冲区的类型和构造方式的其他一些值。 新分配的缓冲区的每个元素被初始化为零。
1. 	清理，翻转和倒带(回退)
	除了访问position，limit和capacity值以及mark和reset的方法之外，缓冲区还定义了以下操作：

    - clear(): 使缓冲区准备新的channel读取或相对的put操作序列：它将capacity和position的limit设置为零。
    - flip(): 使缓冲区准备新的channel写入或相对的get操作序列：它将limit设置为当前position，然后将position设置为零。
    - rewind(): 使缓冲区准备重新读取缓冲区中的数据：它保持limit不变，并将位置position为零。

1. 	只读缓冲区
	每个缓冲区都是可读的，但并不是每个缓冲区都是可写的。 每个缓冲区类的突变方法被指定为可选操作，当在只读缓冲区上调用时，它将抛出一个ReadOnlyBufferException异常。 只读缓冲区不允许其内容被更改，但其标记，位置和限制值是可变的。 缓冲区是否为只读可以通过调用其isReadOnly方法来确定。

1. 	线程安全
	缓冲区不是线程安全的。 如果由多个线程使用缓冲区，则应通过适当的同步来控制对缓冲区的访问。

1. 支持链式调用

## 方法解析
1. 	array(): 返回缓冲区背后的数组(可选操作).
	该方法旨在允许将支撑这个缓冲区的数组更有效的返回到本地代码,其具体的子类会为该方法提供具体的强类型返回。
    对此缓冲区内容的修改将导致返回的数组的内容被修改，反之亦然。

	在调用此方法之前调用hasArray方法，以确保此缓冲区具有可访问的后备数组。
    - ReadOnlyBufferException: 只读缓冲区不支持返回数组.
    - UnsupportedOperationException: 此缓冲区不由数组支持

	```java
    public final boolean hasArray() {
        return (hb != null) && !isReadOnly;//缓冲区的数据不为空,并且不是只读
    }
    public final int[] array() {
        if (hb == null)
            throw new UnsupportedOperationException();
        if (isReadOnly)
            throw new ReadOnlyBufferException();
        return hb;
    }
    ```

    hb数组的定义和初始化:

    ```java
    //这些字段在这里被声明，而不是在Heap-X-Buffer中，以便减少访问这些值所需的虚拟方法的调用次数，这对编写小型缓冲区而言	尤为昂贵。
    final int[] hb;//对于堆缓冲区是非空的

    public static IntBuffer allocate(int capacity) {
        if (capacity < 0)
            throw new IllegalArgumentException();
        return new HeapIntBuffer(capacity, capacity);//new了一个大小为capacity的堆缓冲区.
    }

    HeapIntBuffer(int cap, int lim) {            // 包级私有
    	super(-1, 0, lim, cap, new int[cap], 0);
    }
    //父类构造方法:
    IntBuffer(int mark, int pos, int lim, int cap,   //包级私有
                 int[] hb, int offset)
    {
        super(mark, pos, lim, cap);//初始化Buffer的mark,position,limit,capacity属性
        this.hb = hb;//即new int[cap]
        this.offset = offset;
    }
    ```
    如上可知在堆缓冲区中hb的大小由allocate的参数指定

	例子:

    ```java
    @Test
    public void testArray(){
        IntBuffer buffer = IntBuffer.allocate(array.length);
        boolean b = buffer.put(this.array).hasArray();
        if (b){
            int[] arr = buffer.array();
            assertThat(array).isEqualTo(arr);
        }
        assertThat(b).isTrue();
    }
    ```

1.	arrayOffset(): 返回缓冲区的第一个元素在支撑它的数组中的偏移量.(可选操作)
	如果该缓冲区由数组支撑，则缓冲区位置p对应于数组索引为:p + arrayOffset()。

	在调用此方法之前调用hasArray方法，以确保此缓冲区具有可访问的后备数组。
    - ReadOnlyBufferException: 只读缓冲区不支持返回数组.
    - UnsupportedOperationException: 此缓冲区不由数组支持

    ```java
     @Test
    public void testArrayOffset(){
        IntBuffer buffer = IntBuffer.allocate(array.length);
        boolean b = buffer.put(this.array).hasArray();
        if (b){
            int i = buffer.arrayOffset();
            assertThat(i).isEqualTo(0);
        }
        assertThat(b).isTrue();
    }
    ```

1. 	capacity(): 返回缓冲区的容量
	```java
    @Test
    public void testCapacity(){
        IntBuffer buffer = IntBuffer.allocate(array.length);
        buffer.put(array);
        int capacity = buffer.capacity();
        //缓冲区的容量等于分配(IntBuffer.allocate)的大小
        assertThat(capacity).isEqualTo(array.length);
    }
    ```

1. 	position(): 返回缓冲区的当前位置
	```java
    @Test
    public void testPosition(){
        IntBuffer buffer = IntBuffer.allocate(array.length);
        //初始position为0
        assertThat(buffer.position()).isEqualTo(0);
        buffer.put(array);
        //put()方法会将position设置为array.length
        assertThat(buffer.position()).isEqualTo(10);
    }
    ```
    缓冲区的put方法会将position设置为参数的长度:
    ```java
        public IntBuffer put(int[] src, int offset, int length) {

        checkBounds(offset, length, src.length);
        if (length > remaining())//数组大小不能大于剩余容量(limit-position)
            throw new BufferOverflowException();
        System.arraycopy(src, offset, hb, ix(position()), length);
        position(position() + length);//将position设置为length
        return this;
    }
    ```

1. 	position(int newPosition): 设置这个缓冲区的位置。 如果mark被定义并且大于新位置，则它被丢弃。

	- newPosition:	新位置值 必须是非负数，不得大于当前limit
	- IllegalArgumentException:	如果newPosition的前提条件不成立

    ```java
    public final Buffer position(int newPosition) {
        if ((newPosition > limit) || (newPosition < 0))//必须是非负数，不得大于当前limit
            throw new IllegalArgumentException();
        position = newPosition;
        if (mark > position) mark = -1;//如果newPosition小于mark,则mark被丢弃
        return this;
    }
    ```

	```java
    @Test
    public void testPosition2(){
        IntBuffer buffer = IntBuffer.allocate(array.length);
        buffer.position(5);
        assertThat(buffer.position()).isEqualTo(5);
    }
    ```

1.	clear(): 清除这个缓冲区。 position设置为零，limit设置为capacity，丢弃mark。
	在调用此方法之前使用一系列channel-read或put操作来填充此缓冲区。 例如：
	buf.put(); //填充缓冲区
    buf.clear（）; //准备读取缓冲区
    in.read（BUF）; //读取数据
    这种方法实际上并不会清除缓冲区中的数据，但它被命名为clear的确是因为它最常用来做这种事。

    ```java
    public final Buffer clear() {
        position = 0;
        limit = capacity;
        mark = -1;
        return this;
    }
    ```

    可见缓冲区的clear方法只是修改了position,limit,mark这些属性，并未将hb数字清空。
    例子:

	```java
    @Test
    public void testClear() throws IOException {
        IntBuffer buffer = IntBuffer.allocate(array.length);
        buffer.put(array);

        int[] ints = {11, 12, 13, 14, 15};
        buffer.clear();
        buffer.put(ints);

        //缓冲区中数组的前半部分(下标0-4)被覆盖了.
        int i = buffer.get(ints.length - 1);
        assertThat(i).isEqualTo(ints[ints.length-1]);

        //缓存区数组的后半部分(下标5-9)还是原来的数据
        int i2 = buffer.get(ints.length);
        assertThat(i2).isEqualTo(array[ints.length]);
    }
    ```

    如上可知clear之后的put操作实际上是覆盖原来的数组,那么它是怎么做的呢？

	```java
     public IntBuffer put(int[] src, int offset, int length) {

        checkBounds(offset, length, src.length);
        if (length > remaining())//数组大小不能大于剩余容量(limit-position)
            throw new BufferOverflowException();
        System.arraycopy(src, offset, hb, ix(position()), length);//@1
        position(position() + length);
        return this;
    }
    ```

    由@1行代码可知,put操作是将新put进来的数组,复制到缓冲区的支撑数组hb中。

1.	flip(): 根据当前position翻转缓冲区
	将limit设置为当前的position, position重置为0,丢弃mark。当将数据从一个地方传输到另一个地址时，该方法通常与紧compact方法结合使用。

    ```java
    public final Buffer flip() {
        limit = position;
        position = 0;
        mark = -1;
        return this;
    }
    ```

    例子:

    ```java
    @Test
    public void testFlip() throws IOException {
        IntBuffer buffer = IntBuffer.allocate(array.length);
        buffer.put(array).position(0);
        buffer.get();
        buffer.flip();

        //翻转之后limit为1,所以无法put太大的数组
        assertThatThrownBy(() -> buffer.put(array))
                .isInstanceOf(BufferOverflowException.class);

        int[] ints = {11};
        buffer.put(ints);
    }
    ```

1. 	rewind()

    ```java
    public final Buffer rewind() {
        position = 0;
        mark = -1;
        return this;
    }
    ```

    例子:

	```java
	@Test
    public void testRewind(){
        IntBuffer buffer = IntBuffer.allocate(array.length);
        Buffer rewind = buffer.put(array).position(10).rewind();

        assertThat(rewind.position()).isEqualTo(0);
    }
    ```

1. hasRemaining(): 当前position是否到了限制Limit

	```java
    public final boolean hasRemaining() {
        return position < limit;
    }
    ```

    例子:

    ```java
    @Test
    public void testHasRemaining() throws IOException {
        IntBuffer buffer = IntBuffer.allocate(array.length);
        buffer.put(array).position(0);

        boolean b = buffer.hasRemaining();
        assertThat(b).isTrue();

        buffer.position(10);
        assertThat(buffer.hasRemaining()).isFalse();
    }
    ```

1. isDirect(): 该缓冲区是否是直接内存(堆外内存);
