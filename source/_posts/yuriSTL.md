---
title: yuriSTL
avatar: https://cdn.jsdelivr.net/gh/yuri2078/images/img/custom/avatar.jpg
categories: c/c++
comments: true
description: 我的爱宛如孤岛之花，不为人知的绽放,不为人知的凋零！
cover: https://www.loliapi.com/acg/?id=33
date: 2023-01-15 20:40:13
tags: 
	- c/c++
	- STL
feature: true
keywords: 
---

# 常用知识点

## 引用折叠

> 当有一个为左值时 && & 都会折叠成 &, 只有都是 && 才会折叠成&&
> 两个参数指的是 原来的数据类型 和 接受的数据类型
>
> 举例 : 函数接受 T &x。传入 T && 或者 T &时 x 都为 T & 类型
>
> 函数接受 T &&x。传入 T &时 x 为 T & 类型  传入 T && 时， x 为 T &&类型
>
> 但是 x 肯定是 & 左值引用，因为他有名字，有名字就是左值引用

## new 和 delete 

1. 我们用的`new` 例如，`new Person` 是分配具体对象内存，并进行初始化操作 `new []` 则是分配数组
2. 我们常用的`delete` 例如，`delete p` (p 为 new 出来的对象),他会调用类的析构函数，并释放申请的内存。 `delete []` 则是释放数组
3. `new` 和 `delete`，`new []` 和 `delete []` 是一一对应的。
4. `::operator new(size)` 效果和`malloc` 一样，返回指向size字节地址的`void` 指针
5. `::operator delete(ptr)` 效果和`free`一样，释放`ptr`所指向的地址 
6. `::operator new` 和 `::operator delete` 需要一一对应
7. `::operator new` 并不会初始化对象，只是申请内存
8. `::new(ptr) 参数` 叫做`placement new` 用来初始化对象，比如初始化类，调用对应的构造函数

## 模板萃取

> 有时候我们需要判断是不是基础数据类型，此时我们可以使用模板存取的方法来判断

实现方式：

1. 我们定义两个结构体`__true_type` 表示这是基础类型， `__false_type` 表示这不是基础类型。 通过函数重载实现不同处理方法

   ```c++
   struct __true_type {
   };
   
   struct __false_type {
   };
   
   void fun(struct __false_type);
   void fun(struct __true_type);
   ```

2. 了解第一步我们知道 ---- 如果要通过重载实现不同处理，我们需要确定输入的类型是`__true_type` 还是 `__false_type` 那我们每次都要是实例化不同的结构体，很麻烦。但是我们`typedef __true_type __type` `typedef __true_type __type` 后，每次实例化只用`__type struct_name` 就行，然后我们调用`fun(struct_name)` 就行。

   ```c++
   // 原始情况，不同情况我们需要写不同的代码，不能作到统一
   struct __true_type struct_name; // 基础类型
   struct __fasle_type struct_name; // 非基础类型
   fun(struct_name);
   
   //神秘代码的功能就是 typedef 正确类型 __type
     
   struct __type struct_name; // 我们就能写出通用的代码
   fun(struct_name);
   ```

3. 经过第三步我们已经知道要想写出通用代码我们需要将`__type` 重新定向为真确的类型也就是 `__true_type` 还是 `__false_type`. 此时就用到了模板萃取。我们定义一个结构体`is_type` 他将 `__false_type` 定义为 `__type` 。此时我们使用 `__type`就可以定义`__false_type` 了

   ```c++
   template <typename T>
   struct is_type {
   	typedef __false_type __type;
   };
   ```

4. 此时你想不对啊，我们传入的参数不可能一定是非基础类型啊，也有可能是`int` 类型的。那好我们给模板来个特化版本,他将 `__true_type` 定义为 `__type` 此时我们使用 `__type`就可以定义`__true_type` 了，因为特化版本高于普通模板所以当传入的类型为`int` 时会优先调用特化版本

   ```c++
   template <>
   struct is_type<int> {
   	typedef __true_type __type;
   };
   // 结构体名字必须和上面一样，是上面的特化版本 
   ```

5. 此时我们已经能够处理`int` 和别的类型了。

   如果T 是 int 类型 会实例化is_type的特化版本` struct is_type<int> `然后 `typedef __true_type __type`，所以`is_type<T>::__type` 就是 `__true_type` 

   如果T 不是 int 类型 会实例化is_type的普通版本` struct is_type`然后 `typedef __false_type __type`，所以`is_type<T>::__type` 就是 `__false_type` 

   通过上面的操作我们通过模板萃取了`int`类型和别的类型

   ```c++
   typename is_type<T>::__type struct_name;
   fun(struct_name);
   ```

6. 我们扩大范围，将所有基础类型都特化一个版本，这样只有自定义类型(类/结构体) 的 `is_type<T>::__type` 是 `__false_type` 别的都是`__true_type`

7. 完整代码

   ```c++
   // 判断是不是基础类型
   struct __true_type {
   };
   
   /*
       声明两个结构体一个代表是基础类型 一个代表不是基础类型
       定义is_type 结构体 如果类型已经确定 他会生成 后面已经确定类型对应的is_type
       如果类型不是已经确定的 就会生成第一个 通过将 __type 指代不同的结构体
   	在调用函数时 通过不同的参数重载函数 达到不同的效果
   */
   
   struct __false_type {
   };
   
   // class 类型 需要析构
   template <typename T>
   struct is_type {
   	typedef __false_type __type;
   };
   
   template <>
   struct is_type<bool> {
   	typedef __true_type __type;
   };
   
   template <>
   struct is_type<int> {
   	typedef __true_type __type;
   };
   ......省略一堆特化
   ```

8. 使用案例请看 allocator 的 destroy 实现



## 异常退回

> 我写的推出代码我做主！

1. 内存分配失败异常 ---- operator::new 返回空指针
2. 数组访问越界异常 ---- 一共3个元素，你访问 [3]
3. 当前没有元素你却要他返回东西 ---- 容器是空的，但你然我返回元素

------



# 常用函数

## std::move

> 使用方法： `std::move(Tp &&t);` 将传入的参数转化为右值返回
>
> 作用： 当你需要使用移动构造的时候，需要传入右值。用move 函数可以将传入的参数转换成右值返回，这样就可以调用移动构造了
>
> 实现原理： 将传入的参数`val` 去除引用后得到原来的基础类型，然后强制转换为右值返回就行

```c++
template <typename T>
constexpr typename remove_reference<T>::__type&& move(T &&val) noexcept{
	return static_cast<typename remove_reference<T>::__type &&>(val);
}
```

## std::remove_reference 

> 使用方法: `std::remove_reference<模板参数>::type 变量名` 将type 重新定向为传入的模板参数的去引用类型
>
> 作用： 当你需要数据的去引用类型的时候，比如使用`std::move` 的源码。需要强制转为 `&&` 类型，但是如果参数不是`&&` 类型，同`&&`强转后会发生引用折叠，此时就会返回一个`&`类型。所以需要强转为去除引用后，使用去引用的类型 的 `&&`类型
>
> 实现原理： 重载一个结构体，将`T` typedef 为 `__type` ,然后不论他生成的是哪个模板，因为函数重载的原因，`__type` 总是他的去引用数据类型

```c++
// 定义结构体模板
template <typename T>
struct remove_reference {
	typedef T __type;
};

// 左值显式特化版本
template <typename T>
struct remove_reference<T&> {
	typedef T __type;
};

// 右值显式特化版本
template <typename T>
struct remove_reference<T&&> {
	typedef T __type;
};
```



## std::**forward**

> 使用方法: `Person p(std::forward<T> (val));` 将`val`完美转发到 p 的构造参数行列
>
> 作用： 当你在一个函数里面初始化类的时候，你需要将类使用 传入的参数进行移动构造初始化，比如`fun(T &&val)`,你需要用`val` 移动初始化一个类`Person` 此时你使用`Person p(val)`，他会调用拷贝构造函数，因为，val 是左值，所以需要将val转为右值发送，因为有时你需要拷贝构造函数。你可以重载函数进行此向操作，但太麻烦，此时用了完美转发，就可以避免这个问题！他会将你传入的参数，以原来的数据类型转发过去。

```c++
// 完美转发 : 左值转发
template <typename T>
constexpr T&& forward(typename remove_reference<T>::__type& val) noexcept{
	return static_cast<T&&>(val);
}

/*
  有两个是为了类型一一对应 这里使用了 remove_reference 所以一个固定接受左值引用
	一个固定接受右值，当结果是左值的时候，与T&& 折叠，结果仍然是左值
	当结果是右值时，与T&&折叠结果仍然是右值，达到完美转发的效果
	他并不是所有值进来都转换成右值，这和move是不一样的，也不能和move一样！所以需要区别使用
	而不是直接一个函数无脑 返回 T&&
*/

// 完美转发 ： 右值转发
template <typename T>
constexpr T&& forward(typename remove_reference<T>::__type&& val) noexcept
{
	// 静态断言 如果传入的是个左值就报错
	static_assert(!is_lvalue_reference<T>().value, "ni bu neng yong forward jiang yi ge zuo zhi zhuan huan cheng you zhi");
	return static_cast<T&&>(val);
}
```





------



# yuriSTL

> 手搓STL 源码。 目标 -> 一年内，先简单的把他们搓出来

## allocator

> 内存分配器，用来申请内存，并对实例化的对象进行初始化和析构.
>
> 他本身实例化并不进行任何操作

```c++
template <typename T>
class allocator
{
public:
	typedef T value_type; // 基础数据类型
	typedef T* pointer; // 基础数据指针
  
public:
	allocator() = default;
	~allocator() = default;
```

### **allocate** 

> 申请n 字节的空间，并返回对应类型的指针

需求分析：

1. 申请内存
2. 返回对应指针

案例实现：

1. 使用 `operator new` 分配内存
2. 使用`static_cast` 强转

```c++
// 分配size个空间，size_type 就是unsigned 类型
	static pointer allocate(size_type size) noexcept
	{
		if (size == 0) {
			return nullptr;
    }
		return static_cast<pointer>(::operator new(size * sizeof(value_type)));
	}
```

### **deallocate**

> 销毁申请的空间 ---- 直接operator delete 就行

```c++
static void deallocate(pointer ptr, size_type size) noexcept
{
		if (ptr == nullptr) {
			return;
		}
		::operator delete(ptr);
}
```

### **construct**

> 初始化生成的空间 ---- 直接使用 `placement new` 初始化就行. 

```c++
  template <typename... Args>
	static void construct(pointer ptr, Args&&... args) noexcept
	{
		// 构造类的时候可能有多个参数，这些参数可能是左值，可能是右值，所以我们需要完美转发
		// 完美转发需要配合万能引用使用，所以Args 必须是 &&
		::new(ptr) value_type(yuriSTL::forward<Args>(args)...);
	}
```

### **destroy**

> 析构申请的空间 ---- 基础类型不需要调用析构函数，所以为了节省资源，需要判断是不是基础类型。只有不是基础 类型才进行析构函数的调用

简单指针：

```c++
// 简单指针，直接调用析构函数就行
template <typename T>
void destroy(T* ptr) {
	ptr->~T();
}简单指针直接调用析构就行
```

传入迭代器：   ---- 当传入迭代器时，我们需要考虑是不是基础类型，因为基础类型不用析构，所以我们需要节省这部分开销

通过模板萃取判断是不是基础类型 具体可以看上面模板萃取。不多解释直接上代码

```c++
// 是简单类型什么都不用做
template <typename T>
void destroy__(T* start, T* end, __true_type) { }

// 不是简单类型调用析构函数
template <typename T>
void destroy__(T* start, T* end, __false_type)
{
	// 循环调用析构函数
	for (; start != end; start++) {
		start->~T();
	}
}

// 析构n个数据
template <typename T>
void destroy(T* start, T* end)
{
	// 判断是不是基础类型
	typename is_type<T>::__type type;
	// 通过另一个函数完成最终析构
	destroy__(start, end, type);
}
```



## vector

> 向量模板库，类似与数组，他可以随即访问元素，但是只能从后面插入并不能从正面插入。且插入删除数据效率较低。必要时可能还要重新开辟内存
>
> 实现原理： 直接申请管理维护一串连续的空间就行

```c++
template <typename T>
class vector final
{
public:
	typedef T value_type; // 数据类型
	typedef T* iterator; // 指针/迭代器
  typedef T& reference; // 引用

// 成员变量
private:
	value_type* begin_; // 指向一块内存的起始地址
	value_type* end_; // 最后一个元素的下一个位置
	value_type* tail_; // 内存块的最后一块地址
	allocator<T> alloc; // 新建分配内存的工具
```

### vector()

#### 默认构造函数

> 默认构造函数 ---- 啥也不干，申请16 个对象的空间就行，并把尾指针指向相应位置就好

1. 申请空间
2. 异常返回
3. 更新3个指针

```c++
	vector() noexcept
	{
    // 申请空间
		begin_ = alloc.allocate(16); 
    // 异常返回
		if (begin_ == nullptr) {
			yuriSTL::log("内存分配失败捏!"); // 以红色字体终端打印消息
			exit(1); // 并且退出程序，错误代码1
		}
		// 更新指针位置
		end_ = begin_; 
		tail_ = begin_ + 16;
	}
```

#### 拷贝构造函数

> 传入一个vector 对象，用该对象就行拷贝构造

1. 申请一样大小的空间
2. 异常返回
3. 更新指针
4. 使用拷贝构造初始化

```c++
vector(vector<value_type> &v) noexcept
	{
		// 新建一块和他一样大的内存
    begin_ = alloc.allocate(v.max_size());
  	// 异常判断
		if (begin_ == nullptr) {
			yuriSTL::log("内存分配失败捏!");
			exit(1);
		}
		// 更新指针
		const int size = v.size();
		end_ = begin_ + size;
		tail_ = begin_ + v.max_size();
		// 调用构造函数进行构造
		for (int i = 0; i < size; i++) {
			alloc.construct(begin_ + i, *(v.begin() + i));
		}
	}
```

#### 移动构造函数

> 传入一个vector对象，将他的资源移动过来

1. 移动资源
2. 将原来的地址设为`nullptr` 防止重新利用

```c++
// 移动构造函数
	vector(vector<value_type> &&v) noexcept
	{
		// 移动资源
		begin_ = v.begin_;
		end_ = v.end_;
		tail_ = v.tail_;

		// 将原来的地址设置为nullptr
		v.begin_ = nullptr;
		v.end_ = nullptr;
		v.tail_ = nullptr;
	}
```

#### 申请n个元素的空间

> 直接申请n个元素的空间，但是并不需要进行初始化，是申请内存就行

1. 申请空间
2. 异常返回
3. 更新指针

```c++
// 使用n个对象初始化
	explicit vector(const size_type n)
	{
		// 申请空间
		begin_ = alloc.allocate(n);
		// 异常返回
		if (begin_ == nullptr) {
			yuriSTL::log("内存分配失败捏!");
			exit(1);
		}
		// 更新指针
		end_ = begin_;
		tail_ = begin_ + n;
	}
```

#### 初始化n个元素

> 直接申请n个元素的空间，并且进行初始化就行捏

1. 申请空间
2. 异常返回
3. 更新指针
4. 初始化元素

```c++
// 初始化n个元素
	vector(const size_type n, const value_type &val)
	{
		// 申请空间
		begin_ = alloc.allocate(n);
		// 异常返回
		if (begin_ == nullptr) {
			yuriSTL::log("内存分配失败捏!");
			exit(1);
		}
		// 更新指针
		end_ = begin_ + n;
		tail_ = begin_ + n;
		// 初始化元素
		for (int i = 0; i < n; i++) {
			alloc.construct(begin_ + i, val);
		}
	}
```

### ~vector()

> 析构函数 析构掉对象，并释放掉申请的内存，并将她重新赋值为nulllptr就行

```c++
// 析构函数
	~vector() 
	{
		// 调用函数对类进行析构
		alloc.destroy(begin_, end_);
		// 删除掉申请的内存
		if (begin_) {
			alloc.deallocate(begin_);
		}
		// 将他们设置为nullptr 防止被重新利用
		begin_ = nullptr;
		end_ = nullptr;hu shi hua
		tail_ = nullptr;
	}
```

### push_back

> 从末尾插入一个元素，并进行初始化。如果是左值插入，直接传递参数调用构造函数就行，但是如果是右值就需要使用完美转发了。不然都会调用拷贝构造函数

1. 判断剩余空间是否够
2. 插入并初始化
3. 更新指针

```c++
// 从尾部插入元素 左值
	void push_back(const value_type& val)
	{
		// 判断空间是不是满了
		if (end_ == tail_) {
			relloc(); // 如果空间满了则重新分配空间默认 大小 X 2
		}
		// 插入，并更新指针
		alloc.construct(end_++, val);
	}

// 从尾部插入元素 右值
	void push_back(value_type&& val)
	{
		// 判断空间是不是满了
		if (end_ == tail_) {
			relloc();
		}
		// 通过完美转发传递参数
		alloc.construct(end_++, yuriSTL::forward<value_type>(val));
	}
```

### size

>  返回当前元素个数

```c++
// 返回当前元素个数
    const int size() {
        return end_ - begin_;
    }
```

### max_size

> 返回容器最大元素个数

```c++
// 返回最大元素个数
    const int max_size() {
        return tail_ - begin_;
    }
```

### empty

> 判断容器是不是为空

```c++
// 判断容器是否为空
	const bool empty() { 
	  return end_ == begin_; 
	}
```

### front

> 如果容器不为空就返回第一个元素的引用

```c++
// 返回首部元素
	reference front() 
	{
		if (empty()) {
			yuriSTL::log("当前元素为空!");
			exit(3);
		}
		return *begin_;
	}
```

### back

> 如果容器不为空就返回最后一个元素的引用

```c++
// 返回末尾元素
	reference back()
	{
		if (empty()) {
			yuriSTL::log("当前元素为空!");
			exit(3);
		}
		return *(end_ - 1);
	}
```

### begin

> 返回迭代器的起始位置 ---- 因为是向量直接返回申请的地址首地址就行

```c++
iterator begin() noexcept { 
  return begin_; 
}
```

### end

> 返回迭代器的末尾位置 ---- 返回最后一个元素的下一个地址就行

```c++
// 返回末尾迭代器
	iterator end() noexcept { 
	  return end_;
  }
```

### data

> 返回数据的开始地址

```c++
iterator data() noexcept { 
  return begin_; 
}
```

### at

> 返回某个元素的引用，会检查是否越界。

```c++
// 返回对应元素个数
	reference at(const int k)
	{
		if (k >= end_ - begin_) {
			yuriSTL::log("错误！超出内存范围!");
			exit(2);
		}
    return *(begin_ + k);
	}
```

### operator[]

> 返回指定下标的引用

```c++
// 重载[] 返回对应下标元素
	reference operator[](int k)
	{
      if (k >= end_ - begin_) {
			  yuriSTL::log("错误！超出内存范围!");
			  exit(2);
      }
      return *(begin_ + k);
	}
```

### operator=

> 重载等号返回自身的引用

```c++
// 重载等号
	reference operator=(cosnt vector<value_type> &v)
	{
		// 调用函数对之前的数据进行析构
		alloc.destroy(begin_, end_);
		// 删除掉之前申请的内存
		alloc.deallocate(begin_);
		// 重新申请内存
		begin_ = alloc.allocate(v.max_size());
		// 判断是否异常
    if (begin_ == nullptr) {
			yuriSTL::log("内存分配失败捏!");
			exit(1);
		}
		// 更新指针
		end_ = begin_ + v.size();
		tail_ = begin_ + v.max_size();
		// 完成初始化操作
		for (int i = 0; i < v.size(); i++) {
			*(begin_ + i) = *(v.begin_ + i);
		}

		return *this;
	}
```

