# 计数排序(Counting sort)

## 基本概念

计数排序是一种**非比较排序算法**，在排序时无需比较要排序的数据，也是一种通过更大的空间开销来减小时间开销的算法。其基本思想是将所有元素**按照其大小**分配到**计数数组**中与之对应的位置上，相同的元素分配到同一位置上。具体来说就是将数据中大小为x的元素分配到计数数组中对应索引为x的位置上。

**实现步骤：**

1. 找出待排序的数组中**最大的元素**，并分配大小与之相同的**计数数组**。
2. 统计数组中每个元素出现的次数，存入该元素对应的计数数组中。
3. 累**计计数数组中所有的计数**。
4. **逆序遍历**数据，按照累计数组放入原始空间对应位置中，相应的计数减一。

示例图：
![img](https://www.runoob.com/wp-content/uploads/2019/03/countingSort.gif)



## 性质

### 时间复杂度

计数排序的时间复杂度为$O(N+K)$，其中K代表待排序数据的最大元素。

### 空间复杂度

空间复杂度为$O(N+K)$，N为辅助空间的大小，K为计数数组的大小。

### 稳定性

在最后填充辅助空间时要**逆序遍历原始数据**，这样才能做到在后面的相同元素依旧在后面，才能使排序**稳定**。



## 实现

```c++
template <typename T>
inline void CountingSort(T* const ptr, const size_t count,bool (const T*, const T*) = DefaultCmp)
{
	if (!count || !ptr) {
		return;
	}
	//寻找最大值
	size_t max = *ptr;
	for (size_t i = 0; i < count; i++) {
		if (*(ptr + i) > max) {
			max = *(ptr + i);
		}
	}
	//计数数组
	std::vector<size_t> countArray;
	countArray.reserve(max + 1);
	countArray.resize(max + 1, 0);
	for (size_t i = 0; i < count; countArray[*(ptr + i)]++, i++);
	//累计数组，提高稳定性，最后要倒叙遍历
	for (size_t i = 1; i < max + 1; countArray[i] += countArray[i - 1], i++);
	//进行排序
	std::vector<T> tmpData;
	tmpData.reserve(count);
	tmpData.resize(count, 0);
	for (size_t i = count; i > 0; i--) {
		//把第i个原始数放在累计数组中第原始数个累计数减一的下标处
		tmpData[countArray[*(ptr + i - 1)]-- - 1] = *(ptr + i - 1);
	}
	std::memcpy(ptr, tmpData.data(), count * sizeof(T));
}
```

