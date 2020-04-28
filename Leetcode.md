### 209. Minimum Size Subarray Sum
link：https://leetcode-cn.com/problems/minimum-size-subarray-sum

` Given an array of n positive integers and a positive integer s, find the minimal length of a contiguous subarray of which the sum ≥ s. If there isn't one, return 0 instead.
	Example: 
	Input: s = 7, nums = [2,3,1,2,4,3]
	Output: 2
	Explanation: the subarray [4,3] has the minimal length under the problem constraint.
	Follow up:
	If you have figured out the O(n) solution, try coding another solution of which the time complexity is O(n log n). 

* 思路：滑动窗口思路
	* 双指针分别指向窗口的收尾，尾指针不断向后移动，直到累加值大于等于s
	* 对比并记录最小长度，同时首指针不断向后移动，直到累加值小于s
	* 继续尾指针的移动，直到超过数组长度

```(Java)
class Solution {
    public int minSubArrayLen(int s, int[] nums) {
        if (null == nums || nums.length <= 0) return 0;

        int left = 0;
        int right = 0;
        int sum = 0;
        int len = 0;

        while (right < nums.length) {
            sum += nums[right];

            while (sum >= s) {
                len = len <= 0 ? right - left + 1 : Math.min(len, right - left + 1);
                sum -= nums[left++];
            }

            right++;
        }

        return len;
    }
}
```