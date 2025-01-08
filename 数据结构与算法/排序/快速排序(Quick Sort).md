# 快速排序(Quick Sort)

## 基本概念

快速排序是一种应用十分广泛的排序算法，其采用的方法也是**分治法**，即分而治之。其基本思想是每次**选定一个元素作为中心轴**（ 也称作**基准pivot**），在分别**设置左右指针**，以递增为例，左指针负责将比中心轴大的元素移至中心轴右边，右指针负责将比中心轴小的元素移至中心轴左边，每次移动左指针自加，右指针自减，**左右指针相遇时即为中心轴的位置**，也就是说**每一次遍历都可以确定一个元素的位置**，最后再**分别对中心轴的左右部分执行相同的操作**，直到所有元素位置确定为止。

示例图：
![img](https://www.runoob.com/wp-content/uploads/2019/03/quickSort.gif)



## 性质

### 时间复杂度

快速排序的**最优和平均时间复杂度都为$O(NlogN)$**，**最坏时间复杂度为$O(NlogN) $**，尽管如此快速排序通常比其他$O(NlogN) $算法更快，这是因为在实践中快速排序几乎不可能达到最坏情况，且其平均时间复杂度$O(NlogN)$，其中**隐含的常数因子很小**，快速排序的**内存访问遵循局部性原理**，不需要频繁地访问远处的内存位置，从而减**少了缓存未命中**的情况，提高了算法的效率。

### 空间复杂度

快速排序不适用辅助空间，并且递归调用深度通常为$O(logN)$,因此快速排序的平均空间复杂度为$O(logN)$,最坏的情况下是$O(N)$。

### 稳定性

快速排序是不稳定的算法。



## 实现

```c++
template <typename T>
inline void QuickSort(T* const ptr, const size_t count, bool cmp(const T*, const T*) = DefaultCmp)
{
	if (!count || !ptr) {
		return;
	}
	//选定第一个元素为中心轴
	T pivot = *ptr;
	//左右指针
	T* left = ptr, * right = ptr + count - 1;
	if (left < right) {
		for (; left < right;) {
			//以递增为例，若右指针指向的元素比中心轴大或相等，右指针减减
			while (left < right && cmp(right, &pivot) || *right == pivot) {
				--right;
			}
			//否则右指针的元素移到左指针处
			*left = *right;
			//若左指针指向的元素比中心轴小或相等，左指针加加
			while (left < right && cmp(&pivot, left) || *left == pivot) {
				++left;
			}
			//否则左指针的元素移到右指针处
			*right = *left;
		}
		//确定中心轴的位置
		*left = pivot;
		//左侧递归
		QuickSort(ptr, left - ptr, cmp);
		//右侧递归
		QuickSort(left + 1, count - (left + 1 - ptr), cmp);
	}
}
```

