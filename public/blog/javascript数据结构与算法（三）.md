# 集合

集合是由一组无序且唯一的项组成的。

注意：ES6已经实现了标准Set类

```
class Set {
    constructor() {
        this.items = {}
    }
    // 判断是否存在某个值
    has(value) {
        return value in this.items
    } 
    // 添加
    add(value) {
        if (!this.has(value)) {
            this.items[value] = value
            return true
        }
        return false
    }
    // 删除
    remove(value) {
        if(this.has(value)) {
            delete this.items[value]
            return true
        }
        return false
    }
    // 清空
    clear() {
        this.items = {}
    }
    // 长度
    size() {
        return Object.keys(this.items).length
    }
    // 返回所有属性  以数组的形式
    values() {
        return Object.keys(this.items)
    }
}
```

# 字典

字典存储的是[key, value]，那么它和对象有什么区别？我认为他是对Object的进一步封装，提供了一些更加易用的接口。

注意：ES6实现了Map类，Map类的key可以是任何类型

```
class Dictionary{
    constructor() {
        this.items = {}
    }
    
    has(key) {
        return key in this.items
    }

    set(key, value) {
        this.items[key] = value
    }

    remove(key) {
        if (this.has(key)) {
            delete this.items[key]
            return true
        }
        return false
    }

    clear() {
        this.items = {}
    }

    get(key) {
        return this.has(key) ? this.items[key] : undefined 
    }

    keys() {
        return Object.keys(this.items)
    }

    values() {
        return Object.keys(this.items).reduce((prev, cur) => {
            prev.push(this.get(cur))
            return prev
        }, [])
    }

    size() {
        return this.keys().length
    }

    getItems() {
        return this.items
    }
}
```

# 散列表

```
class HashTable{
    constructor() {
        this.table = []
    }

    // 散列函数
    loseloseHashCode(key) {
        let hash = key.split('').reduce((prev, cur) => {
            return prev + cur.charCodeAt(cur)
        }, 0)

        return hash % 37
    }

    put(key, value) {
        let position = this.loseloseHashCode(key)
        this.table[position] = value
    }

    get(key) {
        return this.table[this.loseloseHashCode(key)]
    }

    remove(key) {
        this.table[this.loseloseHashCode(key)] = undefined
    }
}
```

这样定义的散列表有很明显的一个问题就是容易发生冲突，如果两个key得到的映射值是相同的，那么前面的就会被后面的覆盖。以下是两种解决方案：

- ## 分离链接

分离链接法是在散列表的每一个位置创建一个链表并将元素存储在里面。

```
class HashTable{
    constructor() {
        this.table = []
    }

    loseloseHashCode(key) {
        let hash = key.split('').reduce((prev, cur) => {
            return prev + cur.charCodeAt(cur)
        }, 0)

        return hash % 37
    }

    put(key, value) {
        let position = this.loseloseHashCode(key)
        if (this.table[position] === undefined) {
            this.table[position] = new LinkedList()
        }
        this.table[position].append({
            element: {
                key: key,
                value: value
            },
            next: null
        })
    }

    get(key) {
        let position = this.loseloseHashCode(key)
        if (this.table[position] !== undefined) {
            let current = this.table[position].head
            while (current) {
                if (current.element.key === key) {
                    return current.element.value
                }
                current = current.next
            }
        }
        return undefined
    }

    remove(key) {
        let position = this.loseloseHashCode(key)
        if (this.table[position] !== undefined) {
            let current = this.table[position].head
            while (current) {
                if (current.element.key === key) {
                    this.table[position].remove(current.element)
                    if (this.table[position].isImpty()) {
                        this.table[positon] = undefined
                    }
                    return true
                }
            }
        }
        return undefined
    }
}
```

- ## 线性探查

如果索引为index的位置已经被占据了，就尝试index+1的位置，类推。

```
class HashTable{
    constructor() {
        this.table = []
    }

    loseloseHashCode(key) {
        let hash = key.split('').reduce((prev, cur) => {
            return prev + cur.charCodeAt(cur)
        }, 0)

        return hash % 37
    }

    put(key, value){
        let position = this.loseloseHashCode(key)
        while (position) {
            if (this.table[position] === undefined) {
                this.table[position] = {
                    key: key,
                    value: value
                }
                return
            }
            position++
        }
    }

    get(key) {
        let position = this.loseloseHashCode(key)
        while (position) {
            if (this.table[position].key === key) {
                return this.table[position].value
            }
            position++
        }
    }

    remove(key) {
        let position = this.loseloseHashCode(key)
        while (position) {
            if (this.table[position].key === key) {
                this.table[position] = undefined
                return
            }
            position++
        }
    }
}
```