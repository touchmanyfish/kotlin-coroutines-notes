# LockFreeLinkedListNode

## 结构

结构为双向链表，链表中元素类型为:

* ListClosed(LockFreeLinkedListNode)
* Node(LockFreeLinkedListNode)

## 定义

1.节点类型X,Y,node(3种都是LockFreeLinkedListNode的子类型)

* 在给定node的prev方向连续:

```kotlin
// Y在nodeprev方向上连续
// 至少1个Y
// ...X<->Y(1)<->..Y(n)<->node<-..

// Y在nodeprev方向上不连续
// ...X<->node<-..
```

* 在给定node的next方向连续

```kotlin
// Y在node的next方向上连续
// 至少1个Y
// ...->node<->Y(1)<->..Y(N)<->X..

// Y在node的next方向上不连续
// ...->node<->X..
```


当Node.prev指向一个非ListClosed的Node时，Node.prev指向Node的“last位置”

2.bitMask命中ListClose(forbiddenElementsBitmask)：
* bitMask and forbiddenElementsBitmask == 1 : true  命中
* bitMask and forbiddenElementsBitmask == 0 : false 未命中

## last位置

```kotlin
// LC:ListClosed

// ..->LC(1)<->..<->LC(n)<->Node..
```

当LC在Node.prev方向上连续时，LC(1).prev指向Node的“last位置”

```kotlin
// None_LC:非ListClosed的Node
// Node:LockFreeLinkedListNode

// ..->None_LC<->Node..
```
## close()/addLast()

```kotlin
public actual fun close(forbiddenElementsBit: Int) {
    // ..
}

public actual fun addLast(node: Node, permissionsBitmask: Int): Boolean {
    //..
}
```
双向链表中能包含的节点类型，两者都是LockFreeLinkedListNode:
1. Node
2. ListClose

Node类型可以通过addLast()/addOneIfEmpty()插入；ListClose具有private关键字，所以只能通过close()方法插入.

当调用node.addLast(node2, bitMask)时：  
* 如果node.prev方向上没有ListClosed连续，则将node2插入到node的"last位置"
* 如果node.prev方向上有ListClosed连续：
  1. bitMask命中其中一个ListClose，则跳过插入
  2. bitMask没有命中ListClose，则将则将node2插入到node的"last位置"

close()方法内部直接调用的addLast()方法对ListClose进行插入:
```kotlin
public actual fun close(forbiddenElementsBit: Int) {
    addLast(ListClosed(forbiddenElementsBit), forbiddenElementsBit)
}
```
```kotlin
val A_permiss = 1//001
val B_permiss = 2//010
val C_permiss = 4//010

val node = LockFreeLinkedListNode()
// @->node<-@

val node2 = LockFreeLinkedListNode()
val node3 = LockFreeLinkedListNode()
val node4 = LockFreeLinkedListNode()
val node5 = LockFreeLinkedListNode()

val addResult1 = node.addLast(node2, A_permiss)
//node.prev方向上没有ListClose连续，node2所以插入成功
// @->node2<->node<-@

val addResult2 = node.addLast(node3, A_permiss)
//node.prev方向上没有ListClose连续，node3所以插入成功
// @->node2<->node3<->node<-@

//通过close()方法插入ListClose
val closeResult1 = node.close(A_permiss)
// @->node2<->node3<->LC(A_permiss)<->node<-@

//通过close()方法插入ListClose
val closeResult2 = node.close(B_permiss)
// @->node2<->node3<->LC(B_permiss)<->LC(A_permiss)<->node<-@

//node.prev方向上有ListClose连续，命中，所以插入失败
val addResult4 = node.addLast(node4, A_permiss)
// @->node2<->node3<->LC(B_permiss)<->LC(A_permiss)<->node<-@

//node.prev方向上有ListClose连续，但是没命中，所以插入成功
val addResult3 = node.addLast(node4, C_permiss)
// @->node2<->node3<->node4<->LC(B_permiss)<->LC(A_permiss)<->node<-@

// remove掉node，
val removeResult = node.remove()
// node2.prev上开始有ListClose连续
// @->node2<->node3<->node4<->LC(B_permiss)<->LC(A_permiss)<-@

// 类似的
// node2.prev方向上有ListClose连续，命中，所以插入失败
val addResult5 = node2.addLast(node5, A_permiss)
// @->node2<->node3<->node4<->LC(B_permiss)<->LC(A_permiss)<-@

val addResult6 = node2.addLast(node5, C_permiss)
// node2.prev方向上有ListClose连续，但是没命中，所以插入成功
// @->node5<->node2<->node3<->node4<->LC(B_permiss)<->LC(A_permiss)<-@
```
要想node1.prev方向上有ListClose连续，除了调用node1.close()方法，还可以通过remove其他节点实现：
```kotlin
val A_permiss = 1//001
val B_permiss = 2//010
val C_permiss = 4//010

val node1 = LockFreeLinkedListNode()
val node2 = LockFreeLinkedListNode()

val result1 = node1.addLast(node2, A_permiss)
//@->node2<->node1<-@

node2.close(A_permiss)
node2.close(B_permiss)
//@->LC(B_permiss)<->LC(B_permiss)<->node2<->node1<-@

node2.remove()
//@->LC(B_permiss)<->LC(B_permiss)<->node1<-@
//此时，node1.prev上有LC连续了。效果和如下代码相同：
//val node1 = LockFreeLinkedListNode()
//node1.close(A_permiss)
//node2.close(B_permiss)

val result2 = node1.addLast(node2, A_permiss)
// 失败，因为A_permiss匹配

val result3 = node1.addLast(node2, B_permiss)
// 失败，因为B_permiss匹配

val result4 = node1.addLast(node2, C_permiss)
// 成功.因为没有匹配
```
命中时使用的bitMask和ListClosed.forbiddenElementsBitmask都可以包含不止1个值为1的bit位置:
```kotlin
val A_permiss = 1//001
val B_permiss = 2//010

val node = LockFreeLinkedListNode()
val node2 = LockFreeLinkedListNode()

node.close(A_permiss or B_permiss)
    
// 全部添加失败
val addResult1 = node.addLast(node2, A_permiss)
val addResult2 = node.addLast(node2, B_permiss)
val addResult3 = node.addLast(node2, A_permiss or B_permiss)
```

总的来说：
* node.addLast(node2,bitMask)用于将node2添加到node的“last位置”，是否能添加成功则取决于bitMask是否命中node.prev上的ListClosed。  
  
* 可以使用close()/removed()/addLast()来对某个node上的插入进行权限管理；刚开始node没有任何权限限制，所以调用node.addLast()总是成功。当node的.prev方向
上有ListClosed连续时，调用node.addLast()就会进行权限检测；这种连续可以通过调用close()/remove()方法形成。

## todo
* 其他方法如何实现



