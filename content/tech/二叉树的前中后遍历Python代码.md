---
title: "二叉树的前中后遍历Python代码"
date: 2021-09-25T21:13:45+09:00
slug: "python-tree-traversal"
dropCap: false
tags:
  - 算法
---

遍历二叉树有三种顺序，对于下面的二叉树：
```
      1
    /   \
   2     3
  / \   / \
 4   5 6   7
```
前序结果为 `[1, 2, 4, 5, 3, 6, 7]`   
中序结果为 `[4, 2, 5, 1, 6, 3, 7]`    
后序结果为 `[4, 5, 2, 6, 7, 3, 1]`

### 1. 前序遍历
#### 递归解法
```py
def preorder(root: TreeNode) -> List[int]:
  return [root.val] + preorder(root.left) + preorder(root.right) if root else []
```

#### 迭代解法
```py
def preorder(root: TreeNode) -> List[int]:
    res, stack = [], [(root, False)]
    while stack:
        node, visited = stack.pop()  # the last element
        if node:
            if visited:  
                res.append(node.val)
            else:  # preorder: root -> left -> right
                stack.append((node.right, False))
                stack.append((node.left, False))
                stack.append((node, True))
    return res
```

### 2. 中序遍历
#### 递归解法
```py
def inorder(root: TreeNode) -> List[int]:
  return  inorder(root.left) + [root.val] + inorder(root.right) if root else []
```
#### 迭代解法
```py
def inorder(root: TreeNode) -> List[int]:
    res, stack = [], [(root, False)]
    while stack:
        node, visited = stack.pop()  # the last element
        if node:
            if visited:  
                res.append(node.val)
            else:  # inorder: left -> root -> right
                stack.append((node.right, False))
                stack.append((node, True))
                stack.append((node.left, False))
    return res
```

### 3. 后序遍历
#### 递归解法
```py
def postorder(root: TreeNode) -> List[int]:
  return  postorder(root.left) + postorder(root.right) + [root.val] if root else []
```

#### 迭代解法
```py
def postorder(root: TreeNode) -> List[int]:
    res, stack = [], [(root, False)]
    while stack:
        node, visited = stack.pop()  # the last element
        if node:
            if visited:  
                res.append(node.val)
            else:  # postorder: left -> right -> root
                stack.append((node, True))
                stack.append((node.right, False))
                stack.append((node.left, False))
    return res
```

不难看出对于不同的遍历顺序，解法都是类似的，对于记不住的人还是很有帮助的。

<br />
<br />
<p style="text-align: center;">如果本文对您有帮助，欢迎打赏。</p>
<img src="/images/qr-wechat.png" alt="赞赏码" width="300"/>