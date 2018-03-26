# ConcurrentLinkedQueue原理分析

**作者：** keelic  
**日期：** 2018-03-26  
**标签：** 源码, jdk , CAS,  ConcurrentLinkedQueue

---

## 0. 概述
ConcurrentLinkedQueue是一个线程安全的队列。实现线程安全的队列有两种方式：`阻塞算法`和`非阻塞算法`。阻塞算法可以使用`加锁`的方式来实现，可以使用一个锁（出队/入队共同使用），也可以使用两个锁（出队/入队各自使用一个）。非阻塞算法可以使用循环`CAS`(Compare-And-Set)的方式。JDK8中ConcurrentLinkedQueue就是一个使用非阻塞算法实现的线程安全的队列。  

## 1. CAS原子操作
现代的多处理器系统大多提供了特殊的指令来管理对共享数据的并发访问，这些指令能实现原子化的`读-改-写`操作。现代典型的多处理器系统通常支持两种同步原语（机器级别的原子指令）：CAS 和 LL/SC。Intel，AMD 和 SPARC 的多处理器系统支持`比较并交换`（compare-and-swap，CAS）原子指令。  

CAS包含了`读取-比较-写入`三个步骤，且**原子性地完成这三个步骤**：  
1. 从内存中读取变量item的当前值，假设为itemVal；
2. 比较itemVal是否与期望值expectVal是否相等：
3. 如果itemVal != expectVal，则直接返回操作失败；如果itemVal == expectVal，则将item的变量设置为值val；  

这三个过程可以用如下java代码进行模拟：
```java
// 锁对象
public static final Object lock = new Object();

/**
 * item 要修改值的变量
 * expectVal 期望值
 * val 设定值
 */
public boolean simulateCAS(item, expectVal, val){
    synchronized(lock){
        // 1.读取共享变量item的当前值
        itemVal = getCurrentValueOfVariable(item);

        // 2.判断变量当前值是否与期望值相等
        if(itemVal != expectVal){
            // 2.1 如果不相等，则此次cas操作失败
            return false;
        }

        // 2.2 如果相等，则将共享变量的值设置为val
        setValueOfVariable(item, val);
        return true; 
    }
}
```
