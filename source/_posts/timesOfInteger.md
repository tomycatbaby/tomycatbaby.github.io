---
title: 整数中1出现的次数
date: 2018-04-17 15:38:33
categories: 算法
tags: [剑指offer]
---
<div>
	<center>
		<iframe frameborder="no" border="0" marginwidth="0" marginheight="0" width=330 height=86 src="//music.163.com/outchain/player?type=2&id=552001753&auto=0&height=66"></iframe>
	</center>
</div>
求出1~13的整数中1出现的次数,并算出100~1300的整数中1出现的次数？为此他特别数了一下1~13中包含1的数字有1、10、11、12、13因此共出现6次,但是对于后面问题他就没辙了。ACMer希望你们帮帮他,并把问题更加普遍化,可以很快的求出任意非负整数区间中1出现的次数

> 思路

按照数学的排列组合思路来解，数字1位于不同的位置，共有多少种排列，包括重复的排列。
1、首先拿到一个整数，有m位，数字首位为s，则0~99..99（m-1个9）这个区间1出现的次数是sum=(m-1)\*10^(m-2)（1的重复排列）；
2、接下来count=s\*sum(以2345为例，m为4，0~999区间1出现的次数为1的重复排列3\*10\*10=300，count=2\*300=600)，统计的是后m-1位中1的重复排列与首位排列相乘；
3、后面如果首位s大于1，就加上10^(m-1)(以2345例子就加上1000，count=600+1000=1600)；
			   s等于1，就加上划去首位1的数再加上1（以1345例子看就是345+1，count+=346）
4、接下来就是递归了，我就不赘述了，看代码。
> 详细代码

```
import java.util.HashMap;
import java.util.Map;
public class Solution {
    public int NumberOf1Between1AndN_Solution(int n) {
        if(n==0)
            return 0;
        else if(n<10)
            return 1;
        int count = 0;
        int length = test(n)-1;//长度减一
        int divisor = (int) Math.pow(10,length);//除数
        int y = n%divisor;//余数
        int l = n/divisor;//数字首位
		int sum = (int)(length*Math.pow(10,length-1));
        count = l*sum;
        if(l>1)
            count+=divisor;
        else if(l==1)
            count=count+y+1;
        return count+NumberOf1Between1AndN_Solution(y);
        
    }
    public int test(int n){
        String str = String.valueOf(n);
        char[]chars = str.toCharArray();
        return chars.length;
    }
}
```