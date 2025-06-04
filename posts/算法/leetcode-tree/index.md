# Leetcode-Tree刷题记录




### 结构体定义

Leetcode中TreeNode结构体定义如下：

```C&#43;&#43;
  struct TreeNode {
      int val;
      TreeNode *left;
      TreeNode *right;
      TreeNode() : val(0), left(nullptr), right(nullptr) {}
      TreeNode(int x) : val(x), left(nullptr), right(nullptr) {}
      TreeNode(int x, TreeNode *left, TreeNode *right) : val(x), left(left), right(right) {}
  };
```





### 二叉树的前序遍历



递归版本：

```C&#43;&#43;
class Solution {
public:

    vector&lt;int&gt; res;
    vector&lt;int&gt; preorderTraversal(TreeNode* root) {
        if(!root) return res;
        res.push_back(root-&gt;val);
        preorderTraversal(root-&gt;left);
        preorderTraversal(root-&gt;right);
        return res;
    }
};
```



迭代版本：

```C&#43;&#43;
class Solution {
public:

    vector&lt;int&gt; preorderTraversal(TreeNode* root) {
        vector&lt;int&gt; res;
        stack&lt;TreeNode*&gt; stack;
        if(root){
            stack.push(root);
        }
        while(!stack.empty()){
            TreeNode* curr = stack.top();
            stack.pop();
            res.push_back(curr-&gt;val);
            if(curr-&gt;right != nullptr){
                stack.push(curr-&gt;right);
            }
            if(curr-&gt;left != nullptr){
                stack.push(curr-&gt;left);
            }
        }
        return res;
    }
};
```





### 二叉树的中序遍历



递归版本：

```c&#43;&#43;
class Solution {
public:
    vector&lt;int&gt; res;
    
    vector&lt;int&gt; inorderTraversal(TreeNode* root) {
        if(!root) return res;
        inorderTraversal(root-&gt;left);
        res.push_back(root-&gt;val);
        inorderTraversal(root-&gt;right);
        return res;
    }
};
```





迭代版本：

```C&#43;&#43;
class Solution {
public:
    vector&lt;int&gt; res;
    stack&lt;TreeNode*&gt; stack;
    vector&lt;int&gt; inorderTraversal(TreeNode* root) {
        TreeNode* curr = root;
        while(!stack.empty() || curr != NULL){
            //找到最左侧节点
            while(curr != NULL){
                stack.push(curr);
                curr = curr-&gt;left;
            }
            //此时栈顶为最左节点，curr为最左节点的左孩子节点
            TreeNode* top = stack.top();
            stack.pop();
            res.push_back(top-&gt;val);
            //判断top节点是否存在右子树
            if(top-&gt;right != NULL)
                curr = top-&gt;right;
        }
        return res;
    }
};
```





### 二叉树后序遍历



递归版本：

```C&#43;&#43;
class Solution {
public:
    vector&lt;int&gt; res;
    vector&lt;int&gt; postorderTraversal(TreeNode* root) {
        // 为空则直接返回
        if(root == NULL)
            return res;
        postorderTraversal(root-&gt;left);
        postorderTraversal(root-&gt;right);
        res.push_back(root-&gt;val);
        return res;
    }
};
```



递归版本：

```C&#43;&#43;
class Solution {
public:
    vector&lt;int&gt; postorderTraversal(TreeNode* root) {
        vector&lt;int&gt; ans;
        stack&lt;TreeNode*&gt; stk;
        if(root != NULL)
            stk.push(root);
        // curr存储当前退出栈的结点
        TreeNode* curr = root;
        while(!stk.empty())
        {
            TreeNode* top = stk.top();
            if(top-&gt;left != NULL &amp;&amp; top-&gt;left != curr &amp;&amp; top-&gt;right != curr)
                stk.push(top-&gt;left);
            else if(top-&gt;right != NULL &amp;&amp; top-&gt;right != curr)
                stk.push(top-&gt;right);
            // 当左右子树都处理过或者不存在情况下，说明此结点可以弹栈
            else
            {
                ans.push_back(top-&gt;val);
                stk.pop();
                curr = top;
            }
        }
        return ans;
    }

};
```





### 相同的树

```C&#43;&#43;
class Solution {
public:
    bool check(TreeNode *p, TreeNode *q) {
        if (!p &amp;&amp; !q) return true;
        if (!p || !q) return false;
        return p-&gt;val == q-&gt;val &amp;&amp; check(p-&gt;left, q-&gt;right) &amp;&amp; check(p-&gt;right, q-&gt;left);
    }

    bool isSymmetric(TreeNode* root) {
        return check(root, root);
    }
};
```



---

> Author:   
> URL: http://localhost:1313/posts/%E7%AE%97%E6%B3%95/leetcode-tree/  

