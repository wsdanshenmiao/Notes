# C++关键字

## static

### 类和结构体外的static

被static修饰后只在内部发生链接，其作用域只在其所在的文件中。

### 类和结构体内的static

```C++
struct Entity
{
	static int x, y;
	static void Print()
	{
		std::cout << x << "," << y << std::endl;
	}
};
int Entity::x;	//类中的静态变量需单独声明
int Entity::y;
```

+ 若static修饰变量，在所有类的实例中，此变量只有一个实例，改变某个实例中的静态变量，所有实例中的此变量都会改变，因为所有实例中的静态变量**指向同一个内存**。
	因此改变类中静态变量只需：

	```c++
	Entity::x = 1;
	Entity::y = 0;
	```

+ 若static修饰方法，静态方法不能使用静态变量，静态方法没有类实例。

### 局部静态

生命周期：整个程序

作用域：局部作用域
