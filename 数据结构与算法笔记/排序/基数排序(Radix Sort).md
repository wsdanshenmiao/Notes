# 基数排序(Radix Sort)

## 基本概念

基数排序与计数排序十分类似，相当于相对计数排序在空间和时间上的折中，也是**非比较排序算法**。其基本思想是**按数据的最大位数进行拆分**，再依次**对每一位进行一轮排序**，一般是使用计数排序对每一位排序。基数排序分为两种，第一种是按从小位到大位的顺序进行比较，也就是从左到右，被称为**MSD(Most Significant Digit first)**基数排序；第二种是按从大位到小位的顺序进行比较，也就是从右到左，被称为**LSD(Least Significant Digit first)**基数排序。MSD需要使用**递归**来实现，适合排序字符串，LSD使用循环即可，适合排序自然数。

**实现步骤：**

1. 找出待排序的数组中**最大的元素**，并**对其位数进行拆分**。
2. 按照数据的基数分配计数数组，按照数据量分配计数数组。
3. 按每一位进行一轮计数排序。



## 性质

### 时间复杂度

基数排序的时间复杂度为$O(d(n+k))$，其中d为数据的最大位数，k为数据的基数，如十进制的10，或字母的26，由此当最大位数很小、基数很大时很适合使用基数排序。

### 空间复杂度

空间复杂度为$O(n+k)$，k为数据的基数，k相对计数排序小很多。

### 稳定性

与计数排序相同是稳定的算法。



## 实现

```c++
template <typename T>
inline void RadixCountSort(T* const ptr, const size_t count, bool(const T*, const T*) = DefaultCmp)
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
	std::vector<T> tmpData;
	tmpData.reserve(count);
	tmpData.resize(count, 0);
	//分别按每个位分配
	for (size_t digit = 1; max / digit > 0; digit *= 10) {
		//每一位的计数排序
		std::vector<size_t> countArr;
		countArr.resize(10, 0);
		//*(ptr + i) / digit为需要排序的位数，如123/100=1；再%10为要计数的数
		for (size_t i = 0; i < count; countArr[*(ptr + i) / digit % 10]++, i++);
		//累计数组，方便排序
		for (size_t i = 1; i < 10; countArr[i] += countArr[i - 1], i++);
		//收集按位排序后的数据
		for (size_t i = count; i > 0; i--) {
			//把第i个原始数放在累计数组中第原始数个累计数减一的下标处
			tmpData[countArr[*(ptr + i - 1) / digit % 10]-- - 1] = *(ptr + i - 1);
		}
		std::memcpy(ptr, tmpData.data(), count * sizeof(T));
	}
}
```

其中`for (size_t digit = 1; max / digit > 0; digit *= 10)`即为按每一位进行一轮计数排序，其余的与计数排序类似。