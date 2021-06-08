# 面试常考排序算法——Go语言实现


<!--more-->

LeetCode相关题：[https://leetcode-cn.com/problems/sort-an-array/](https://leetcode-cn.com/problems/sort-an-array/)

## 插入排序

~~~go
func insertionSort(nums []int, left, right int) {
    for i := left + 1; i <= right; i++ {
        j, temp := i-1, nums[i]
        for ; j >= 0 && nums[j] > temp; j-- {
            nums[j+1] = nums[j]
        }
        nums[j+1] = temp
    }
}
~~~

## 快速排序

~~~go
func quickSort(nums []int, left, right int) {
    if left >= right {
        return
    }
    index := partition(nums, left, right)
    quickSort(nums, left, index-1)
    quickSort(nums, index+1, right)
}

func partition(nums []int, left, right int) int {
    // 随机选取一个元素作为基准
    pos := rand.Intn(right-left+1) + left
    nums[left], nums[pos] = nums[pos], nums[left]
    i, j, pivot := left, right, nums[left]
    for i < j {
        for i < j && nums[j] >= pivot {
            j--
        }
        for i < j && nums[i] <= pivot {
            i++
        }
        nums[i], nums[j] = nums[j], nums[i]
    }
    nums[i], nums[left] = nums[left], nums[i]
    return i
}
~~~

## 归并排序

~~~go
func mergeSort(nums []int) []int {
    if len(nums) <= 1 {
        return nums
    }

    mid := len(nums) / 2 
    left, right := mergeSort(nums[:mid]), mergeSort(nums[mid:])
    return mergeTwoArrays(left, right)
}

func mergeTwoArrays(left, right []int) []int {
    m, n := len(left), len(right)
    res := make([]int, m+n)
    i, j := 0, 0
    for i < m && j < n {
        if left[i] < right[j] {
            res[i+j] = left[i]
            i++
        } else {
            res[i+j] = right[j]
            j++
        }
    }

    for i < m {
        res[i+j] = left[i]
        i++
    }

    for j < n {
        res[i+j] = right[j]
        j++
    }

    return res
}
~~~

## 堆排序

~~~go 
func heapSort(nums []int, left, right int) {
    for i := right / 2; i >= 0; i-- {
        sink(nums, i, right)
    }

    for i := right; i > left; i-- {
        nums[i], nums[left] = nums[left], nums[i]
        sink(nums, left, i-1)
    }
}

func sink(nums []int, root, end int) {
    for {
        child := 2*root + 1 
        if child > end {
            return
        }
        if child < end && nums[child] <= nums[child+1] {
            child++
        }
        if nums[root] > nums[child] {
            return
        }
        nums[root], nums[child] = nums[child], nums[root]
        root = child
    }
}
~~~
