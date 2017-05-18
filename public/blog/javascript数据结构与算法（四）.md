# 二叉树
```
function BinarySearchTree() {
    var Node = function(key) {
        this.key = key
        this.left = null
        this.right = null
    }
    // 根节点
    var root = null

    // 插入一个键
    this.insert = function(key) {
        var newNode = new Node(key)
        if (root === null) {
            root = newNode
        } else {
            insertNode(root, newNode)
        }
    }
    
    function insertNode(node, newNode) {
        if (newNode.key < node.key) {
            node.left === null ? (node.left = newNode) : insertNode(node.left, newNode)
        } else {
            node.right === null ? (node.right = newNode) : insertNode(node.right, newNode)
        }
    }

    // 中序遍历
    this.inOrderTraverse = function(callback) {
        inOrderTraverseNode(root, callback)
    }

    function inOrderTraverseNode(node, callback) {
        if (node !== null) {
            inOrderTraverseNode(node.left, callback)
            callback(node.key)
            inOrderTraverseNode(node.right, callback)
        }
    }

    // 先序遍历
    this.preOrderTraverse = function(callback) {
        preOrderTraverseNode(root, callback)
    }

    function preOrderTraverseNode(node, callback) {
        if (node !==null) {
            callback(node.key)
            preOrderTraverseNode(node.left, callback)
            preOrderTraverseNode(node.right, callback)
        }
    }

    // 后续遍历
    this.postOrderTraverse = function(callback) {
        postOrderTraverseNode(root, callback)
    }

    function postOrderTraverseNode(node, callback) {
        if (node !==null) {
            preOrderTraverseNode(node.left, callback)
            preOrderTraverseNode(node.right, callback)
            callback(node.key)
        }
    }

    // 搜索最小值
    this.min = function() {
        return minNode(root)
    }

    function minNode(node) {
        if (node) {
            while (node.left !== null) {
                node = node.left
            }
            return node.key
        } 
        return null
    }

    // 搜索最大值
    this.max = function() {
        return maxNode(root)
    }

    function maxNode(node) {
        if (node) {
            while (node.right !== null) {
                node = node.right
            }
            return node.key
        }
        return null
    }

    // 搜索特定值
    this.search = function(key) {
        return searchNode(root, key)
    }

    function searchNode(node, key) {
        if (node === null) {
            return false
        }

        if (key < node.key) {
            return searchNode(node.left, key)
        } else if (key > node.key) {
            return searchNode(node.right, key)
        } else {
            return true
        }
    }

    // 移除节点
    this.remove = function(key) {
        root = removeNode(root, key)
    }

    function removeNode(node, key) {
        if (node === null) {
            return null
        }

        if (key < node.key) {
            node.left = removeNode(node.left, key)
            return node
        } else if (key > node.key) {
            node.right = removeNode(node.right, key)
            return node
        } else {
            if (node.left === null && node.right === null) {
                node = null 
                return node
            }
            if (node.left === null) {
                node = node.right
                return node
            } else if (node.right === null) {
                node = node.left 
                return node
            }
            var aux = findMinNode(node.right)
            node.key = aux.key
            node.right = removeNode(node.right, aux.key)
            return node
        }
    }
}
```