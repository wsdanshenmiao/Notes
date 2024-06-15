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
>生命周期：整个程序
>
>作用域：局部作用域



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
>
> ![](https://img2024.cnblogs.com/blog/3406761/202405/3406761-20240526104231381-1456994188.png)
>
> 当成员函数的 const 和  non-const 版本同时存在时，**const 对象只会调用 const 版本**，**non-const 对象只会调用 non-const 版本**。



## mutable

> **const修饰的方法可修改mutable修饰的变量。**

## 动态内存new和delete

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



## 异常处理throw，try和catch

- **throw:** 当问题出现时，程序会抛出一个异常。这是通过使用 **throw** 关键字来完成的。
- **catch:** 在您想要处理问题的地方，通过异常处理程序捕获异常。**catch** 关键字用于捕获异常。
- **try:** **try** 块中的代码标识将被激活的特定异常。它后面通常跟着一个或多个 catch 块。

语法：
```c++
try{
   // 保护代码
}
catch( ExceptionName e1 ){
   // catch 块
}
catch( ExceptionName eN ){
   // catch 块
}
```

如果 **try** 块在不同的情境下会抛出不同的异常，这个时候可以尝试罗列多个 **catch** 语句，用于捕获不同类型的异常。

### 抛出异常

throw 语句的操作数可以是任意的表达式，表达式的结果的类型决定了抛出的异常的类型。如：

```c++
throw "Division by zero condition!";	//抛出const char*类型的异常
```

```c++
throw std::out_of_range("out of range");	//抛出std::out_of_range类型的异常
```

### 捕获异常

**catch** 块跟在 **try** 块后面，用于捕获异常。可以指定想要捕捉的异常类型，这是由 catch 关键字后的括号内的异常声明决定的。

```c++
try{
   // 保护代码
}
catch( ExceptionName e ){
  // 处理 ExceptionName 异常的代码
}
catch( ... )	//若想处理任何种类的异常，可使用(...)
```

### C++ 标准的异常

C++ 提供了一系列标准的异常，定义在 **<exception>** 中，我们可以在程序中使用这些标准的异常。它们是以父子类层次结构组织起来的，如下所示：
![C++ 异常的层次结构](https://www.runoob.com/wp-content/uploads/2015/05/exceptions_in_cpp.png)

下表是对上面层次结构中出现的每个异常的说明：

| 异常                   | 描述                                                         |
| :--------------------- | :----------------------------------------------------------- |
| **std::exception**     | 该异常是所有标准 C++ 异常的父类。                            |
| std::bad_alloc         | 该异常可以通过 **new** 抛出。                                |
| std::bad_cast          | 该异常可以通过 **dynamic_cast** 抛出。                       |
| std::bad_typeid        | 该异常可以通过 **typeid** 抛出。                             |
| std::bad_exception     | 这在处理 C++ 程序中无法预期的异常时非常有用。                |
| **std::logic_error**   | 理论上可以通过读取代码来检测到的异常。                       |
| std::domain_error      | 当使用了一个无效的数学域时，会抛出该异常。                   |
| std::invalid_argument  | 当使用了无效的参数时，会抛出该异常。                         |
| std::length_error      | 当创建了太长的 std::string 时，会抛出该异常。                |
| std::out_of_range      | 该异常可以通过方法抛出，例如 std::vector 和 std::bitset<>::operator[]()。 |
| **std::runtime_error** | 理论上不可以通过读取代码来检测到的异常。                     |
| std::overflow_error    | 当发生数学上溢时，会抛出该异常。                             |
| std::range_error       | 当尝试存储超出范围的值时，会抛出该异常。                     |
| std::underflow_error   | 当发生数学下溢时，会抛出该异常。                             |

除了使用C++标准异常，还可通过**继承和重载std::exception类**来自定义异常。
