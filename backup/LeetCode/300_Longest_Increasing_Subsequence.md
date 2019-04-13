---
layout: post
comments: true
title: 300.Longest Increasing Subsequence解题记录
show: 0
---

>approach 1：动态规划

O(_n<sup>2</sup>_)时间复杂度
```c
int lengthOfLIS(int* nums, int numsSize) {
    int i,j;
    int *dp=malloc(sizeof(int)*numsSize);
    int res=0;
    for(i=0;i<numsSize;i++){
        dp[i]=1;
    }
    for(i=0;i<numsSize;i++){
        for(j=0;j<i;j++){
            if(nums[i]>nums[j]&&dp[j]+1>dp[i])
                dp[i]=dp[j]+1;
        }
        if(dp[i]>res)
            res=dp[i];
    }
    return res;
}
```
>approach 2：结合二分查找的动态规划

O(_n logn_)复杂度解法
