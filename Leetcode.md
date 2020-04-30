### [209. Minimum Size Subarray Sum](https://leetcode-cn.com/problems/minimum-size-subarray-sum)

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
```

### [210. Course Schedule II](https://leetcode-cn.com/problems/course-schedule-ii/)
` There are a total of n courses you have to take, labeled from 0 to n-1.
	Some courses may have prerequisites, for example to take course 0 you have to first take course 1, which is expressed as a pair: [0,1]
	Given the total number of courses and a list of prerequisite pairs, return the ordering of courses you should take to finish all courses.
	There may be multiple correct orders, you just need to return one of them. If it is impossible to finish all courses, return an empty array.

	Example 1:
	Input: 2, [[1,0]] 
	Output: [0,1]
	Explanation: There are a total of 2 courses to take. To take course 1 you should have finished course 0. So the correct course order is [0,1] .
	
	Example 2:
	Input: 4, [[1,0],[2,0],[3,1],[3,2]]
	Output: [0,1,2,3] or [0,2,1,3]
	Explanation: There are a total of 4 courses to take. To take course 3 you should have finished both courses 1 and 2. Both courses 1 and 2 should be taken after you finished course 0. So one correct course order is [0,1,2,3]. Another correct ordering is [0,2,1,3] .
	
	Note:
	The input prerequisites is a graph represented by a list of edges, not adjacency matrices. Read more about how a graph is represented.
	You may assume that there are no duplicate edges in the input prerequisites.

* 思路：投票。
	* 如果有A课依赖B课，就给A课投上一票。
	* 零投票的课程入队列，同时寻找有依赖该零投票课程的课程，减一票。
	* 如果有解，所有的课程都会变为0票。
	* 反之无解。
	* 题目的核心是：
		* 1.对有依赖于其他课程的课程投票；
		* 2.遍历减票的过程中使用Stack保存零投票的课程；

```(Java)
public int[] findOrder(int numCourses, int[][] prerequisites) {
  if (numCourses <= 0) {
    return new int[0];
  }

  int[] refs = new int[numCourses];
  for (int[] pair: prerequisites) {
    refs[pair[0]]++;
  }

  final Stack<Integer> emptyStack = new Stack<Integer>();
  for (int i = 0; i < numCourses; i++) {
    if (refs[i] <= 0) {
      emptyStack.push(i);
    }
  }

  final int[] array = new int[numCourses];
  int arrayIndex = 0;

  while (!emptyStack.isEmpty()) {
    final int emptyKey = emptyStack.pop();
    array[arrayIndex++] = emptyKey;

    for (int[] pair: prerequisites) {
      if (pair[1] == emptyKey) {
        refs[pair[0]]--;

        if (refs[pair[0]] <= 0) {
          emptyStack.push(pair[0]);
        }
      }
    }
  }

  if (arrayIndex < array.length) {
    return new int[0];
  }

  return array;
}
```