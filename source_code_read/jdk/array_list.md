## ArrayList源码阅读(jdk8)

#### 作者 
keelic
#### 日期
2018-03-14
#### 标签
源码, jdk , ArrayList

---

## 0 简介
ArrayList是List接口的一种实现，其底层数据结构是一个Object[]数组。

## 1 重要属性
```java
/** ArrayList实际存储元素的结构 */
transient Object[] elementData;
/** list包含元素个数 */
private int size;  
/** 默认容量大小 */
private static final int DEFAULT_CAPACITY = 10;
/**  */
```
