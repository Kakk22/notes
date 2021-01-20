# HashMap 总结

## 数据结构

1.7为 数组+链表

1.8为 **数组+链表+红黑树**

## 基本属性

```java
//默认初始容量为16
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; 
//默认负载因子为0.75
static final float DEFAULT_LOAD_FACTOR = 0.75f;
//Hash数组(在resize()中初始化)
transient Node<K,V>[] table;
//元素个数
transient int size;
//容量阈值(元素个数大于等于该值时会自动扩容)  
int threshold;
```

table数组里面存放的是Node对象，Node是HashMap的一个内部类，用来表示一个key-value，源码如下：

```java
static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

           ......

        public final V getValue() {
            return value;
        }
    
        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }
     ......
}
```

总结

- 默认初始容量为16，默认负载因子为0.75
- threshold = 数组长度 * loadFactor，当元素个数大于等于threshold(容量阈值)时，HashMap会进行扩容操作
- table数组中存放指向链表的引用

这里需要注意的一点是**table数组并不是在构造方法里面初始化的，它是在resize(扩容)方法里进行初始化的**。

这里说句题外话：可能有刁钻的面试官会问**为什么默认初始容量要设置为16？为什么负载因子要设置为0.75？**

我们都知道HashMap数组长度被设计成2的幂次方(下面会讲)，那为什么初始容量不设计成4、8或者32....其实这是JDK设计者经过权衡之后得出的一个比较合理的数字，，如果默认容量是8的话，当添加到第6个元素的时候就会触发扩容操作，扩容操作是非常消耗CPU的，32的话如果只添加少量元素则会浪费内存，因此设计成16是比较合适的，负载因子也是同理。

## table数组长度永远为2的幂次方

众所周知，HashMap数组长度永远为2的幂次方(指的是table数组的大小)，那你有想过为什么吗？

首先我们需要知道HashMap是通过一个名为tableSizeFor的方法来确保HashMap数组长度永远为2的幂次方的，源码如下：

```java
/*找到大于或等于 cap 的最小2的幂，用来做容量阈值*/
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

**tableSizeFor的功能（不考虑大于最大容量的情况）是返回大于等于输入参数且最近的2的整数次幂的数**。比如10，则返回16。

该算法让最高位的1后面的位全变为1。最后再让结果n+1，即得到了2的整数次幂的值了。

让cap-1再赋值给n的目的是另找到的目标值大于或等于原值。例如二进制1000，十进制数值为8。如果不对它减1而直接操作，将得到答案10000，即16。显然不是结果。减1后二进制为111，再进行操作则会得到原来的数值1000，即8。通过一系列位运算大大提高效率。

**那在什么地方会用到tableSizeFor方法呢？**

答案就是在构造方法里面调用该方法来设置threshold，也就是容量阈值。

**这里你可能又会有一个疑问：为什么要设置为threshold呢？**

因为在扩容方法里第一次初始化table数组时会将threshold设置数组的长度，后续在讲扩容方法时再介绍。

```java
/*传入初始容量和负载因子*/
public HashMap(int initialCapacity, float loadFactor) {
 
 if (initialCapacity < 0)
 throw new IllegalArgumentException("Illegal initial capacity: " +initialCapacity);
 if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
 if (loadFactor <= 0 || Float.isNaN(loadFactor))
 throw new IllegalArgumentException("Illegal load factor: " +loadFactor);
 
 this.loadFactor = loadFactor;
 this.threshold = tableSizeFor(initialCapacity);
}
```

## 那么为什么要把数组长度设计为2的幂次方呢？

我个人觉得这样设计有以下几个好处：

1、当数组长度为2的幂次方时，可以使用位运算来计算元素在数组中的下标

HashMap是通过`index = hash & ( table.length - 1 )`这条公式来计算元素在table数组中存放的下标，就是把元素的hash值和数组长度减1的值做一个与运算，即可求出该元素在数组中的下标，这条公式其实等价于hash%length，也就是对数组长度求模取余，只不过**只有当数组长度为2的幂次方时，hash&(length-1)才等价于hash%length**，使用位运算可以提高效率。

2、 增加hash值的随机性，减少hash冲突

如果 length 为 2 的幂次方，则 length-1 转化为二进制必定是 11111……的形式，这样的话可以使所有位置都能和元素hash值做与运算，如果是如果 length 不是2的次幂，比如length为15，则length-1为14，对应的二进制为1110，在和hash 做与运算时，最后一位永远都为0 ，浪费空间。

## HashMap的工作原理

`HashMap`底层是hash数组和单向链表实现,数组中的每个元素都是链表,由Node内部类(实现`Map.Entry<K,V>`接口)实现,`HashMap`通过`put&get`方法存储和获取。

**存储对象时,将`K/V`键值传给`put()`方法:**

- ①、调用hash(k)方法计算K的hash值,然后通过`index = hash & ( table.length - 1 )`这条公式得到数组下标

- ②、调整数组大小(当容器中的元素个数大于capacity*loadfactor时,容器会进行扩容resize为2n);

- ③

- - i.如果K的`hash`值在`HashMap`中不存在,则执行插入,若存在,则发生碰撞;
  - ii.如果K的`hash`值在`HashMap`中存在,且它们两者`equals`返回`true`,则更新键值对;
  - iii.如果K的`hash`值在`HashMap`中存在,且它们两者`equals`返回false,则插入链表的尾部(尾插法)或者红黑树中(树的添加方式)。
- ![微信图片_20201012191330](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20201012191330.jpg)



## 你知道hash的实现吗?为什么要这样实现?

`JDK1.8`中,是通过`hashCode()`的高16位异或低16位实现的:`( h = k.hashCode() ) ^ ( h>>> 16 )`,主要是从速度,功效和质量来考虑的,减少系统的开销,也不会造成因为高位没有参与下标的计算,从而引起的碰撞。

## 为什么要用异或运算符?

保证了对象的`hashCode`的32位值只要有一位发生改变,整个`hash()`返回值就会改变。尽可能的减少碰撞。

## 查找

在看源码之前先来简单梳理一下查找流程：

1. 首先通过自定义的hash方法计算出key的hash值，求出在数组中的位置
2. 判断该位置上是否有节点，若没有则返回null，代表查询不到指定的元素
3. 若有则判断该节点是不是要查找的元素，若是则返回该节点
4. 若不是则判断节点的类型，如果是红黑树的话，则调用红黑树的方法去查找元素
5. 如果是链表类型，则遍历链表调用equals方法去查找元素

HashMap的查找是非常快的，要查找一个元素首先得知道key的hash值，在HashMap中并不是直接通过key的hashcode方法获取哈希值，而是通过内部自定义的hash方法计算哈希值

`( h = k.hashCode() ) ^ ( h>>> 16 )`

知道如何计算hash值后我们来看看get方法

```java
public V get(Object key) {
    Node<K,V> e;
 return (e = getNode(hash(key), key)) == null ? null : e.value;//hash(key)不等于key.hashCode
}

final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; //指向hash数组
    Node<K,V> first, e; //first指向hash数组链接的第一个节点，e指向下一个节点
    int n;//hash数组长度
    K k;
 /*(n - 1) & hash ————>根据hash值计算出在数组中的索引index（相当于对数组长度取模，这里用位运算进行了优化）*/
 if ((tab = table) != null && (n = tab.length) > 0 && (first = tab[(n - 1) & hash]) != null) {
 //基本类型用==比较，其它用equals比较
 if (first.hash == hash && ((k = first.key) == key || (key != null && key.equals(k))))
 return first;
 if ((e = first.next) != null) {
 //如果first是TreeNode类型，则调用红黑树查找方法
 if (first instanceof TreeNode)
 return ((TreeNode<K,V>)first).getTreeNode(hash, key);
 do {//向后遍历
 if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
 return e;
            } while ((e = e.next) != null);
        }
    }
 return null;
}
```

## 插入

我们先来看看插入元素的步骤：

1. 当table数组为空时，通过扩容的方式初始化table
2. 通过计算键的hash值求出下标后，若该位置上没有元素(没有发生hash冲突)，则新建Node节点插入
3. 若发生了hash冲突，遍历链表查找要插入的key是否已经存在，存在的话根据条件判断是否用新值替换旧值
4. 如果不存在，则将元素插入链表尾部，并根据链表长度决定是否将链表转为红黑树
5. 判断键值对数量是否大于等于阈值，如果是的话则进行扩容操作

先看完上面的流程，再来看源码会简单很多，源码如下：

```java
public V put(K key, V value) {
 return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,boolean evict) {
    Node<K,V>[] tab;//指向hash数组
    Node<K,V> p;//初始化为table中第一个节点
 int n, i;//n为数组长度，i为索引
 
 //tab被延迟到插入新数据时再进行初始化
 if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
 //如果数组中不包含Node引用，则新建Node节点存入数组中即可    
 if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);//new Node<>(hash, key, value, next)
 else {
        Node<K,V> e; //如果要插入的key-value已存在，用e指向该节点
        K k;
 //如果第一个节点就是要插入的key-value，则让e指向第一个节点（p在这里指向第一个节点）
 if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
 //如果p是TreeNode类型，则调用红黑树的插入操作（注意：TreeNode是Node的子类）
 else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
 else {
 //对链表进行遍历，并用binCount统计链表长度
 for (int binCount = 0; ; ++binCount) {
 //如果链表中不包含要插入的key-value，则将其插入到链表尾部
 if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
 //如果链表长度大于或等于树化阈值，则进行树化操作
 if (binCount >= TREEIFY_THRESHOLD - 1)
                        treeifyBin(tab, hash);
 break;
                }
 //如果要插入的key-value已存在则终止遍历，否则向后遍历
 if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
 break;
                p = e;
            }
        }
 //如果e不为null说明要插入的key-value已存在
 if (e != null) {
            V oldValue = e.value;
 //根据传入的onlyIfAbsent判断是否要更新旧值
 if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
 return oldValue;
        }
    }
    ++modCount;
 //键值对数量大于等于阈值时，则进行扩容
 if (++size > threshold)
        resize();
    afterNodeInsertion(evict);//也是空函数？回调？不知道干嘛的
 return null;
}
```

从源码也可以看出**table数组是在第一次调用put方法后才进行初始化的**。
这里还有一个知识点就是在**JDK1.8版本HashMap是在链表尾部插入元素的，而在1.7版本里是插入链表头部的**，1.7版本这么设计的原因可能是作者认为新插入的元素使用到的频率会比较高，插入头部的话可以减少遍历次数。

## 链表树化

指的就是把链表转换成红黑树，树化需要满足以下两个条件：

- **链表长度大于等于8**
- **table数组长度大于等于64**

为什么table数组容量大于等于64才树化？

因为当table数组容量比较小时，键值对节点 hash 的碰撞率可能会比较高，进而导致链表长度较长。这个时候应该优先扩容，而不是立马树化

## 数组扩容的过程?

创建一个新的数组,其容量为旧数组的两倍,并重新计算旧数组中结点的存储位置。结点在新数组中的位置只有两种,原下标位置或原下标+旧数组的大小。

## 拉链法导致的链表过深问题为什么不用二叉查找树代替,而选择红黑树?为什么不一直使用红黑树?

之所以选择红黑树是为了解决二叉查找树的缺陷,二叉查找树在特殊情况下会变成一条线性结构(这就跟原来使用链表结构一样了,造成很深的问题),遍历查找会非常慢。而红黑树在插入新数据后可能需要通过左旋,右旋、变色这些操作来保持平衡,引入红黑树就是为了查找数据快,解决链表查询深度的问题,我们知道红黑树属于平衡二叉树,但是为了保持"平衡"是需要付出代价的,但是该代价所损耗的资源要比遍历线性链表要少,所以当长度大于8的时候,会使用红黑树,如果链表长度很短的话,根本不需要引入红黑树,引入反而会慢。

## 说说你对红黑树的见解?

- 1、每个节点非红即黑
- 2、根节点总是黑色的
- 3、如果节点是红色的,则它的子节点必须是黑色的(反之不一定)
- 4、每个叶子节点都是黑色的空节点(NIL节点)
- 5、从根节点到叶节点或空子节点的每条路径,必须包含相同数目的黑色节点(即相同的黑色高度)

## **为什么HashMap的key需要重写equals()和hashcode()方法？**

简单看个例子，这里以Person为例：

```java
public class Person {
    Integer id;
    String name;
 
 public Person(Integer id, String name) {
 this.id = id;
 this.name = name;
    }

 @Override
 public boolean equals(Object obj) {
 if (obj == null) return false;
 if (obj == this) return true;
 if (obj instanceof Person) {
            Person person = (Person) obj;
 if (this.id == person.id)
 return true;
        }
 return false;
    }

 public static void main(String[] args) {
        Person p1 = new Person(1, "aaa");
        Person p2 = new Person(1, "bbb");
        HashMap<Person, String> map = new HashMap<>();
        map.put(p1, "这是p1");
        System.out.println(map.get(p2));
    }
}
```

- 原生的equals方法是使用==来比较对象的
- 原生的hashCode值是根据内存地址换算出来的一个值

Person类重写equals方法来根据id判断是否相等，当没有重写hashcode方法时，插入p1后便无法用p2取出元素，这是因为p1和p2的哈希值不相等。

HashMap插入元素时是根据元素的哈希值来确定存放在数组中的位置，因此HashMap的key需要重写equals和hashcode方法。