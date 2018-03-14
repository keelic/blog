# ArrayList/LinkedList与iterator和ConcurrentModificationException之间的相爱相杀

**作者：** keelic  
**日期：** 2018-03-14  
**标签：** 源码, jdk , ConcurrentModificationException 

---

## 0. 概述
在ArrayList/LinkedList源码分析中曾提到过`线程不同步`、`fail-fast`和`ConcurrentModificationException异常`的问题。
线程不同步问题已经在相关部分分析清楚了。  

`fail-fast`与`ConcurrentModificationException异常`是与iterator迭代器相关的：  
如果iterator迭代器A一旦创建，就不允许通过任何`外界方法`对其所迭代的实例作结构性修改；  
否则，在迭代过程中会立即抛出ConcurrentModificationException异常；  
注意，上述所说的`外界方法`指迭代器实例A之外的任何方法，包括实例本身的方法以及其它迭代器的方法。

## 1. 回顾结构性改变
ArrayList/LinkedList共同的父类AbstractList中维护了一个公共属性`protected transient int modCount`。
它记录了由于调用list实例本身的方法而导致其底层数据结构发生改变的次数。  

ArrayList能够引发modCount改变的方法有：
```java
// 调用add/addAll/ensureCapacity方法时首先就是检查容量，直接导致modCount修改
private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    // overflow-conscious code
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}

// remove会引发
public E remove(int index) {
    rangeCheck(index);
    
    modCount++;
    E oldValue = elementData(index);

    int numMoved = size - index - 1;
    if (numMoved > 0)
       System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // clear to let GC do its work
    return oldValue;
}

// clear会引发
public void clear() {
    modCount++;
    
    // clear to let GC do its work
    for (int i = 0; i < size; i++)
        elementData[i] = null;
    
    size = 0;
}
```
LinkedList中凡是要新增或删除链表节点的操作都会导致modCount的修改。

## 2. 迭代器如何判断其所迭代的实例是否被外界方法作了结构性修改
首先看看生成迭代器的方法：
```java
// ArrayList的iterator()方法，每次调用会new一个新的实例
public Iterator<E> iterator() {
    return new Itr();
}

// 迭代器接口的一个实现类，其内有一个expectedModCount属性，
// 可以看作是保存了一份创建迭代器实例时modCount的一份快照
// 注意：Itr是ArrayList的一个内部类
private class Itr implements Iterator<E> {
    //其它属性...
    
    // 每次创建迭代器实例的时候都会保存一份当前modCount的快照
    // 如果是该迭代器实例外界的方法修改了list实例的modCount方法，
    // 那么该迭代器expectedModCount的值肯定是不会随之改变的
    int expectedModCount = modCount;

    Itr() {}
    
    //其它方法...
}
```

在看看迭代器的remove方法，很快就能明白迭代器判断外部发生结构性修改的原理。
```java
public void remove() {
    if (lastRet < 0)
        throw new IllegalStateException();
    
    // 检查迭代器实例expectedModCount的值是否等于其所迭代list实例的modCount值
    // 如果不等，则立马抛出ConcurrentModificationException异常
    checkForComodification();

    try {
        // 调用实例的remove方法对实例作了结构性修改
        // 该方法返回时expectedModCount不等与modCount
        ArrayList.this.remove(lastRet);
        cursor = lastRet;
        lastRet = -1;
        
        // 自己做的事，自己负责。刷新自己所保存的快照
        expectedModCount = modCount;
    } catch (IndexOutOfBoundsException ex) {
        throw new ConcurrentModificationException();
    }
}
```
从上面代码看，迭代器的remove方法也是调用`外界方法`对list实例作了结构性修改。这与之前说的不矛盾：  
虽然迭代器中使用了外部方法，且作了结构性修改，但是在修改完成后迭代器刷新了自己的快照，相当于本次修改被自己所承认。
而如果单纯是其它线程调用`外界方法`对list实例作了结构性修改，就会缺少一次刷新快照的操作，相当于这个修改不被自己所承认。

整个流程就是：检查 → 结构性修改 → 刷新expectedModCount来记录自己的修改  
只要是被`外界方法`作了结构性修改，那么`检查`阶段肯定就会出现expectedModCount不等与modCount的情况，
随之抛出ConcurrentModificationException异常；  

除了迭代器的remove方法会检查之外，其next()方法入口处也会作检查。  

LinkedList以及ListItr等其它迭代器不展开分析了，道理一样。  

## 3. 三段小代码加深理解
```java
    /**
     * 会引发ConcurrentModificationException
     */
    public static void outsideModify(){
        ArrayList<Integer> list = new ArrayList<Integer>(3);

        list.add(1);
        list.add(2);
        list.add(3);

        // 此时expectedModCount = modCount = 3
        Iterator<Integer> itr = list.iterator();
        while (itr.hasNext()){
            Integer next = itr.next();

            // 被外界方法对list实例作了结构性修改
            // list的remove方法会触发modCount++操作
            // 第一次迭代结束时，就出现了expectedModCount 不等于 modCount的情况
            list.remove(next);
        }
    }

    /**
     * 会引发ConcurrentModificationException
     */
    public static void outsideModify2(){
        ArrayList<Integer> list = new ArrayList<Integer>(3);

        list.add(1);
        list.add(2);
        list.add(3);

        // 此时expectedModCount = modCount = 3
        Iterator<Integer> itr = list.iterator();
        while (itr.hasNext()){
            Integer next = itr.next();

            // 引发一次不被承认的结构性修改
            list.ensureCapacity(20);
        }
    }

    /**
     * 不会有异常
     */
    public static void insideModify(){
        ArrayList<Integer> list = new ArrayList<Integer>(3);

        list.add(1);
        list.add(2);
        list.add(3);

        // 此时expectedModCount = modCount = 3
        Iterator<Integer> itr = list.iterator();
        while (itr.hasNext()){
            Integer next = itr.next();

            // 采用迭代器自己的方法对list实例作结构性修改
            // 修改完成后，remove方法会刷新一次expectedModCount的值
            // 表示自己承认本次修改
            itr.remove();
        }
    }
```