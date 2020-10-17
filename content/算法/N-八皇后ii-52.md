---
title: "N 八皇后ii 52"
date: 2020-10-18T00:34:46+08:00
draft: false
---


52. N皇后 II


n 皇后问题研究的是如何将 n 个皇后放置在 n×n 的棋盘上，并且使皇后彼此之间不能相互攻击。
![](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/10/12/8-queens.png
)
上图为 8 皇后问题的一种解法。给定一个整数 n，返回 n 皇后不同的解决方案的数量。
示例:


输入: 4
输出: 2
解释: 4 皇后问题存在如下两个不同的解法。
[
 [".Q..",  // 解法 1
  "...Q",
  "Q...",
  "..Q."],

 ["..Q.",  // 解法 2
  "Q...",
  "...Q",
  ".Q.."]
]

提示：

皇后，是国际象棋中的棋子，意味着国王的妻子。皇后只做一件事，那就是“吃子”。当她遇见可以吃的棋子时，就迅速冲上去吃掉棋子。当然，她横、竖、斜都可走一或 N-1 步，可进可退。（引用自 百度百科 - 皇后 ）



### 题解

#### 回溯法

```java

class Solution {
    int ans = 0;
    boolean[] pie, na, colums;

    public int totalNQueens(int n) {
           pie = new boolean[2 * n];
           na = new  boolean[2 * n];
           colums = new boolean[n];

           backtracking(0, n);
           return ans;
    }

    void backtracking(int row, int size){
        if(row == size){
            ans++;
            return;
        }
        // process current level
        for(int col = 0; col < size; col++){

            // 前向检查可以放
            int checkPie = col + row;
            int checkNa =  col - row + size;
            if(colums[col] || pie[checkPie] || na[checkNa]) continue;
            // 则放
            colums[col] = pie[checkPie] = na[checkNa] = true;
            // process next level
            backtracking(row + 1, size);
            // restore status
            colums[col] = pie[checkPie] = na[checkNa] = false;
        }

    }

}


```

### 二进制解法

* [ ] todo:


### 相关链接

[leetcode](https://leetcode-cn.com/problems/n-queens-ii/)
[leetcode discurss](https://leetcode.com/problems/n-queens-ii/discuss/20048/Easiest-Java-Solution-(1ms-98.22))



