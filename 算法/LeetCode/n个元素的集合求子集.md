# n个元素的集合求子集



## 问题

给定一组**不含重复元素**的整数数组 *nums*，返回该数组所有可能的子集（幂集）。

https://leetcode-cn.com/problems/subsets/



## 分析

这道题在阿里面试的时候被问过了，当时应该直接使用回溯法，然后将清楚怎么递归回溯的。

回到正题，首先讲讲递归，递归的话，可能有很多层，从而把脑袋给绕晕了，引用前辈的做法，在递归的时候，脑袋里面只想2层，这样子脑袋就不会晕了。

然后就是这一题，n个元素的子集，那么每个元素可能在集合中，也可能不在集合中。想到这个地方，然后在脑海中递归两层，解法就出来了。

假如这个集合只有2个元素a、b，那么可能a、b都在集合中，ab都考虑过后（即全部元素都考虑过后），一种方案就出来了，加到结果集中。然后把b给去掉，另外一种方案也出来了。



## 代码

```java
import java.util.ArrayList;
import java.util.List;

/**
 * @author huangy on 2020-04-11
 */
public class Subsets {

    private List<List<Integer>> result = new ArrayList<>();

    private int[] nums;

    /**
     * 回溯法求解
     */
    public List<List<Integer>> subsets(int[] nums) {
        this.nums = nums;

        dfs(0, new ArrayList<>());

        return result;
    }

    /**
     * 回溯法
     */
    private void dfs(int t, List<Integer> currentList) {

        if (t == nums.length) {
            result.add(new ArrayList<>(currentList));
            return;
        }

        currentList.add(nums[t]);

        dfs(t+1, currentList);

        currentList.remove(Integer.valueOf(nums[t]));

        dfs(t+1, currentList);
    }

    public static void main(String[] args) {
        int[] nums = {1, 2, 3};
        System.out.println(
                new Subsets().subsets(nums));;
    }
}
```









## 参考

[n个元素的子集](https://m.toutiaocdn.com/group/6813999174061654539/?app=news_article&timestamp=1586562985&req_id=202004110756250101290421340E42B631&group_id=6813999174061654539&tt_from=mobile_qq&utm_source=mobile_qq&utm_medium=toutiao_android&utm_campaign=client_share)

