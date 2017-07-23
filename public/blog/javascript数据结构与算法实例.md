1.输入两个链表，分别代表两个数字的倒序排列如下： 342+465 = 807;
> Input: (2 -> 4 -> 3) + (5 -> 6 -> 4)   Output: 7 -> 0 -> 8
```
function Node(value) {
    this.value = value 
    this.next = null
}
// 链表类
class List {
    constructor() {
        this.head = null
    }

    append(value) {
        let node = new Node(value),
            current = this.head
        if (this.head) {
            while (current.next) {
                current = current.next
            }
            current.next = node
        } else {
            this.head = node
        }

        return this
    }
}

const l1 = new List()
l1.append(4)
const l2 = new List()
l2.append(6).append(6)

// 主逻辑
function add(l1, l2) {
    let l3 = new List(),
        current1 = l1.head, 
        current2 = l2.head,
        flag = 0
    
    // while里面的flag是为了兼容边界情况 
    // 如( 3 -> 5 ) + ( 4 -> 5 )
    while (current1 || current2 || flag) {
        let sum = 0 

        if (current1) {
            sum += current1.value
        }
        if (current2) {
            sum += current2.value
        }

        sum += flag
        
        if (sum < 10) {
            l3.append( sum % 10 )
            flag = 0
        } else {
            l3.append( sum % 10 )
            flag = 1
        }

        current1 = current1 ? current1.next : null
        current2 = current2 ? current2.next : null
    }

    return l3
}
```
2.给定数组和固定的数，返回数组中元素和为该数的下标
> 比如： nums = [2, 7, 11, 15], target = 9, 因为 nums[0] + nums[1] = 2 + 7 = 9, return [0, 1].
```
function findNum(arr, sum) {
    for (let i = 0; i < arr.length; i++) {
        for (let j = i + 1; j < arr.length; j++) {
            if ((arr[i] + arr[j]) === sum) {
                return [i, j]
            }
        }
    }
}
```
