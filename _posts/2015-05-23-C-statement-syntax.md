---
title: 理解C的声明语法
layout: post
categories: C和C++
---

# 1 声明语法

C语言的声明语法确确实实让很多初学者(比如我)感到迷茫,我们先来看看下面给两个声明:

```c
int *f();   // f: 返回指向int指针的函数

int (*pf)();  // pf: 指向返回int的函数的指针
```

如果直接阅读上面的声明，我们会不会觉得是不是搞反了，如果说(*pf)()是指向函数的指针,那么为什么先使用括弧把星号(指针)扩起来呢?这个问题的答案得从C语言的创造者说起,如果上面的声明用**英语来阅读**就很好理解了，如果从pf开始用英语来阅读上面的声明应该是这样的：

```
pf is a pointer to function returning int
```

翻译成中文，则为

```
pf 为返回int的函数的指针
```

但是C语言中拥有的声明的形式多种多样，例如：

```c
int (*foo)[3]; // foo是指向int的数组(元素个数为3)的指针
int *foo[10];  // foo是指向int的指针的数组(元素个数为10)
```

怎么样才能把花样如此繁多的声明语法掌握呢？使用上面提到的英文阅读法可以帮助我们解读C的声明语法，具体的步骤如下：

1. 首先着眼于标识符（变量名或者函数名）
2. 从距离标识符最近的地方开始，按照优先顺序解释派生类型（指针、数组和函数）。优先顺序如下：
  * 用于整理声明内容的括弧
  * 用于表示数组的[]，用于表示函数的()
  * 用于表示指针的*
3. 解释完成派生类型，使用"of","to","returning"将它们连接起来。
4. 最后,追加数据类型修饰符（在左边，int，double等）
5. 翻译成熟悉的语言，例如中文

数组元素个数或者函数的参数属于类型的一部分。应该将它们作为附属于类型的属性进行解释。

下面距离来说明，例如：

```c
int (*func_p)(double);
```

可以按照下面的顺序来理解：

1 首先着眼于标识符。

```c
fun_p;
```

英语的表达为：

```
func_p is 
```

2 因为存在括号，这里着眼于*。

```c
(*fun_p);
```

英语的表达为：

```
func_p is a pointer
```

3 解释用于函数的(),参数是double。

```c
(*fun_p)(double);
```

英语的表达为：

```
func_p is a pointer to function(double) returning
```

4 最后解释类型修饰符 int。

```c
int (*func_p)(double);
```

英语的表达为：

```
func_p is a pointer to function(double) returning in
```

5 翻译成中文。

```
func_p 是指向返回int的函数（参数为double）的指针。
```

利用上面的方法，基本可以掌握C语言的声明语法了。


下面是一些例子

C语言 | 英语表达 | 中文表达
-------- | -------- | -------
int hoge; | hoge is int | hoge 是int
int hoge[10]; | hoge is array(元素个数10) of int | hoge 是int的数组（元素个数为10）
int hoge[10][3]; | hoge is array(元素个数为10) of array(元素个数为3) of int | hoge int数组（元素个数为3）的数组（元素个数为10）
int *hoge[10]; | hoge is array(元素个数为10) of pointer to int | hoge 是指向int的指针的数组（元素个数为10）
int (*hoge)[10]; | hoge is pointer to array(元素个数为10） of int | hoge是指向int的数组（元素个数为10）的指针
int func(int a); | func is function(参数为int a）returning int | func是返回int的函数（参数是int a）
int (*func)(int a); | func is pointer to function (参数为int a） returning int | func_p 是指向返回int的函数（参数为int a）的指针


# 2 包含const的情况

上面的规则描述中没有提到C中一个非常重要的关键词——const，下面就包含const的情况做一下说明。

**const** 是在ANSI C追加的修饰符，它将类型修饰为“只读”。名不副实的是，**const不一定代表常量**。**const**主要用于修饰函数的参数，将一个常量传递给函数是没有意义的。无论怎样，使用**const**修饰符（变量名），只意味着使其“只读”。

我们可以通过下面的规则解读**const**声明：

1. 遵从前面提到的规则，从标识符开始，使用英语由内向外顺序地解释下去。
2. 一旦解释完毕的部分的左侧出现了**const**，就在当前位置追加read-only。
3. 如果解释完毕的部分的左侧出现了数据类型修饰符，并且其左侧存在**const**，姑且先去掉数据类型修饰符，追加read-only。
4. 在翻译成中文的过程中，我们可以使用这个规则：**const修饰的是紧跟在它后面的单词**。

因此，

```c
char * const src
```

可以解释成：

```
src is read-only pointer to char

src是指向char的只读的指针
```

而

```c
char const *src
```

可以解释成：

```
src is pointer to read-only char

src是指向只读的char的指针
```

此外容易造成混乱的是，

```c
char const *src
```

和

```c
const char *src
```

的**意思完全相同**。






   







