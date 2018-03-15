# ArrayList源码阅读(jdk8)

**作者：** keelic  
**日期：** 2018-03-14  
**标签：** 源码, jdk , ArrayList  

---

## 0. 概述
ArrayList是List接口的一种实现，其底层数据结构是一个Object[]数组。  
**特点：**  
1 get/set的随机访问很快，得益于数组随机访问的高效率；  
2 add/remove效率没有LinkedList高，极有可能需要动态扩容与数据移位；

## 1. 重要属性
```java
/** ArrayList实际存储元素的数组 */
transient Object[] elementData;
/** list包含元素个数 */
private int size;  
/** 默认容量大小 */
private static final int DEFAULT_CAPACITY = 10;

/** 该属性是ArrayList的父类AbstractList的属性，它标记着实例发生结构性改变的次数 */
protected transient int modCount = 0;
```

`modCount`属性在ArrayList的父类AbstractList中，它用于标记实例发生`结构性改变`的次数。但凡在方法（例如：remove/add）的实现中对modCount作了修改，那就说明这个方法的调用会发生结构性改变。它与`ConcurrentModificationException`异常的引发有直接的关系。

## 2. 三个构造函数
```java
/** 指定初始容量大小，new一个指定大小的数组，用于存放元素 */
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    }
}

/** 没有指定容量大小，则使用默认容量10，在第一次调用add添加元素时才会实际new一个大小为10的数组 */
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}

/** 传入一个集合时，调用其toArray()方法对elementData进行初始化 */
public ArrayList(Collection<? extends E> c) {
    elementData = c.toArray();
    if ((size = elementData.length) != 0) {
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        if (elementData.getClass() != Object[].class)
            elementData = Arrays.copyOf(elementData, size, Object[].class);
    } else {
        // replace with empty array.
        this.elementData = EMPTY_ELEMENTDATA;
    }
}
```

## 3. 动态扩容策略
每次调用add/addAll方法向list中添加`k个元素`时，首先要检查当前可用容量是否足够。如果不够`（即：elementData.length - size < k）`，则要先作动态扩容操作。
```java
public boolean add(E e) {
    // 保证容量足够，不够时进行动态扩容
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    // 可以看出，所传入的e元素允许为null
    elementData[size++] = e;
    return true;
}

/** ensureCapacityInternal方法如果判断需要扩容，则会调用一个私有方法grow来完成扩容操作 */
private void grow(int minCapacity) {
    int oldCapacity = elementData.length;
    // 尝试增加1/2容量
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    // 新增1/2以后如果不够用，尝试将扩容大小设置为所需容量的最小值
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    // 如果所需容量大于MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8时，则将扩展容量设置为Integer.MAX_VALUE
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // 新建用于存放元素的数组，将当前elementData中的元素拷贝到新数组中。拷贝完成后，旧数组将会被GC自动回收
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

`public boolean addAll(Collection<? extends E> c)`方法扩容流程与add方法一样：先调用`ensureCapacityInternal(size + len(c))`方法保证底层数组容量够用，然后完成数组拷贝。

除了以上说的add/addAll能引发扩容操作之外，ArrayList还提供了供外部调用的扩容方法`ensureCapacity(int minCapacity)`，允许我们自己灵活的实现扩容操作。其底层依然是调用`private void grow(int minCapacity)`方法来完成动态扩容。**注意：** `ensureCapacity`方法不是list接口定义的方法，使用时需要将先对象转换成ArrayList。在需要多次调用add/addAll方法（`尤其是for循环中调用`）的场景中可以有效避免多次数据拷贝，提高效率，当然前提是可以大概预测出所需容量大小。

## 4. 并发安全性
ArrayList的实现不是同步的(not synchronized)，多个线程同时访问同一个ArrayList实例时，只要有任何一个线程在对该实例作`结构性改变`，我们都**必须**在外部对该实例进行同步处理。

如果存在并发访问的情况，尤其是并发访问的同时存在`结构性改变`，我们可以在创建ArrayList实例的时候用Collections.synchronizedList方法将该实例包装起来。如下：  
```java
List list = Collections.synchronizedList(new ArrayList(...));
```

**结构性改变：** 包括任何新增、删除，以及改变底层数组elementData容量大小的操作。注意，仅仅是调用`set(int index, E element)`方法设置元素的值不属于`结构性改变`范畴。

判断某个操作是否为`结构性改变`的终极方法：是否会修改modCount属性值。如下：
```java
public E remove(int index) {
    rangeCheck(index);
    // 改变modCount属性值，说明remove方法会导致结构性改变
    modCount++;
    E oldValue = elementData(index);

    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // clear to let GC do its work

    return oldValue;
}
```
