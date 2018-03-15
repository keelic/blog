# CopyOnWriteArrayList与Collections#synchronizedList如何保证线程安全

**作者：** keelic  
**日期：** 2018-03-15  
**标签：** 源码, jdk , CopyOnWriteArrayList,  Collections#synchronizedList

---

## 0. 概述
ArrayList和LinkedList的实现不是同步的。在一方对非同步list进行遍历，而另一方对其进行修改时，会抛出ConcurrentModificationExcpetion异常。  
但是，这还不是唯一的问题，另外一个问题是**多个线程对非同步list进行并发修改时，会导致数据丢失。**  

CopyOnWriteArrayList与Collections#synchronizedList是用来解决同步修改导致数据丢失的两种解决方案。各有优劣。

## 1. Collections#synchronizedList同步包装方法
