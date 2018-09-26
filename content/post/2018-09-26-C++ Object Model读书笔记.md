---
title: Inside The C++ Object Model 读书笔记
subtitle: Making a Note
date: 2018-09-26
tags: ["c++", "note"]
---

## 第一章 关于对象

> ### c++在布局相对c的额外负担由virtual引起
> + `virtual function` 机制：
> + `virtual base class`： 用以实现 "多次出现在继承体系中的base class, 有一个单一而被共享的实体"。
> + 多重继承下的额外负担

### C++ 对象模式

> + class 两种data members : `static nonstatic`
> + class 三种functions member : `static nonstatic virtual`

#### 三种模型
##### 简单对象模型
> + `object`是一系列的是`slots` 每个`slot`指向一个`member`
##### 表格驱动对象模型
##### c++对象模型
>
> ADT:抽象数据类型


#### 加上继承

#### 对象的差异
> paradigm：典范
> + 程序模型`procedural model`
> + 抽象数据类型模型 `abstract data type model, ADT`
> + 面对对象模型 `object-oriented model`
##### object大小
> + nonstatic data 总和大小
> + 加上字节对齐的所需要的额外空间
> + 加上为了支持virtual而由内部产生的额外负担

## 第二章 构造函数语意学

>
>implicit    : 隐含
>explicit    : 明确的 (禁止隐式转换)
>trivial     : 没有用的
>nontrivial  : 有用的
>memberwise  : 对每一个member实以...
>bitwise     : 对每一个bit施以...
>semantics   : 语意

> + [inline](https://www.cnblogs.com/fnlingnzb-learner/p/6423917.html)      : 内联 (定义在类中的成员函数缺省都是内联的，如果在类定义时就在类内给出函数定义，那当然最好。如果在类中未给出成员函数定义，而又想内联该函数的话，那在类外要加上inline，否则就认为不是内联的。**关键字inline 必须与函数定义体放在一起才能使函数成为内联**, 仅将inline 放在函数声明前面不起任何作用)

### 默认构造函数
#### 四种情况下如果没有默认构造函数编译器会构建默认构造函数
+ `class member` has default constructor  
+ `base class` has default constructor  
> + `virtual function` 和**vptr**有关?
> + `base class` has `virtual function` 和**vptr**有关?
+ **其他情况下如果没有default constructor也不会被编译器合成出来**

### Copy Constructor 拷贝构造函数
#### 三种情况会调用 Copy Constructor
> + operator `=`
> + 传参的时候
> + 函数返回值

#### 在`bitwise copy semantics`下不会在没有Copy Constructor的情况下构造
#### 编译器构建 Default Copy Constructor 的情型与上面一致



### 程序转化语意`program transformation semantics`

+ 定义是指占用内存的行为
+ **nrv**(name return value)优化 下面有个例子

```
  class test {
      friend  test foo(double);
  public:
  	test() {
  		memset(array, 0, sizeof(array));
  	}
    void test(const test &);
  private:
  	double array[100];
  }
  
  //增加copy constructor 效率会高不少
  inline void test::test(const test &t) {
      memcopy(this->array, t.array, sizeof(this->array));
  }
  
  test foo(double val) {
  	test local;
  	local.array[0] = val;
  	local.array[99] = val;
  	
  	return local;
  }
  
  int main() {
  	for (int count = 0; count < 10000; count ++) {
  		test t = foo(double(32));
  	}
  	return 0;
  }
```
+ 上面例子自己试了下好像没什么效果 - -！

### 成员的初始化队伍
```
    class test {
    public:
        int x;
        int y;
        int z;
    public:
        test(int yy):z(yy), y(yy) {
            x = yy;
        }
    };
```
+ 上述初始化的顺序是 y - > x -> x （ *initialization list 中的初始化顺序是按照类内的申明顺序 但是他的顺序会排在 explicit user code 之前）* 

## 第三章 Data语意学

### Data Member的绑定
### Data Member的布局
### Data Member的存取

#### Static Data Members
#### Nonstatic Data Members

>   Point3d origin, *pt = &origin;
>   origin.x = 0.0;
>   pt->x = 0.0;
>   这两个在一下情况下是不一样的:

+ 当*Point3d*的继承的基类含有虚基类时，并且存取的成员*x* 是从虚基类继承而来的成员，就会有重大的差异。因为这个时候*pt*指向哪一个类型是不确定的，这个操作就必须等到执行期经过间接引导才能够确定。但是如果是origin的话会在编译期间就确定了偏移值。**所以指针调用的效率会低一点**。其它情况下是一致的。


### 继承与Data Member
#### 没有多态的继承
>  继承之间会进行`alignmengt`
```
eg1:    class class1 {
        private:
            int i_;
            char j_;
        }
        sizeof(class1) = 8
        
        class class2 : public class1 {
        private:
            char z_;
        }
        sizeof(class2) = 12
    
eg2:    class class3 {
        private:
            int i_;
            char j_;
            char z_;
        }
        sizeof(class3) = 8
原因:   在派生类的基类之间赋值会导致数据区错误
```

#### 多态的继承 增加空间负担 
+ virtual table
+ virtual ptr
+ constructor
+ destructor

### 对象成员的效率(Object Member Efficiency)
+ **TODO** 关于`cc` 和 `ncc` 效率问题
```
程序员如果关心效率 必须要进行实际测试 可以推论 但仅当参考
```
+ 虚继承慢的原因是因为有些需要跑到执行期间进行某些操作才会找到正确的路径 而没有虚函数的操作在编译期间就就确定了(这句话不是很正确 但大概就是那么个意思，其实不用指针的话也不会产生*多态* 也就还是在编译期间就确定了)


### 指向Data Member 的指针 (Point to Data Members)
```
Point3d::x 表示 Point3d中的偏移地址(会加 1? 这个不同编译器不一样吧 书上的vs c++ 会 我在centos上测试了不会 所以还是要实际测试啊)
float Point3d::*p2 = &Point3d::x;
Point3d::* 的意思是："指向 Point3d data member" 的指针类型
```

+ 指向Member的指针的效率问题
```
float Point3d::*p2 = &Point3d::x;
Point3d ob;
ob.*p2 === ob.x (相当于把指向data memeber的指针重新绑定到class object上去 但是这样效率非常低)
原来c++还有这么多奇奇怪怪的东西
```
+ 类似的继承类不会影响效率 但是会影响优化效率

## 第四章 Function语意学

### Member的各种调用方式

##### Nonstatic Member Functions

##### 名称的特殊处理

#####  Virtual Member Functions（虚拟成员函数） 

+ 明确的调用实体会比较有效率，并因此压制由于虚拟机制而产生的不必要的重复调用操作

```
//明确的调用操作会压制虚拟机制
register float msg = Point3d::magnitude();
```

##### Static Member Functions (静态成员函数)

+ 将会被转换成一般的nonmember 函数调用
+ 不能直接存取class中的nonstatic members  `初始化要比nonstatic members 要早`
+ 不能被声明为 `const`  `volatile` `virtual`
+ 调用的时候不需要 class object

##### Virtual Member Functions（虚拟成员函数）

+ 积极多态 

```
eg : ptr->z() (ptr是个虚基类指针 我姑且这么叫吧)
	 Point3d *p3d = dynamic_cast<Point3d*>(ptr) （RTTI runtime type identifucation 效率很低 不要用）
```

要知道怎么才能调用`z()`实体 我们需要知道

+ `ptr` 所指对象的真是类型
+ `z()` 实体位置

首先在一个多态的class上加两个members

+ 一个字符串或者数字 表示`class`的类型
+ 一个指针 指向表格  _virtual table_


###### virtual functions 是如何被构建的：

```
virtual table 在执行期间是不会有变更的， 所以在编译期间就已经顶好了
```

一个class只会有一个_virtual table_。每个table 内包含了对应的  `class object` 中所有 `active virtual functions` 函数实体的地址 包括以下:

+ class 所定义的函数实体，他会改写一个可能存在的 `base class virtual function`函数实体
+ 继承`base class`的函数实体  在`deruved class`决定不该写`virtual function`时才会出现的。
+ `pure_virtual_called()` 函数实体 既是空间保卫者，也可以作为执行期异常处理函数

```
这里就可以解释为什么虚类的析构函数必须时virtual的了 算是明白原理了
```
##### 多重继承下的virtual Functions(这个有点难懂 trunk 看完后再反回来看看)

###### 主要的困在落在了base2 subobject 身上

[thunk](faculty.stanford.edu/~knuth/taocp.html)  [En](https://en.wikipedia.org/wiki/Thunk)

```
"只是编写可提供地址的代码的一部分：”，P.Z.Ingerman如是说。他是在1961年发明这个定义的。当时是定义这样一个方法：把实际参数绑定到在Algol-60程序里呼叫到的定义。如果一个程序在被呼叫的时候，用一个表达式占据了形参的位置，那么编译器就会产生一个thunk，这个thunk指的是：计算表达式的地址，然后把这个地址放在适宜的位置。
```

##### 虚拟继承下的Virtual Functions

```
建议:不要在virtual bass class 中声明nonstatic data members(会很复杂)
```
