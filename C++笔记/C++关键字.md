# C++关键字

## static

> ### 类和结构体外的static
>
> 被static修饰后只在内部发生链接，其作用域只在其所在的文件中。
>
> ### 类和结构体内的static
>
> ```C++
> struct Entity
> {
> 	static int x, y;
> 	static void Print()
> 	{
> 		std::cout << x << "," << y << std::endl;
> 	}
> };
> int Entity::x;	//类中的静态变量需单独声明
> int Entity::y;
> ```
>
> + 若static修饰变量，在所有类的实例中，此变量只有一个实例，改变某个实例中的静态变量，所有实例中的此变量都会改变，因为所有实例中的静态变量**指向同一个内存**。
> 	因此改变类中静态变量只需：
>
> 	```c++
> 	Entity::x = 1;
> 	Entity::y = 0;
> 	```
>
> + 若static修饰方法，静态方法不能使用静态变量，静态方法没有类实例。
>
> ### 局部静态
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
> + const修饰指针： 
>
> 	1. 常量指针：**指针指向的内容不可改变。**
> 		``` c++
> 		const int* p = &a;	//const int* p == int const* p
> 		//*p = 8;	//指针指向的内容不可改变
> 		```
>
> 	2. 指针常量：**指针的指向不可改变**。
> 		```c++
> 		int* const p = &a;
> 		//p1 = &c;	//指针的指向不可改变
> 		```
>
> 	3. **指向的内容和指向都不能改变。**
>
> 		```c++
> 		const int* const p = &a;
> 		```
>
> + const修饰方法（只可在类中使用）：**const修饰后表示方法不会修改任何实际的类。**
>
> ```c++
> class Entity
> {
> private:
> 	int m_X, m_Y;
> public:
> 	int GetX()const	//表明该方法不会修改任何实际的类
> 	{
> 		//m_X = 3;	//const修饰后不可改变，mutable修饰的变量除外
> 		return m_X;
> 	}
> };
> ```

## mutable

> + const修饰的方法可修改mutable修饰的变量。
>    ```c++
>     class Entity
>      {
>      private:
>      	std::string m_Name;
>      	mutable int DebugCount = 0;
>      public:
>      	const std::string& GetName()const
>      	{
>      		DebugCount++;
>      		return m_Name;
>      	}
>      };
>      int main()
>      {
>      	const Entity e;
>      	e.GetName();
>      }
>    ```

## new和delete

> + new关键字在堆上创建内存并调用构造函数。
>	```c++
> 	//两者的区别仅是new关键字还会调用构造函数
> 	Entity* e1 = new Entity();
> 	Entity* e2 = (Entity*)malloc(sizeof(Entity));
> 	```
> 
>+ 若使用new[]来分配数组，需使用delete[]。
> 	```c++
>	int* b = new int[10];
> 	delete[] b;
> 	```

## explicit

> ​	取消隐式转换，要求显式的调用构造函数。

## auto

> 编译器自动转换成需要的类型。
