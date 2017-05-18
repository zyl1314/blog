# 排序算法

## 冒泡排序
比较两个相邻的项，如果第一个比第二个大，就交换它们

```
function bubbleSort(arr) {
    var len = arr.length
    for (var i = 0; i < len; i++) {
        for (var j = 0; j < len - (i + 1); j++) {
            if (arr[j] > arr[j+1]) {
                swap(arr, j, j+1)
            }
        }
    }    
    return arr
}

function swap(arr, index1, index2) {
    var temp = arr[index1]
    arr[index1] = arr[index2]
    arr[index2] = temp
}
```

## 选择排序
选择排序算法是一种原址比较排序算法。选择排序大致的思路是找到数据结构中的最小值并将其放置在第一位，接着找到第二小的值放在第二位，以此类推。

```
function selectionSort(arr) {
    var len = arr.length,
        flag
    for (var i = 0; i < len; i++) {
        flag = i 
        for (var j = i + 1; j < len; j++) {
            if (arr[j] < arr[flag]) {
                flag = j
            }
        }
        if (flag !== i) {
            swap(arr, flag, i)
        }
    }
    return arr
}
```

## 插入排序
插入排序每次排一个数组项，以此方式构建最后的排序数组。

```
function insertionSort(arr) {
    var len = arr.length,
        j,
        temp
    for (var i = 1; i < len; i++) {
        j = i 
        temp = arr[j]
        while (j > 0 && temp < arr[j - 1]) {
            arr[j] = arr[j - 1]
            j--
        }
        arr[j] = temp
    }
    return arr
}
```

## 归并排序
将原始数组划分为较小的数组，知道每个小数组只有一个位置，接着将小数组归并成较大的数组，知道最后只有一个排序完毕的大数组。

```
function mergeSort(arr) {
    if (arr.length <= 1) {
        return arr
    }
    var mid = Math.floor(arr.length / 2)
    var left = arr.slice(0, mid)
    var right = arr.slice(mid)

    return merge(mergeSort(left), mergeSort(right))
}

function merge(left, right) {
    var result = []
    while (left.length && right.length) {
        if (left[0] < right[0]) {
            result.push(left.shift())
        } else {
            result.push(right.shift())
        }
    }
    return left.length ? result.concat(left) : result.concat(right)
}
```

## 快速排序
（1）选择主元
（2）将小于主元的放置主元左侧，大于主元的放置主元右侧
（3）对划分后的小数组重复上述两步

```
function quickSort(arr) {
    if (arr.length <= 1) {
        return arr
    }
    var mid = Math.floor(arr.length / 2)
    var flag = arr.splice(mid, 1)
    var left = []
    var right = []
    for (var i = 0; i < arr.length; i++) {
        if (arr[i] <= flag) {
            left.push(arr[i])
        } else {
            right.push(arr[i])
        }
    }
    return quickSort(left).concat(flag, quickSort(right))
} 
```