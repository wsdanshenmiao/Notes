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

### 顺序结构实现

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



### 单链表实现

单链表由于不能够随机访问和后退，因此实现与普通数组的快排稍有不同，但是核心思想相同，也是找定一个基准元素，将所有元素以该元素为基准进行划分，然后再递归的划分左右两个部分。

首先第一个不同就是需要遍历链表来找到最后一个元素来作为结束点随后调用递归实现的快排：

```C++
ListNode* SortList(ListNode* head)
{
    if (head == nullptr || head->next == nullptr) return head;
    // write code here
    // 获取最后一个节点
    auto tail = head;
    for (; tail->next != nullptr; tail = tail->next);
    QuickSort(head, tail);
    return head;
}
```

排序的主题部分与普通实现相同：

```C++
void QuickSort(ListNode* begin, ListNode* end) 
{
    if (begin == end || begin == nullptr || end == nullptr) return;
    ListNode* node = Partition(begin, end);
    QuickSort(begin, node);
    QuickSort(node->next, end);
}
```

不同的是其中的`Partition`函数的实现，由于单链表无法前向迭代，因此不能使用左右双指针迭代，需要在一趟顺序遍历中分离所有元素并找出分离点。一趟找出分离点的代码如下：
```C++
ListNode* Partition(ListNode* begin, ListNode* end) 
{
    assert(begin != nullptr && end != nullptr);
    if (begin == end) return begin;
    auto sentry = begin->val;
    ListNode* left = begin, * right = begin->next;
    for (; right != end->next; right = right->next) {
        if (right->val < sentry) {
            left = left->next;
            Swap(left, right);
        }
    }
    Swap(begin, left);
    return left;
}
```

该方法的基本思想为将第一个元素作为基准，第一和第二个元素作为双指针的起点，right会逐个遍历所有元素，若right所处的元素比基准元素小，则将left前进并交换left和right，这样就能保证  (pivot, left ]区间内的元素都比基准小，(left, right] 区间内所有元素都比基准大，当right遍历完所有元素，前面的准则依然成立，因此left此时的位置就是基准元素所在的位置。



完整实现如下：

```C++
// 使用快速排序对链表进行排序
class ListQuickSort {
public:
    ListNode* SortList(ListNode* head)
    {
        if (head == nullptr || head->next == nullptr) return head;
        // write code here
        // 获取最后一个节点
        auto tail = head;
        for (; tail->next != nullptr; tail = tail->next);
        QuickSort(head, tail);
        return head;
    }


private:
    void QuickSort(ListNode* begin, ListNode* end) 
    {
        if (begin == end || begin == nullptr || end == nullptr) return;
        ListNode* node = Partition(begin, end);
        QuickSort(begin, node);
        QuickSort(node->next, end);
    }

    ListNode* Partition(ListNode* begin, ListNode* end) 
    {
        assert(begin != nullptr && end != nullptr);
        
        if (begin == end) return begin;
        auto sentry = begin->val;
        ListNode* left = begin, * right = begin->next;
        for (; right != end->next; right = right->next) {
            if (right->val < sentry) {
                left = left->next;
                Swap(left, right);
            }
        }
        Swap(begin, left);
        return left;
    }

    void Swap(ListNode* n0, ListNode* n1) 
    {
        assert(n0 != nullptr && n1 != nullptr);
        int tmp = n0->val;
        n0->val = n1->val;
        n1->val = tmp;
    }

};
```

