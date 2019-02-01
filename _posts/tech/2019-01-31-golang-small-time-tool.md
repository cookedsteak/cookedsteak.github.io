---
layout: post
title: Go时间交并集小工具
category: 技术
keywords: golang,go,工具
comments: true
toc: true
---

> 示例代码（含测试）[在这里](https://github.com/cookedsteak/timemix)

## 需求
在甘特图的场景下，我们经常会遇到这种情况，五位员工A, B, C, D, E，可能他们的工作都是并行的，我们需要计算
某段时间内他们总的工作时长。

我们不能简单得把五个人的工作时间都加起来，因为当中会有重叠的部分。
所以这时候我们就需要一个计算时间交并集的工具。

## 思路
将一组离散的时间段按照开始时间，从小到大排序。像这样
```
[{2 7} {4 11} {10 19} {10 30} {16 18} {19 29} {23 35} {24 42} {25 30} {27 49}]
```
我这里将时间用十分小的秒来代替，方便理解。

循环排序后的数组，如果下一个时间段开始时间介于上个时间段的开始时间和结束时间之间，那么就进行`合并`，否则就`分离`。
可以看到我们这里有两个关键动作，`合并`，`分离`，而这个就是我们要实现的核心代码。

## 实现
一段连续的工作时间都会有两个点，`开始时间`和`结束时间`。
所以我们可以把这个时间结构设计成：
```
type T struct {
	Start int64
	End   int64
}
```

而一个人的工作时间是由多个 T 组成的，所以我们在定义一个切片类型
```
type TSlice []T
```

为了能顺序合并时间，我们需要将`TSlice`进行排序。
我们知道 Go 中有个 sort 包，我们只需要实现 sort 类型的接口，就能实现 TSlice 的排序了。
我们实现下：
```
func (t TSlice) Len() int { return len(t) }

func (t TSlice) Swap(i, j int) { t[i], t[j] = t[j], t[i] }

func (t TSlice) Less(i, j int) bool { return t[i].Start < t[j].Start }
```
三个方法分别是，长度、交换位置、比小。

这样一来，我们就能直接用 `sort.Stable()` 稳定排序，对我们的时间段切片排序了。

好，接下来我们实现并集的方法，我们取名为 `Union`：
```
func (t TSlice) Union() TSlice {
    // 新建一个空的时间切片
	var s TSlice
    // 如果有至少两个是时间段，我们才排序，否则直接返回
	if len(t) > 1 {
		// @todo 合并逻辑
	}

	return s
}
```
Union 方法将会返回一个同样的 `TSlice` 时间切片，只不过是经过并集处理的。

一旦 t 中的时间段个数大于1，我们就要执行处理逻辑了：
```
if len(t) > 1 {
    sort.Stable(t)
    s = append(s, t[0])

    // @todo 循环比较合并
}
```
我们先对时间切片进行排序，然后把第一个时间段作为第一个元素放进我们的结果 TSlice 中，好让我们开始进行循坏的比较。
```
if len(t) > 1 {
		sort.Stable(t)
		s = append(s, t[0])

		for k, v := range t {
            // 如果开始时间大于结束时间，那其实是错误数据，但是我们这里正常返回
            // 你可以根据自己的需要定制错误处理逻辑
			if v.Start > v.End {
				return s
			}
            // 第一组元素我们不做任何操作
			if k == 0 {
				continue
			}

            // 当开始时间介于上一个时间段的开始时间和结束时间之间
			if v.Start >= s[len(s)-1].Start && v.Start <= s[len(s)-1].End {
				// 合并
				if v.End > s[len(s)-1].End {
					s[len(s)-1].End = v.End
				}
            // 如果大于上一个时间段的结束时间
			} else if v.Start > s[len(s)-1].End {
				// 分离
				inner := T{Start: v.Start, End: v.End}
				s = append(s, inner)
			}
		}
	}
```
来张图其实就清楚了：
![union](/assets/img/timemix/timemix1.png)

可以看到最后输出的也是一个 `TSlice` 类型。
上面就是 union，求并集的过程，那交集的？

其实交集也很简单，如果两个时间段相交，我们只要判断：开始时间取最大的那个，结束时间取两个时间段中最小的那个。
```
func (t TSlice) Intersect() TSlice {
	var s TSlice

	if len(t) > 1 {
		sort.Stable(t)
		s = append(s, t[0])

		for k, v := range t {
			if v.Start > v.End {
				return s
			}

			if k == 0 {
				continue
			}
            // 两个时间段相交
			if v.Start >= s[0].Start && v.Start <= s[0].End {
                // 开始时间取最大的那个
				s[0].Start = v.Start
                // 结束时间取最小的那个
				if v.End <= s[0].End {
					s[0].End = v.End
				}
			} else {
				return s[:0]
			}
		}
	}

	return s
}
```
一样，我们来个图：
![timemix2](/assets/img/timemix/timemix2.png)

需要注意的是，这个求交集的结果是全相交--只有当所有时间段都有共同时间才会有结果。
这样的需求在实际过程中用到的是不是不太多？？所以我想是不是能够实现：一次相交，两次相交...的条件筛选。


## 看看效果
我们随机生成了一组时间切片
```
func makeTimes(t int) TSlice {
    var set TSlice
	rand.Seed(time.Now().Unix())

	for i := 0; i < t; i++ {
		randStart := rand.Int63n(50)
		randEnd := randStart + rand.Int63n(25) + 1
		set = append(set, T{Start: randStart, End: randEnd})
	}

	return set
}

testSet := makeTimes(10) // 生成10个时间段的时间切片
res := testSet.Union()   // 直接调用 Union() 或者 Intersect()
```
输入数据为：
```
[{10 21} {34 52} {49 54} {18 31} {26 44} {24 27} {43 51} {41 53} {20 41} {48 67}]
```
输出结果：
```
[{10 67}]
```

结果还行~

## 额外
我发现在求并集的过程中，会要求求最终的时间之和，所以我们为 TSlice 加一个 Sum() 方法，
就是简单的循环求和：
```
func (t TSlice) Sum() (sum int64) {
	for i := 0; i < len(t); i++ {
		sum += t[i].End - t[i].Start
	}

	return sum
}
```