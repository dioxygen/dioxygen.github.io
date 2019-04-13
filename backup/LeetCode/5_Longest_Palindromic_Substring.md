---
layout: post
comment: true
title: 5.Longest Palindromic Substring
show: 0
---

<head>
    <script src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>
    <script type="text/x-mathjax-config">
        MathJax.Hub.Config({
            tex2jax: {
            skipTags: ['script', 'noscript', 'style', 'textarea', 'pre'],
            inlineMath: [['$','$']]
            }
        });
    </script>
</head>

>approach 1:暴力求解

直接尝试所有的回文序列可能在的起始和结束位置（可以从序列长度最长的情况尝试到长度最小的情况）
```c
char* longestPalindrome(char* s) {
    char pal[1001];
    int size=0,i=0,j;
    int t_size,t_i;
    while(*(s+i)!=0){
        size++;
        i++;
    }
    for(t_size=size;t_size>=1;t_size--){
        for(i=0;i+t_size-1<=size;i++){
            t_i=i;
            j=i+t_size-1;
            while(*(s+t_i)==*(s+j)){
                t_i++;
                j--;
                if(t_i>=j){
                   *(s+i+t_size)=0;
                    return (s+i);
                }
            }
        }
    }
    return s;
}
```
>approach 2: 中央扩展法

考虑到回文序列是以中央位置呈镜像对称，因为以选取一个中央元素然后试探两边的元素是否对称，这样一共有2n-1种情况（回文序列长度为奇数n种，回文序列长度为偶数n-1种），时间复杂度为*O*(n<sup>2</sup>)

>approach 3: 动态规划

递推公式：

$$
p(i,j)=\begin{cases} true,\quad S_{i}...S_{j}是回文序列\\\\false,\quad否则\end{cases}
$$

因此:

$$
p(i,j)=(p(i+1,j-1)\&\&S_{i}==S_{j})
$$

初始条件：

$$
p(i,i)=true
$$

$$
p(i,i+1)=(S_{i}==S_{j})
$$

```c
#define dp(r,c)   dp[(r)*size+c]//这里r没加括号因为优先级的关系出错
char* longestPalindrome(char* s) {
    int i,j,max=1,start=0;
    int size=strlen(s);
    char *dp=calloc(sizeof(char),(size*size));
    if(size==0)
        return s;
    // for(i=0;i<=size-1;i++){
    //     dp(i,i)=1;
    //     if(i<size-1&&s[i]==s[i+1]){
    //         dp(i,i+1)=1;
    //         if(flag==0){
    //             start=i;
    //             len=2;
    //             flag=1;
    //         }
    //     }
    // }
    // flag=0;
    // for(step=2;step<=size-1;step++){
    //     for(i=0;i+step<=size-1;i++){
    //         if(dp(i+1,i+step-1)==1&&s[i]==s[i+step]){
    //             dp(i,i+step)=1;
    //             if(flag==0){
    //                 start=i;
    //                 len=step+1;
    //                 flag=1;
    //             }
    //         }
    //     }
    //     flag=0;
    // }
    for(j=0;j<=size-1;j++){
        dp(j,j)=1;
        for(i=0;i<j;i++){
            if(s[i]==s[j]&&(j==i+1||dp(i+1,j-1)==1)){
                dp(i,j)=1;
                if(j-i+1>max){
                    max=j-i+1;
                    start=i;
                }
            }
        }
    }
    *(s+start+max)=0;
    return s+start;
}
```
（感觉动态规划无法结合贪心策略）
>approach 4：Manacher算法

>approach 5：后缀树
