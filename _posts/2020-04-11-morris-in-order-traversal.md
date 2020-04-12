---
layout:     post
title:      "Morris 遍历"
subtitle:   "二叉树中序遍历"
date:       2020-04-11
author:     "leasy"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - Algorithm
    - Tree
---

> "Morris 中序遍历二叉树"

这篇文章介绍一种二叉树的遍历方式，这种遍历方式可以做到以下要求：

- O(1)空间复杂度；
- 二叉树的形状不能被破坏(遍历的过程中可以有修改，遍历完成之后二叉树形状和初始相同)。

本文只介绍中序遍历的方式。

#### 定义树的数据结构

```java
public class TreeNode {
    int val;
    TreeNode left;
    TreeNode right;
    TreeNode(int val) {
        this.val = val;
    }
}
```

#### 二叉树常用的遍历方式

- 递归

递归的实现非常简单直观，不过递归的方式本身占用stack空间，递归的实现代码

```java
/**
* 时间复杂度O(n)
* 空间复杂度O(n)
*/
public void inOrderTraversal(TreeNode root) {
    inOrderTraversal(root.left);
    //Do something on root
    System.out.println(root.val);
    inOrderTraversal(root.left);
}
```

- 辅助栈

辅助栈不需要递归，不过需要建立一个队列来存放树节点

```java
/**
* 时间复杂度O(n)
* 空间复杂度O(n)
*/
public void inOrderTraversal(TreeNode root) {
    Stack<TreeNode> stk = new Stack<>();
    TreeNode cur = root;
    while (cur != null || !stk.isEmpty()) {
        if (cur != null) {
            stk.push(cur);
            cur = cur.left;
        } else if (!stk.isEmpty()) {
            System.out.println(stk.peek().val);
            cur = stk.peek().right;
            stk.pop();
        }
    }
}
```

#### Morris遍历

Morris 遍历方法可以做到这两点。要使用O(1)空间进行遍历，最大的难点在于，遍历到子节点的时候怎样重新返回到父节点，由于不能用栈作为辅助空间。为了解决这个问题，Morris方法用到了线索二叉树的概念。在Morris方法中不需要为每个节点额外分配指针指向其前驱和后继节点，只需要利用叶子节点中的左右空指针指向前驱节点或后继节点就可以了。

##### 步骤

- 1.如果当前节点的左孩子为空，则输出当前节点并将其右孩子作为当前节点。
- 2.如果当前节点的左孩子不为空，在当前节点的左子树中找到当前节点在中序遍历下的前驱节点。
  - 如果前驱节点的右孩子为空，将它的右孩子设置为当前节点。当前节点更新为当前节点的左孩子。
  - 如果前驱节点的右孩子为当前节点，将它的右孩子重新设为空（恢复树的形状）。输出当前节点。当前节点更新为当前节点的右孩子。
- 3.重复以上1、2直到当前节点为空。

![遍历过程图示]((https://leasyzhang.github.io/img/in-post/morris-traversal/morris.jpg))

代码

```java
/**
* 时间复杂度O(n): n个节点的二叉树有n-1条边，每条边最多走两次，一次是为了访问节点，一次是为了访问前驱节点
* 空间复杂度O(1):用了两个辅助变量
*/
public void morrisInOrder(TreeNode root) {
    TreeNode cur = root;
    while(cur != null) {
        if(cur.left != null) {
            TreeNode prev = cur.left;
            while(prev.right != null && prev.right != cur) {
                prev = prev.right;
            }

            if(prev.right == cur) {
                System.out.println(cur.val);
                cur = cur.right;
                prev.right = null;
            } else {
                prev.right = cur;
                cur = cur.left;
            }
        } else {
            System.out.println(cur.val);
            cur = cur.right;
        }
    }
}
```
