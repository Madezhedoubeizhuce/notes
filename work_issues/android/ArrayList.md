# Iterator ConcurrentModificationException

在使用ArrayList的iterator遍历数组元素时经常会发生`ConcurrentModificationException`，一般发生这个异常说明当前的数据在其他地方被改动了，下面分析一下ArrayList中是怎么检测的。

这是一段可能发生`ConcurrentModificationException`的代码：

```java
Iterator<HandleTask> iterator = handleTasks.iterator();
while (iterator.hasNext()) {
    HandleTask task = iterator.next();
    if (task.connectID.equals(handleTask.connectID) || task.socket == null || task.socket.isClosed()) {
        Log.d(TAG, "removeHandleTask: remove " + task.connectID + ", " + task);
        iterator.remove();
    }
}
```

我们先来看一下`iterator.next();`的实现：

```java
public E next() {
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
    int i = cursor;
    if (i >= limit)
        throw new NoSuchElementException();
    Object[] elementData = ArrayList.this.elementData;
    if (i >= elementData.length)
        throw new ConcurrentModificationException();
    cursor = i + 1;
    return (E) elementData[lastRet = i];
}
```

这里面一开始就校验modeCount和expectedModCount是否相等，如果不相等，就认为当前List的数据被修改过了，然后抛出异常。

其中expectedModCount在调用`iteator()`方法创建Iterator对象时被初始化为当前的modCount：

```java
 private class Itr implements Iterator<E> {
        // Android-changed: Add "limit" field to detect end of iteration.
        // The "limit" of this iterator. This is the size of the list at the time the
        // iterator was created. Adding & removing elements will invalidate the iteration
        // anyway (and cause next() to throw) so saving this value will guarantee that the
        // value of hasNext() remains stable and won't flap between true and false when elements
        // are added and removed from the list.
        protected int limit = ArrayList.this.size;

        int cursor;       // index of next element to return
        int lastRet = -1; // index of last element returned; -1 if no such
        int expectedModCount = modCount;
     
     ...
 }
```

modCount在ArrayList添加或删除元素时都会被修改，因此如果在使用迭代器遍历List元素时有其他线程修改了此List的元素，都会导致迭代器报错。