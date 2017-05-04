# 栈
栈是遵循后进先出原则的有序集合。新添加或者待删除的元素都保存在栈的末尾，称为栈顶，另一端就叫栈底。
## 创建

```
class Stack {
    constructor() {
        this.items = []
    }
    // 添加元素
    push(element) {
        this.items.push(element)
    }

    // 删除元素
    pop() {
        return this.items.pop()
    }

    // 返回栈顶
    peek() {
        let len = this.items.length
        return this.items[len - 1]
    }

    // 是否是空栈
    isEmpty() {
        return this.items.length === 0
    }

    // 返回栈里元素个数
    size() {
        return this.items.length
    }

    // 清空栈
    clear() {
        this.items = []
    }

    // 打印
    print() {
        console.log(this.items.toString())
    }
}
```

# 队列
栈是遵循先进先出原则的有序集合。
## 创建

```
class Queue {
    constructor() {
        this.items = []
    }
    // 添加元素
    enqueue(element) {
        this.items.push(element)
    }

    // 删除元素
    dequeue() {
        return this.items.shift()
    }

    front() {
        return this.items[0]
    }

    isEmpty() {
        return this.items.length === 0
    }

    // 返回栈里元素个数
    size() {
        return this.items.length
    }

    // 清空栈
    clear() {
        this.items = []
    }

    // 打印
    print() {
        console.log(this.items.toString())
    }
}
```