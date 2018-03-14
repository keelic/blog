# LinkedList源码阅读(jdk8)

**作者：** keelic  
**日期：** 2018-03-14  
**标签：** 源码, jdk , LinkedList 

---

## 0. 概述
LinkedList是List接口的另外一种实现。其底层结构是一个双向链表。相对于ArrayList它有如下特点：  
1 `add/remove`执行效率高，没有动态容量扩展及数据移动过程；  
2 `get/set`随机访问效率低，完全依靠遍历链表来实现；  
3 实现了`Deque<E> `接口，具有队列和栈相关的api；  
4 所需存储空间更大，需要维护链表指针

## 1. 重要属性
```java
// 第一个节点指针
transient Node<E> first;

// 最后一个节点指针
transient Node<E> last;
```
维护链表首位指针便于高效执行`addLast/addFirst/removeFirst/removeLast`方法。同时，便于实现队列和栈相关的`push/pop`方法。

## 2. 链表节点定义
```java
    private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
```
 
## 3. 并发安全性
与ArrayList一样，也存在`并发结构性修改不安全`的特征。其结构性修改包括`add/delete`方法，同样可以采用如下包装的方法来规避此问题：  
```java
List list = Collections.synchronizedList(new LinkedList());
```
除此之外，也存在`iterator`遍历过程中也存在`fail-fast`问题及`ConcurrentModificationException`异常问题。  

并发安全性详细内容可以参考`ArrayList源码阅读`相关内容。`fail-fast`和`ConcurrentModificationException`可以参考iterator相关内容。