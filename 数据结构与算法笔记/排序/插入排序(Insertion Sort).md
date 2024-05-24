# 插入排序(Insertion Sort)

## 基本概念

插入排序是一种简单的排序，适合对较小的数据量进行排序，插入排序的基本思想类似于整理扑克牌，即从第一个元素开始**将第$n$个未排序的元素插入到前面$n-1$个有序的数据中**。

示例图：
![img](https://www.runoob.com/wp-content/uploads/2019/03/insertionSort.gif)



## 性质

### 时间复杂度

再最优的情况下，即所有元素都是有序的情况下，插入排序仍需从第二个元素开始遍历一遍数据并进行比较，因此插入排序的**最优时间复杂度为$O(N)$**。最坏的情况下，即所有元素都是逆序的情况下，第$n$个元素需要与前面$n-1$个元素进行比较，最后的结果就是$1+2+……+(n-1)$，也就是$n^2/2$，也就是说**时间复杂度为$O(N^2)$**。**平均时间复杂度为$O(N^2)$**。

### 空间复杂度

插入排序**不需要额外的空间**，所以**空间复杂度为$O(1)$**。

### 稳定性

当一个元素向前比较时,遇到相同的元素就会停下，并插入其后，相对位置不变，所以插入排序是**稳定**的。



## 实现

```c++
template <typename T>
inline void InsertSort(T* const ptr, const size_t count, bool cmp(const T* v1, const T* v2) = DefaultCmp)
{
	if (!count || !ptr) {
		return;
	}
	for (size_t i = 1; i < count; i++) {	//从第二个元素开始
		T value = *(ptr + i);	//记录第i个元素
		long long j = i - 1;	//要插入元素的前一个元素
		for (; j >= 0 && cmp(ptr + j, &value);) {	//cmp返回true时执行
			//*(ptr + j + 1) = *(ptr + j);	//若第j个元素比value大，第j个元素后移
			j--;	//索引前移
		}
		//使用memmove在元素内存很大时可显著提速
		std::memmove(ptr + j + 2, ptr + j + 1, sizeof(T) * (i - j - 1));
		*(ptr + j + 1) = value;	//插入元素
	}
}
```

实现时注意第n个元素不断与前面元素比较时可先不对前面的元素进行操作，而是**仅将索引前移**，最后在**统一移动**，可提高效率。