# 最大子数组和问题（Maximum Subarray Problem）

题目：

https://leetcode.cn/problems/maximum-subarray/description/

给你一个整数数组 `nums` ，请你找出一个具有最大和的连续子数组（子数组最少包含一个元素），返回其最大和。

子数组是数组中的一个连续部分。

```bash
示例 1：

输入：nums = [-2,1,-3,4,-1,2,1,-5,4]
输出：解释连续子数[4,-1,2,1] 的和最大，为 6 。

示例 2：

输入：ms = [1]
输出 1

示例 3：

输入：ms = [5,4,-1,7,8]
输出 23
```

## 解释思路（Kadane 算法）

- 从左往右遍历数组；

- `currSum` 表示当前连续子数组的和；

- 如果当前值 `nums[i]` 比 `currSum + nums[i]` 更大，就丢弃之前的子数组，重新开始；

- 记录出现过的最大值作为 `maxSum`。

## 代码

```go
func maxSubArray(nums []int) int {
    if len(nums) == 0 {
        return 0
    }

    maxSum := nums[0]
    currSum := nums[0]

    for i := 1; i < len(nums); i++ {
        currSum = max(nums[i], currSum+nums[i])
        maxSum = max(maxSum, currSum)
    }

    return maxSum
}

func max(a, b int) int {
    if a > b {
        return a
    }
    return b
}
```

| 优点    | 内容             |
| ----- | -------------- |
| 时间复杂度 | `O(n)`，只遍历一次数组 |
| 空间复杂度 | `O(1)`，不使用额外数组 |
| 适用场景  | 连续子数组最大和       |


# 两数之和

题号：1
给定一个整数数组 nums 和一个整数目标值 target，请你在该数组中找出 和为目标值 target  的那 两个 整数，并返回它们的数组下标。
你可以假设每种输入只会对应一个答案。但是，数组中同一个元素在答案里不能重复出现。
你可以按任意顺序返回答案。

```go
func twoSum(nums []int, target int) []int {
	// 创建一个map, key为数组中的值，value为下标
	m := make(map[int]int)
	for k, v := range nums {
		// 将已经遍历过的值存入map中，通过map的key快速查找
		if i, ok := m[target-v]; ok {
			return []int{i, k}
		}
		m[v] = k
	}
	return nil
}
```

# 回文数

题号：9
给你一个整数 x ，如果 x 是一个回文整数，返回 true ；否则，返回 false 。
回文数是指正序（从左向右）和倒序（从右向左）读都是一样的整数。例如，121 是回文，而 123 不是。

```go
func isPalindrome(x int) bool {
	// 判断边界，负数不是回文数
	if x < 0 {
		return false
	}
	if x == 0 {
		return true
	}
	// 整数转为字符串
	str := strconv.Itoa(x)
	for i := 0; i < len(str)/2; i++ {
		if str[i] != str[len(str)-1-i] {
			return false
		}
	}
	return true
}
```

# 丢失的数字

题号：268
给定一个包含 [0, n] 中 n 个数的数组 nums ，找出 [0, n] 这个范围内没有出现在数组中的那个数。

```go
func missingNumber(nums []int) int {
	var result int
	// 对数组进行排序
	sort.Ints(nums)
	// 如果是缺失的值不是最后一个，则数组的下标不会等于值
	for k, v := range nums {
		if k != v {
			result = k
			return result
		}
	}
	// 如果缺失的值是最后一个，则返回数组最后一个值
	result = len(nums)
	return result
}
```
