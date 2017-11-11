---
layout: post
title: Binary Search Tree
date: 2017-11-07 20:00:00 +0800
tags:
- algorithm
- bst
---

在计算机科学中，[二叉搜索树](https://en.wikipedia.org/wiki/Binary_search_tree)（有时称为有序或排序的二叉树）是特定类型的容器：将元素（如数字、名称等）存储在内存中。它允许快速查找、添加和删除元素。

其定义如下：
1. 空树为二叉搜索树
2. 若它的左子树不为空，则左子树上所有结点的值都小于根结点的值
3. 若它的右子树不为空，则右子树上所有结点的值都大于根结点的值

特点：

1. 特定的插入顺序可能会导致树很不均衡，造成搜索效率的下降
2. 中序遍历的结果是一个排序的数组

```
满足BST的二叉树

                4
              /   \
            2       6
          /  \    /   \
        1     3  5     7

不满足BST的二叉树(7和9)

                10
              /   \
            6       12
          /  \    /   \
        3     8  9     15
      /  \
     2    7
```

判断一棵树是否为BST一个容易犯的错误是只考虑根结点值和左右儿子值的大小关系，实现一种错误的代码实现：

```
struct Node {
    int value;
    Node *left;
    Node *right;
};

bool checkBST(Node *root) {
    if (root == NULL) {
        return false;
    }
    if (root->left == NULL && root->right == NULL) {
        return true;
    }
    if (root->left == NULL) {
        return (root->value < root->right->value && checkBST(root->right));
    }
    if (root->right == NULL) {
        return (root->value > root->left->value && checkBST(root->left));
    }
    return root->value > root->left->value && root->value < root->right->value &&
           checkBST(root->left) && checkBST(root->right);
}
```

显然上述代码对于上面第二棵树的返回值为 true，而这确实错误的。正确的算法应该为：

```
bool checkBST(Node *root) {
    return checkBST(root, INT_MIN, INT_MAX);
}

bool checkBST(Node *node) {
    if (node == NULL) {
        return true;
    }

    if (node->value > max || node->value < min) {
        return false;
    }

    return checkBST(node->left, min, node->value) &&
           checkBST(node->right, node->value, max);
}
```

**20171111更新**

上面提到过，BST的中序遍历是排序的数组，因此可以利用这点来判断一棵树是否为BST，复杂度依然是O(n)。

```
bool checkBST(Node * root) {
    if (root == NULL) return true;

    vector<int> array;

    inorderTraversal(root, array);

    for (int i = 1; i < array.size(); i++) {
        if (array[i] <= array[i-1]) {
            return false;
        }
    }

    return true;
}

void inorderTraversal(Node * root, vector<int> &array) {
    if (root->left != NULL) {
        inorderTraversal(root->left, array);
    }

    array.push_back(root->data);

    if (root->right != NULL) {
        inorderTraversal(root->right, array);
    }
}
```
