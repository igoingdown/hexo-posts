---
style: summer

title: c++入门

tags:
  - c++
  - gcc
  - gdb
  - c
  - vector
---


读《c++ primer》，刷LeetCode，看coolshell就是我的c++自学之路。
为什么是说‘自学’呢？因为学校教的时候我在玩，学校不教了，我才明白
自己啥也不会！考完研，每天按照考研期间的作息，把《c++ primer》通读了好几遍，
以前不懂的东西，竟然也能静下心来思考清楚了，学习真是一件奇妙的事情。
反复看，做习题，尝试刷leetcode的easy题，一开始代码又长又臭，看了不少优美的解法之后，
才明白C++程序应该怎么写，怎么写更漂亮，怎么写能避开C++的坑。
还是顺利过了研究生入学的机试，也顺利找到了第一份实习。
这是第一篇，主要说一下在刷题过程中常用的一些技巧。

<!-- more -->

---

#### 使用浮点数要注意的坑

* 浮点数转换为int
   * 下行转换
	    ```c++
	    float x;
	    int target = int(x);
	    // 普通的强制转换
	    ```

   * 几舍几入
        ```c++
        float x;
        int target = int(x + 0.5);  // 四舍五入
        int target = int (x + 0.7); // 二舍三入
        ```

   * 上行转换
        ```c++
        #include <cmath>
        float x;
        int target = ceil(x);
        ```

* 浮点数由于精度比较低，不能直接用`==`进行相等判别，
而是要以一个非常小的数作为边界来比较

---


#### vector与数组

* 关于`vector`，清华学霸的经验帖写的很棒：

    * [小唯-THU的vector总结](http://www.cnblogs.com/wei-li/archive/2012/06/08/2541576.html)

    **PS**: 这个学霸的python下matplotlib使用总结写的也非常棒！ 

* 数组：

    * 虽然vector确实好用，为了美化代码、简化逻辑，有时候还是会用到数组，如DP相关的算法题。
	数组经常和`memset`方法一起用
	```c++
	int a[10000];
	memset(a, 0, sizeof(a));
	```

---

#### auto关键字

* auto关键字可以让编译器做类型推导，还可以通过加`&`实现引用类型推导，
写过较为复杂的迭代器的朋友一定记得被迭代器类型名称支配的恐惧！好在auto关键字拯救了我们！

    ```c++
    vector<vector<int> > res;
    for (auto vec: res) {
        vec.push_back(1);
    }
    // 这种方式不能达到修改res的目的！
    // 因为vec是复制生成的新的临时变量！

	for (auto &vec: res) {
		vec.push_back(1)
	}
    // 这种方式可以修改res，`auto`关键字的作用是编译器做类型推导，
	// 常用于迭代器遍历数组。上面的例子也能体现auto关键字的强大

	for (auto iter = res.begin(); iter != res.end(); iter++) {
		iter->push_back(1)
	}
    ```

---


#### 使用C的内容

* 使用C标准库
    ```c
    # include <cstdio>
    // 不使用 #include<stdio.h>
    // 前者是后者的模板化版本，也是标准的C++版本。
	// 其他C标准库的命名规则同。
    // 这样就可以使用C的输入输出函数如
    // printf.


    #include <cstdlib>
	// 不使用#inlcude <stdlib.h>.
    // 这样就可使用system("PAUSE") .
    ```

---

#### c++与java中字符串取子串函数的区别

* c++

	```cpp
    string str("hello world");
	// 函数原型: string.substr(start_index, len = end_index)
	s = str.substr(1)  // s: "ello world"
	s = str.substr(0, 5)  // s: "hello"
	```

* java

	```java
	String str = new String("hello world");
	// 函数原型: String.substring(start_index, end_index)
	String s = str.substring(0, 5);
	// s: "hello"
	```

----


#### C++ 标准算法库

* 头文件
	~~~c++
	#include<algorithm>
	~~~

* 排列函数

	```c++
	next_permutation(beg, end)
	next_permutation(beg, end, comp)
	```

	输入两个迭代器，可以指定比较函数。
序列的所有排列可以规定次序。如123在132前，acb在bac前。
调用这一函数时，函数调整输入序列的顺序，生成下一个排列。
如果序列已经是最后一个排列中，则`next_permutation`
将序列重新排列为最低排列并返回`false`；否则，
它将输入序列变换为下一个排列，即字典序的下一个排列，并返回`true`。


* 排序函数
	
	只能用于顺序容器如vector，deque，list，不能用于键值对容器，
这和Python中的`sorted`函数的限制是一致的。

	```c++
	sort(beg, end)
	//实际是快排,平均时间复杂度为N log(N)。

	sort(beg, end, comp)
	// comp是函数或者函数指针，签名是 
	//   bool comp(elem_type first_arg, elem_type second_arg)，
	// 返回的bool值的含义是第一个参数是否应该排在第二个参数的前面。
	// 标准库中有一些函数类的模板，如`greater`和`less`，实际上默认
	// 的comp就是`less`，使用`less`时只需要被排序的迭代器指向的元素
	// 支持`<`操作即可。

	stable_sort (beg,  end)
	//实际是归并排序,在空间足够的情况下时间复杂度为N log(N)。
	stable_sort(beg, end, comp)
	```

* min、max

	```c++
	min_element(beg, end)
	min_element (beg, end, comp)
	max_element (beg, end)
	max_element (beg, end, comp)
	// 可输入数组指针，也可输入迭代器。
	// 输入指针就返回指针，输入迭代器就返回迭代器。
	// 另有类似的min和max函数，只能输入迭代器，不能输入指针。
	```

* 二分搜索函数

	```c++
	lower_bound(beg, end, val)
	lower_bound(beg, end, val, comp)
	// lower_bound得到大于等于val的第一个元素的迭代器

	upper_bound(beg, end, val)
	upper_bound(beg, end, val, comp)
	// upper_bound得到大于val第一个元素的迭代器
	// 输入的数据必须是有序的。
	// 若搜索不到，则返回最后一个元素的下一个元素的迭代器，即end()迭代器。

	vector<int> vec = { 1, 2, 3, 4, 4, 5, 7, 8 };
	auto it = upper_bound(vec.begin(), vec.end(), 4);
	printf("%d", it - vec.begin());
	// 输出为5。
	```

	只有进行字符串匹配时考虑用find进行搜索，其他情况都用`lower_bound`和`upper_bound`进行搜索。

---

#### 数值类型到string类型互换

* 数值类型-\>string类型: 直接`to_string(val)`即可，因为多态

* C风格字符串->数值类型：

	使用`atoi()`，`atof()`, `atol()`, 和`atoll()`等，意思是将ASCII码转换为int,float,long或者long long等类型。
函数原型如下：
	
	```c++
    int atoi (const char * str);
	double atof (const char* str);
	long int atol (const char * str );
	long long int atoll (const char * str);
	```

* string类型->数值类型
	
	```c++
	double stod (const string&  str, size_t* idx = 0);
    float stof (const string&  str, size_t* idx = 0);
	int stoi (const string&  str, size_t* idx = 0, int base = 10);
	long stol (const string&  str, size_t* idx = 0, int base = 10);
	long double stold (const string&  str, size_t* idx = 0);
	long long stoll (const string&  str, size_t* idx = 0, int base = 10);
	unsigned long stoul (const string&  str, size_t* idx = 0, int base = 10);
    unsigned long long stoull (const string&  str, size_t* idx = 0, int base = 10);
	```

	这套函数和`to_string()`函数体系是对应的 ！
	
---

#### vector使用下标访问注意

在使用vector的下标访问vector的元素时，必须先调用`size()`函数确保访问vector时不会发生越界。
也可以使用`at()`代替传统下标访问，`at()`会自动检查数组是否越界！


---


#### char型数据判断类型  

```cpp
isdigit(int c)
// 判断ASCII字符是否为十进制数字

isalpah(int c)
//判断ASCII字符是否为字母

isalnum(int c)
// 判读ASCII是否为字符或十进制数
```

----

#### C和C++风格代码比较：

```c++
int a[] = {1, 2, 3};
while (1)
{
    cout << a[0] << a[1] << a[2] << endl;
    if (!next_permutation(a, a + 3)){
        break;
    }
}

vector<string> vecStr{"abc", "xyz", "aeiou"};
while (1)
{
    cout << vecStr[0] << vecStr[1] << vecStr[2] << endl;
    if (!next_permutation(vecStr.begin(), vecStr.end())) {
        break;
    }

}
```
上述展示的C风格的代码性能上显著优于于C++风格的代码。

---


#### 杂项

有序序列集合运算, 参见《C++ Primer》。
一般般，不是特别好用，得到的结果可能和预期的不一样。

debug模式与release模式:Release模式下，编译器会猜测用户代码的实际意义来改写代码已生成更加高效的汇编代码，可以通过快速启动反汇编观察实际的汇编代码以验证。而debug模式下，编译器会严格按照用户的指示来生成汇编代码，也因此debug模式下运行速度要慢的多

---

#### 书写规范

1.  注意缩进，操作符的操作数之间要留空格（各种留空格，可参考QT的示例代码）
2.  注意段划分，各段之间要空余一行

#### 用指针固定数组基址

访问二维数组`array[100][100]`，实验发现

方法1：

```cpp
for (i = 1 : 100)
  ptr = 行首地址
  for (j = 1 : 100)
    *(ptr++)
```
比方法2：
```cpp
for (i = 1:100)
  for (j = 1:100)
    array[i][j]
```
**快得多**


----

