# STL

**STL基本概念：**

* STL(Standard Template Library,**标准模板库**)。
* STL 从广义上分为: **容器(container) 算法(algorithm) 迭代器(iterator)。**
* **容器**和**算法**之间通过**迭代器**进行无缝连接。
* STL 几乎所有的代码都采用了模板类或者模板函数。

**STL六大组件:**

STL大体分为六大组件，分别是:**容器、算法、迭代器、仿函数、适配器（配接器）、空间配置器**

1. 容器：各种数据结构，如vector、list、deque、set、map等,用来存放数据。
2. 算法：各种常用的算法，如sort、find、copy、for_each等。
3. 迭代器：扮演了容器与算法之间的胶合剂。
4. 仿函数：行为类似函数，可作为算法的某种策略。
5. 适配器：一种用来修饰容器或者仿函数或迭代器接口的东西。
6. 空间配置器：负责空间的配置与管理。



## 容器

### string容器

#### string的基本概念

**本质：**

* string是C++风格的字符串，而string本质上是一个类

**string和char * 区别：**

* char* 是一个**指针**
* string是一个**类**，类内部封装了char\*，管理这个字符串，是一个**char*型的容器**。

**特点：**

+ string 类内部**封装了很多成员方法**，例如：查找find，拷贝copy，删除delete 替换replace，插入insert

+ string管理char*所分配的内存，**不用担心复制越界和取值越界等**，由类内部进行负责



#### string构造函数

**构造函数原型：**

* `string();`								     	 //创建一个空的字符串
* `string(const char* s);`	           //使用字符串s初始化
* `string(const char* s, int n);`//使用字符串s前n个字符进行初始化
* `string(const string& str);`	  //使用一个string对象初始化另一个string对象   
* `string(int n, char c);`	           //使用n个字符c初始化 



#### string赋值操作

**赋值的函数原型：**

* `string& operator=(const char* s);`             //char*类型字符串 赋值给当前的字符串
* `string& operator=(const string &s);`         //把字符串s赋给当前的字符串
* `string& operator=(char c);`                          //字符赋值给当前的字符串
* `string& assign(const char *s);`                  //把字符串s赋给当前的字符串
* `string& assign(const char *s, int n);`     //把字符串s的前n个字符赋给当前的字符串
* `string& assign(const string &s);`              //把字符串s赋给当前字符串
* `string& assign(int n, char c);`                  //用n个字符c赋给当前字符串



#### string字符串拼接

**函数原型：**

* `string& operator+=(const char* str);`                   //重载+=操作符
* `string& operator+=(const char c);`                         //重载+=操作符
* `string& operator+=(const string& str);`                //重载+=操作符
* `string& append(const char *s); `                               //把字符串s连接到当前字符串结尾
* `string& append(const char *s, int n);`                 //把字符串s的前n个字符连接到当前字符串结尾
* `string& append(const string &s);`                           //同operator+=(const string& str)
* `string& append(const string &s, int pos, int n);`//字符串s中从pos开始的n个字符连接到字符串结尾



#### string查找和替换

**函数原型：**

* `int find(const string& str, int pos = 0) const;`              //查找str第一次出现位置,从pos开始查找
* `int find(const char* s, int pos = 0) const; `                     //查找s第一次出现位置,从pos开始查找
* `int find(const char* s, int pos, int n) const; `               //从pos位置查找s的前n个字符第一次位置
* `int find(const char c, int pos = 0) const; `                       //查找字符c第一次出现位置
* `int rfind(const string& str, int pos = npos) const;`      //查找str最后一次位置,从pos开始查找
* `int rfind(const char* s, int pos = npos) const;`              //查找s最后一次出现位置,从pos开始查找
* `int rfind(const char* s, int pos, int n) const;`              //从pos查找s的前n个字符最后一次位置
* `int rfind(const char c, int pos = 0) const;  `                      //查找字符c最后一次出现位置
* `string& replace(int pos, int n, const string& str); `       //替换从pos开始n个字符为字符串str
* `string& replace(int pos, int n,const char* s); `                 //替换从pos开始的n个字符为字符串s



#### string字符串比较

**比较方式：**

* 字符串比较是按字符的ASCII码进行对比
	= 返回   0
	\> 返回   1 
	< 返回  -1

**函数原型：**

* `int compare(const string &s) const; `  //与字符串s比较
* `int compare(const char *s) const;`      //与字符串s比较



#### string字符存取

string中单个字符存取方式有两种

* `char& operator[](int n); `     //通过[]方式取字符
* `char& at(int n);   `                    //通过at方法获取字符



#### string插入和删除

**函数原型：**

* `string& insert(int pos, const char* s);  `                //插入字符串
* `string& insert(int pos, const string& str); `        //插入字符串
* `string& insert(int pos, int n, char c);`                //在指定位置插入n个字符c
* `string& erase(int pos, int n = npos);`                    //删除从Pos开始的n个字符 



#### string子串

**函数原型：**

* `string substr(int pos = 0, int n = npos) const;`   //返回由pos开始的n个字符组成的字符串

**示例：**

```c++
std::string email("danshenmiao@outlook.com");
int pos = email.find('@');
std::string username = email.substr(0, pos);
std::cout << username << std::endl;
```



### vector容器

#### vector的基本概念

* vector数据结构和**数组非常相似**，也称为**单端数组**
* 不同之处在于数组是静态空间，而vector可以**动态扩展**
* vector容器的迭代器是支持随机访问的迭代器

创建一个动态数组（**==动态数组在堆上创建内存==**）：

```c++
std::vector<type> v;	//创建一个动态数组
```

**在当前函数中==构造一个对象==，再将此对象==复制到数组中==：**

```c++
v.push_back({ 1,2 });	//用push_back给数组追加
```

**动态扩展：**

* **动态数组不是在原空间之后续接新空间，而是找==更大的内存空间==，然后==将原数据拷贝新空间==，==释放原空间==。**

**==动态数组的使用优化：==**

 1. 可**==提前为数组设置空间==**，省去每次扩大空间时造成的不必要的复制。

 	v.reserve(3);	//为数组提前创建3个空间

 2. **==使用emplace_back==**，传递构造函数的参数列表，直接使用参数列表在动态数组的内存内创建对象，从而避免不必要的复制。

 	v.emplace_back(1, 2);



#### vector构造函数

**函数原型：**

* `vector<T> v; `               		     		//采用模板实现类实现，默认构造函数

	```c++
	std::vector<int> v;
	```

* `vector(v.begin(), v.end());   `    //拷贝v[begin(), end())区间中的元素，是前闭后开，v.end()不包括在内

* `vector(n, elem);`                           //构造函数将n个elem拷贝给本身。

	```c++
	std::vector<char> v(10, 'a');
	```

* `vector(const vector &vec);`       //拷贝构造函数。

	```c++
	std::vector<int> vec(10, 0);
	std::vector<int> v(vec)
	```



#### vector赋值操作

**函数原型：**

* `vector& operator=(const vector &vec);`//重载等号操作符

	```c++
	std::vector<int> v1; //无参构造
	v1.push_back(1);
	std::vector<int> v2 = v1;
	```


* `assign(beg, end);`       //将[beg, end)区间中的数据拷贝赋值给本身。

	```c++
	std::vector<int> v2.assign(v1.begin(), v1.end());
	```

* `assign(n, elem);`        //将n个elem拷贝赋值给本身。

	```c++
	v2.assign(10, 2);
	```

	

#### vector容量和大小

**函数原型：**

* `capacity();`                      //返回容器的容量

* `size();`                              //返回容器中元素的个数

* `resize(int num);`            //重新指定容器的长度为num，若容器变长，则以默认值0填充新位置；
          												                                           如果容器变短，则末尾超出容器长度的元素被删除。

* `resize(int num, elem);` //重新指定容器的长度为num，若容器变长，则以elem值填充新位置。

  ​				                                如果容器变短，则末尾超出容器长度的元素被删除



#### vector插入和删除

**函数原型：**

* `push_back(ele);`                                         			   //尾部插入元素ele
* `pop_back();`                                                                 //删除最后一个元素
* `insert(const_iterator pos, ele);`                     //迭代器指向位置pos插入元素ele
* `insert(const_iterator pos, int count,ele);`//迭代器指向位置pos插入count个元素ele
* `erase(const_iterator pos);`                                 //删除迭代器指向的元素
* `erase(const_iterator start, const_iterator end);`//删除迭代器从start到end之间的元素
* `clear();`                                                                       //删除容器中所有元素



####  vector数据存取

**函数原型：**

* `at(int idx); `     //返回索引idx所指的数据
* `operator[]; `       //返回索引idx所指的数据
* `front(); `            //返回容器中第一个数据元素
* `back();`              //返回容器中最后一个数据元素



#### vector互换容器

**函数原型：**

* `swap(vec);`  // 将vec与本身的元素互换

**应用：收缩内存**

```c++
vector<int> v;
for (int i = 0; i < 100000; i++) {
	v.emplace_back(i);
}
v.resize(3);
vector<int>(v).swap(v); //收缩内存
```

+ `vector<int>(v).swap(v); //收缩内存`,会先**创建一个容量为v的大小的匿名对象**，再与v互换，最后由系统释放匿名对象的内存，可**防止内存浪费**。



#### vector预留空间

**函数原型：**

* `reserve(int len);`		//容器预留len个元素长度，预留位置不初始化，元素不可访问。

	```c++
	vector<int> v;
	v.reserve(100000);	//预留空间
	int num = 0;
	int* p = NULL;
	for (int i = 0; i < 100000; i++) {
		v.push_back(i);
		if (p != &v[0]) {	//统计扩大了几次
			p = &v[0];
			num++;
		}	
	    std::cout << num <<std::endl;
	}
	```

若数据量较大，可提前预留空间，减少复制的次数，提高效率。





### deque容器

####  deque容器基本概念

**功能：**

* 双端数组，可以对头端进行插入删除操作

**deque与vector区别：**

* vector对于**头部的插入删除效率低**，数据量越大，效率越低
* deque相对而言，对**头部的插入删除速度回比vector快**
* **vector访问元素时的速度会比deque快**,这和两者内部实现有关

**deque内部工作原理:**

+ deque内部有个**中控器**，维护每段缓冲区中的内容，**维护的是每个缓冲区的地址**，缓冲区中存放真实数据，使得使用deque时像一片连续的内存空间。

* deque容器的迭代器也是**支持随机访问**的。



#### deque构造函数

**函数原型：**

* `deque<T>` ;                      //默认构造形式
* `deque(beg, end);`                  //构造函数将[beg, end)区间中的元素拷贝给本身。
* `deque(n, elem);`                    //构造函数将n个elem拷贝给本身。
* `deque(const deque &deq);`   //拷贝构造函数



#### deque赋值操作

**功能描述：**

* 给deque容器进行赋值

**函数原型：**

* `deque& operator=(const deque &deq); `         //重载等号操作符


* `assign(beg, end);`                                           //将[beg, end)区间中的数据拷贝赋值给本身。
* `assign(n, elem);`                                             //将n个elem拷贝赋值给本身。



#### deque大小操作

**函数原型：**

* `deque.empty();`                       //判断容器是否为空

* `deque.size();`                         //返回容器中元素的个数

* `deque.resize(num);`                //重新指定容器的长度为num,若容器变长，以默认值填充，若容器变短，则末尾超出容器长度的元素被删除。

* `deque.resize(num, elem);`     //重新指定容器的长度为num,若容器变长，以elem值填充，若容器变短，则末尾超出容器长度的元素被删除。



#### deque 插入和删除

**函数原型：**

两端插入操作：

- `push_back(elem);`          //在容器尾部添加一个数据
- `push_front(elem);`        //在容器头部插入一个数据
- `pop_back();`                   //删除容器最后一个数据
- `pop_front();`                 //删除容器第一个数据

指定位置操作：

* `insert(pos,elem);`         //在pos位置插入一个elem元素的拷贝，返回新数据的位置。

* `insert(pos,n,elem);`     //在pos位置插入n个elem数据，无返回值。

* `insert(pos,beg,end);`    //在pos位置插入[beg,end)区间的数据，无返回值。

* `clear();`                           //清空容器的所有数据

* `erase(beg,end);`             //删除[beg,end)区间的数据，返回下一个数据的位置。

* `erase(pos);`                    //删除pos位置的数据，返回下一个数据的位置。



#### deque 数据存取

**函数原型：**

- `at(int idx); `     //返回索引idx所指的数据
- `operator[]; `      //返回索引idx所指的数据
- `front(); `            //返回容器中第一个数据元素
- `back();`              //返回容器中最后一个数据元素



#### deque 排序

**算法：**

* `sort(iterator beg, iterator end)`  //对beg和end区间内元素进行排序





### stack容器

####  stack 基本概念

**概念：**stack是一种**先进后出**(First In Last Out,FILO)的数据结构，它只有一个出口

+ 栈中只有顶端的元素才可以被外界使用，因此栈不允许有遍历行为
+ 栈中进入数据称为  --- **入栈**  `push`
+ 栈中弹出数据称为  --- **出栈**  `pop`



####  stack 常用接口

构造函数：

* `stack<T> stk;`                                 //stack采用模板类实现， stack对象的默认构造形式
* `stack(const stack &stk);`            //拷贝构造函数

赋值操作：

* `stack& operator=(const stack &stk);`           //重载等号操作符

数据存取：

* `push(elem);`      //向栈顶添加元素
* `pop();`                //从栈顶移除第一个元素
* `top(); `                //返回栈顶元素

大小操作：

* `empty();`            //判断堆栈是否为空
* `size(); `              //返回栈的大小





### queue 容器

####  queue 基本概念

**概念：**Queue是一种**先进先出**(First In First Out,FIFO)的数据结构，它有两个出口

+ 队列容器允许从一端新增元素，从另一端移除元素
+ 队列中只有队头和队尾才可以被外界使用，因此队列不允许有遍历行为
+ 队列中进数据称为 --- **入队**    `push`
+ 队列中出数据称为 --- **出队**    `pop`



####  queue 常用接口

构造函数：

- `queue<T> que;`                                 //queue采用模板类实现，queue对象的默认构造形式
- `queue(const queue &que);`            //拷贝构造函数

赋值操作：

- `queue& operator=(const queue &que);`           //重载等号操作符

数据存取：

- `push(elem);`                             //往队尾添加元素
- `pop();`                                      //从队头移除第一个元素
- `back();`                                    //返回最后一个元素
- `front(); `                                  //返回第一个元素

大小操作：

- `empty();`            //判断堆栈是否为空
- `size(); `              //返回栈的大小





### list容器

####  list基本概念

**功能：**将数据进行链式存储

**链表**（list）是一种物理存储单元上非连续的存储结构，数据元素的逻辑顺序是通过链表中的指针链接实现的

+ 链表的组成：链表由一系列**结点**组
+ 结点的组成：一个是存储数据元素的**数据域**，另一个是存储下一个结点地址的**指针域**
+ STL中的链表是一个双向循环链表
+ 由于链表的存储方式并不是连续的内存空间，因此链表list中的迭代器只支持前移和后移，属于**双向迭代器**
+ List有一个重要的性质，插入操作和删除操作都不会造成原有list迭代器的失效，这在vector是不成立的。

**list的优点：**

* 采用动态存储分配，不会造成内存浪费
* 链表执行插入和删除操作十分方便，修改指针即可，不需要移动大量元素

**list的缺点：**

* 链表灵活，但是空间(指针域) 和 时间（遍历）额外耗费较大



####  list构造函数

**函数原型：**

* `list<T> lst;`                               //list采用采用模板类实现,对象的默认构造形式：
* `list(beg,end);`                           //构造函数将[beg, end)区间中的元素拷贝给本身。
* `list(n,elem);`                             //构造函数将n个elem拷贝给本身。
* `list(const list &lst);`            //拷贝构造函数。



#### list赋值和交换

**函数原型：**

* `assign(beg, end);`            //将[beg, end)区间中的数据拷贝赋值给本身。
* `assign(n, elem);`              //将n个elem拷贝赋值给本身。
* `list& operator=(const list &lst);`         //重载等号操作符
* `swap(lst);`                         //将lst与本身的元素互换。



#### list 大小操作

**函数原型：**

* `size(); `                             //返回容器中元素的个数

* `empty(); `                           //判断容器是否为空

* `resize(num);`                   //重新指定容器的长度为num，若容器变长，则以默认值填充新位置，如果容器变短，则末尾超出容器长度的元素被删除。

* `resize(num, elem); `       //重新指定容器的长度为num，若容器变长，则以elem值填充新位置，如果容器变短，则末尾超出容器长度的元素被删除。



#### list 插入和删除

**函数原型：**

* push_back(elem);	//在容器尾部加入一个元素
* pop_back();             //删除容器中最后一个元素
* push_front(elem);    //在容器开头插入一个元素
* pop_front();          //从容器开头移除第一个元素
* insert(pos,elem);          //在pos位置插elem元素的拷贝，返回新数据的位置。
* insert(pos,n,elem);         //在pos位置插入n个elem数据，无返回值。
* insert(pos,beg,end);         //在pos位置插入[beg,end)区间的数据，无返回值。
* clear();            //移除容器的所有数据
* erase(beg,end);            //删除[beg,end)区间的数据，返回下一个数据的位置。
* erase(pos);         //删除pos位置的数据，返回下一个数据的位置。
* remove(elem);        //删除容器中所有与elem值匹配的元素。



#### list 数据存取

**函数原型：**

* `front();`        //返回第一个元素。
* `back();`         //返回最后一个元素。



#### list 反转和排序

**函数原型：**

* `reverse();`   //反转链表
* `sort();`        //链表排序，可提供函数或仿函数进行自定义排序



