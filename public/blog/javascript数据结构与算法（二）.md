# 链表
要存储多个元素，数组是最常用的数据结构。但是数组有一个缺点：数组的大小是固定的的，从数组的起点或中间插入或移除项的成本很高。

链表也可以存储有序的数据集合，但是不同于数组，链表中的元素在内存中并不是连续放置的。每个元素由一个存储元素本身的节点和一个指向下一个元素的引用组成。

链表的好处在于添加或者删除元素的时候不需要移动其他元素。

## 创建
注意：要想访问链表中间的一个元素，必须从起点（表头）开始迭代链表直至找到所需的元素，因此需要将表头存储。

```
class LinkedList {
    constructor() {
        // 链表头
        this.head = null
        // 链表长度
        this.length = 0
    }
}
```
- append

append：向链表尾部添加一个新的项

```
append(target) {
    if (this.head === null) {
        this.head = target
    } else {
        let current = this.head 
        while (current.next) {
            current = current.next
        }
        current.next = target
    }
    this.length++
}

// target（待添加的元素）结构如下
{
    elemnt: '',
    next: null
}
```

- removeAt

从链表的特定位置移除元素

```
removeAt(position) {
    // 检查是否越界
    if (position > -1 && position < this.length) {
        let current = this.head,
            previous,
            index = 0
        // 移除第一项
        if (position === 0) {
            this.head = current.next
        } else {
            while (index++ < position) {
                previous = current
                current = current.next
            }
            // 改变next指向  跳过待删除值
            previous.next = current.next
        }
        this.length--
        // 返回删除值
        return current.element
    } else {
        return null
    }
}
```

- insert

插入一个值

```
insert(position, target) {
    // 检查是否越界
    if (position > -1 && position < this.length) {
        let current = this.head,
            previous,
            index = 0
        // 在首项位置添加
        if (position === 0) {
            target.next = current
            this.head = target
        } else {
            while (index++ < position) {
                previous = current
                current = current.next
            }
            // 改变next指向 
            previous.next = target
            target.next = current
        }
        this.length++
        // 返回删除值
        return true
    } else {
        return false
    }
}
```

- indexOf

查找元素的位置，找不到返回-1

```
indexOf(element) {
    let current = this.head,
        index = 0
    while (current) {
        if (current.element === element) {
            return index
        }
        index++
        current = current.next
    }
    return -1
}
```

- remove 

从链表中删除一项

```
remove(element) {
    // 这里无需判断是否存在  因为removeAt已经做了边界约束
    let index = this.indexOf(element)
    return this.removeAt(index)
}
```

- 其他

```
isEmpty() {
    return this.length === 0
}

size() {
    return this.length
}

getHead() {
    return this.head
}
```