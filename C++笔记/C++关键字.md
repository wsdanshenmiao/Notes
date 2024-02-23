# C++关键字

## static

### 类和结构体外的static

> **被static修饰后==只在内部发生链接==，其作用域只在其所在的文件中。**

### 类和结构体内的static

> ```C++
> struct Entity
> {
> 	static int x, y;	静态成员变量
> 	static void Print()
> 	{
> 		std::cout << x << "," << y << std::endl;
> 	}
> };
> int Entity::x;	//类中的静态变量需单独声明
> int Entity::y;
> ```
>
> **静态成员变量：**
>
> + **若static修饰变量，在所有类的实例中，==此静态成员变量只有一个实例==，改变某个实例中的静态变量，所有实例中的此变量都会改变，因为==所有实例中的静态变量指向同一个内存==。**
>   因此改变类中静态变量只需：
>
>   ```c++
>   Entity::x = 1;
>   Entity::y = 0;
>   ```
>
> + **在编译阶段分配内存。**
>
> + ==**类中的静态变量需在类外单独声明或初始化。**==
>
> **静态成员函数：**
>
> + **若static修饰函数，静态函数只能访问静态变量，静态方法没有类实例。**
>
> 	```C++
> 	static void function（）{}
> 	```
>
### 局部静态

>
> 生命周期：整个程序
>
> 作用域：局部作用域

## const

> + const修饰变量：**变量的值不可改变。**
>
> ```c++
> const int a = 2;
> //a = 10;	//const修饰变量后变量不可改变
> ```
>
> + **const修饰指针： **
>
> 	1. 常量指针：**==指针指向的内容==不可改变。**
> 		``` c++
> 		const int* p = &a;	//const int* p == int const* p
> 		//*p = 8;	//指针指向的内容不可改变
> 		```
>
> 	2. 指针常量：**==指针的指向==不可改变**。
> 		```c++
> 		int* const p = &a;
> 		//p1 = &c;	//指针的指向不可改变
> 		```
>
> 	3. **==指向的内容和指向==都不能改变。**
>
> 		```c++
> 		const int* const p = &a;
> 		```
>
> + **常函数**（只可在类中使用）：**const修饰后表示方法==不会修改任何实际的类==。**
>
> 	+ **常函数内==不可以修改成员属性==。**
>
> 	+ **成员属性声明时加关键字mutable后，在常函数中依然可以修改**
> 		
> 		```c++
> 		class Entity
> 		{
> 		private:
> 			 mutable int m_X； 
> 		    int m_Y;
> 		public:
> 			int GetX()const	//表明该方法不会修改任何实际的类
> 			{
> 				m_X = 3;	//const修饰后不可改变，mutable修饰的变量除外
> 				return m_X;
> 			}
> 		};
> 		```
>
>
> + 常对象：
>
> 	+ **常对象只能调用常函数。**
> 		
> 		```c++
> 		const Entity e;
> 		```

## mutable

> **const修饰的方法可修改mutable修饰的变量。**

## new和delete

### new

> + new关键字在堆上**创建内存**并**调用构造函数**。
>
>   ```c++
>   //两者的区别仅是new关键字还会调用构造函数
>   Entity* e1 = new Entity();
>   Entity* e2 = (Entity*)malloc(sizeof(Entity));
>   ```
>
> + 若使用new[]来分配数组，需使用delete[]。
>   ```c++
>   int* b = new int[10];
>   delete[] b;
>   ```
>
> **placement new:**
>
> `new (address) (type) initializer`
>
> + placement new可以让对象**在已知地址完成构造**。
> 	```C++
> 	m_Data = static_cast<T*>(::operator new(sizeof(T) * size));
> 	new(m_Data + m_Size) T(value);	//在m_Data + m_Size的位置上构造一个T(value)
> 	```
>
> + delete只能删除堆中分配的内存，所以placement new不能使用delete删除内存。
>
> + 可使用析构函数辅助删除。
> 	```c++
> 	m_Data[i].~T();
> 	```

### delete

> + delete关键字在堆上**删除内存**并**调用析构函数**。

## explicit

> **取消隐式转换，要求显式的调用构造函数。**

## auto

> **编译器自动转换成需要的类型。**

## constexpr

> **指示编译器在==编译时计算表达式的值==，并将其视为==常量==，可减小运行时的计算开销，提高程序的性能。**
