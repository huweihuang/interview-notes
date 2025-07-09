> 本文介绍go常用的算法相关的包

# 排序

可以使用内置sort的包进行排序。

## 整数排序

```go
import "sort"

# 升序
ints := []int{3, 1, 4, 2}
sort.Ints(ints) // 升序排序


# 降序
nums := []int{5, 2, 8, 3}

sort.Slice(nums, func(i, j int) bool {
    return nums[i] > nums[j] // 注意这里是 > 实现降序
})

fmt.Println(nums) // 输出: [8 5 3 2]

# 判断是否已经排序
sort.IntsAreSorted([]int)      // bool
sort.StringsAreSorted([]string) // bool
```

## 字符串排序

```go
sort.Strings([]string)
```

## sort.Slice

`sort.Slice` 是 Go 语言中一个非常实用的函数，用于对 **任意切片（slice）进行排序**，不需要你显式实现 `sort.Interface` 接口。

```go
func Slice(slice any, less func(i, j int) bool)
```

slice: 任意类型的切片（例如 []int、[]string、[]struct{}）

less: 一个比较函数，当 less(i, j) 返回 true 时，表示 slice[i] 应排在 slice[j] 之前

结构体切片按字段排序：

```go
type Person struct {
    Name string
    Age  int
}

people := []Person{
    {"Alice", 30},
    {"Bob", 25},
    {"Charlie", 35},
}

// 按 Age 升序
sort.Slice(people, func(i, j int) bool {
    return people[i].Age < people[j].Age
})
```
