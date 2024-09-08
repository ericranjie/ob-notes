# 

Original 往事敬秋风 深度Linux

 _2024年08月27日 22:34_ _湖南_

vector底层使用动态数组来存储元素对象，同时使用size和capacity记录当前元素的数量和当前动态数组的容量。如果持续的push_back(emplace_back)元素，当size大于capacity时，需要开辟一块更大的动态数组，并把旧动态数组上的元素搬移到当前动态数组，然后销毁旧的动态数组。

![](http://mmbiz.qpic.cn/mmbiz_png/dkX7hzLPUR0Ao40RncDiakbKx1Dy4uJicoqwn5GZ5r7zSMmpwHdJt32o95wdQmPZrBW038j8oRSSQllpnOUDlmUg/300?wx_fmt=png&wxfrom=19)

**深度Linux**

拥有15年项目开发经验及丰富教学经验，曾就职国内知名企业项目经理，部门负责人等职务。研究领域：Windows&Linux平台C/C++后端开发、Linux系统内核等技术。

191篇原创内容

公众号

## 一、vector的底层原理

vector底层是一个动态数组，包含三个迭代器，start和finish之间是已经被使用的空间范围，end_of_storage是整块连续空间包括备用空间的尾部。

std::vector是C++标准库中的一个动态数组容器，它能够存储任意类型的元素，并能够动态改变其大小。当空间不够装下数据（vec.push_back(val)）时，会自动申请另一片更大的空间（1.5倍或者2倍），然后把原来的数据拷贝到新的内存空间，接着释放原来的那片空间【vector内存增长机制】。

当释放或者删除（vec.clear()）里面的数据时，其存储空间不释放，仅仅是清空了里面的数据，因此，对vector的任何操作一旦引起了空间的重新配置，指向原vector的所有迭代器会都失效了。

以下是std::vector底层实现的简化概述：

- `std::vector`动态管理一个数组。
    
- 当数组满时，它会分配一个更大的新数组，并将所有元素复制到新数组中。
    
- 可以通过指针访问数组元素，通常是连续的。
    
- 在尾部添加元素时，如果数组满了，则分配一个更大的数组并复制现有元素。
    
- 在中间插入或删除元素时，会移动元素，因此不适合频繁插入和删除。
    

由于std::vector是一个复杂的数据结构，具体实现可能因编译器而异。但大多数实现会使用动态分配的数组，并在需要时动态重分配和复制。

![Image](https://mmbiz.qpic.cn/mmbiz_jpg/dkX7hzLPUR0nFAKV2NA53qGPALOzShkib9TbQ1pn7bxZDL7qzBic5t7woaWTshPoYx32dNEvfGxicDx5zPVSGOTKw/640?wx_fmt=other&from=appmsg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

其实vector和string的实现非常相似，都是利用了顺序表结构，在vector的实现上我们遵循底层用三个指针来完成，_statr，_finish，_end_fo_storage分别指向顺序表的头，顺序表存储数据的有效个数的位置，顺序表的结束

```cpp
template<class T>class vector{public:	typedef T* iterator;	typedef const T* const_iterator;	private:	iterator _start;	iterator _finish;	iterator _endofstorage;};
```

## 二、初始化vector的函数

### 2.1构造函数

①最常用的无参默认构造vector()

```cpp
vector():_strat(nullptr),_finish(nullptr),_endofstorage(nullptr){}
```

这个很简单，我就不多讲了，后面遇到难点我会详细说明，便于大家理解

②用n个val构造一个vector

```cpp
explicit vector (size_type n, const value_type& val = value_type();
```

库里面用explicit修饰构造函数，是为了防止构造函数发生隐式类型转换

```cpp
vector(size_t n, const T& val = T()){	_start = new T[n];	_finish = _start;	while (_finish!=_start+n)	{		*_finish = val;		_finish++;	}	_endofstorage = _start + n;}
```

### 2.2拷贝构造

> vector(const vector& v);

拷贝构造：用一个已经存在的对象来初始化另一个正在创建的对象

```cpp
vector(const vector& v){	_start = new T[v.size()];	memcpy(_start, v._start, sizeof(T) * v.size());	_finish = _start + v.size();	_endofstorage = _finish;}
```

### 2.3赋值构造

赋值构造：两个已经存在的对象，一个赋值给另一个`vector& operator= (const vector& v);`

```cpp
void swap(vector& v){	std::swap(_start, v._start);	std::swap(_finish, v._finish);	std::swap(_endofstorage, v._endofstorage);}//v1=v2;vector& operator= (vector v){	swap(v);	return *this;}
```

这里我利用库里的函数来实现swap，只交换vector的三个指针更高效，赋值构造的参数是传值传参，会调用拷贝构造，将v2拷贝一份给v，然后之间调用swap函数，将v和this交换，最后返回this即可

### 2.4initializer_list构造

> `vector (initializer_list<T> il);`

tips:这里的initializer_list实际是个类，C++底层将其封装了，里面也有begin，end，size

```cpp
//vector<int> v={1,2,3,4,5};
vector(initializer_list<T> il){	for (auto e : il)	{		push_back(e);	}}
```

### 2.5迭代器区间构造

> `template <class InputIterator> vector(InputIterator first, InputIterator last);`

```cpp
// 类模板的成员函数可以是函数模板
template <class InputIterator>vector(InputIterator first, InputIterator last){	while (first != last)	{		push_back(*first);		++first;	}}
```

注意：如果加了迭代器区间构造会造成一个问题，就是在调用时和`vector(size_t n, const T& val = T())`会出现冲突，底层给出的解决方案就是加一个重载`vector(int n, const T& val = T())`

## 三、vector的核心框架接口的模拟实现

### 3.1vector的迭代器实现

```cpp
Iteratot cend()const {			return final_end;		}		Iteratot cbegin()const {			return start;		}			Iteratot end() {			return final_end;		}		Iteratot begin() {			return start;		}
```

vector的迭代器是一个原生指针,他的迭代器和String相同都是操作指针来遍历数据：

- begin()返回的是vector 容器对象的起始字节位置；
    
- end()返回的是当前最后一个元素的末尾字节；
    

迭代器失效问题

迭代器的主要作用就是让算法能够不用关心底层数据结构，其底层实际就是一个指针，或者是对指针进行了封装，比如：vector的迭代器就是原生态指针T* 。因此迭代器失效，实际就是迭代器底层对应指针所指向的空间被销毁了，而使用一块已经被释放的空间，造成的后果是程序崩溃(即如果继续使用已经失效的迭代器，程序可能会崩溃)。

### 3.2reserve()扩容

```cpp
void reserve(size_t n) {	if (n > capacity()) {			T* temp = new T  [n];			//把statrt中的数据拷贝到temp中
												 size_t size1 = size();			memcpy(temp, start, sizeof(T*) * size());						start = temp;		final_end = start + size1;		finally = start + n;			}		}
```

当 vector 的大小和容量相等（size==capacity）也就是满载时，如果再向其添加元素，那么 vector 就需要扩容。vector 容器扩容的过程需要经历以下 3 步：

- 完全弃用现有的内存空间，重新申请更大的内存空间；
    
- 将旧内存空间中的数据，按原有顺序移动到新的内存空间中；
    
- 最后将旧的内存空间释放。
    

这也就解释了，为什么 vector 容器在进行扩容后，与其相关的指针、引用以及迭代器可能会失效的原因。

由此可见，vector 扩容是非常耗时的。为了降低再次分配内存空间时的成本，每次扩容时 vector 都会申请比用户需求量更多的内存空间（这也就是 vector 容量的由来，即 capacity>=size），以便后期使用。

vector 容器扩容时，不同的编译器申请更多内存空间的量是不同的。以 VS 为例，它会扩容现有容器容量的 50%。

使用memcpy拷贝问题

reserve扩容就是开辟新空间用memcpy将老空间的数据拷贝到新开空间中，假设模拟实现的vector中的reserve接口中，使用memcpy进行的拷贝，以下代码会发生什么问题？

```cpp
int main(){bite::vector<bite::string> v;v.push_back("1111");v.push_back("2222");v.push_back("3333");return 0;}
```

问题分析：

- memcpy是内存的二进制格式拷贝，将一段内存空间中内容原封不动的拷贝到另外一段内存空间中
    
- 如果拷贝的是自定义类型的元素，memcpy即高效又不会出错，但如果拷贝的是自定义类型元素，并且自定义类型元素中涉及到资源管理时，就会出错，因为memcpy的拷贝实际是浅拷贝。
    

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

结论：如果对象中涉及到资源管理时，千万不能使用memcpy进行对象之间的拷贝，因为memcpy是浅拷贝，否则可能会引起内存泄漏甚至程序崩溃。

### 3.3尾插尾删(push_back(),pop_back())

```c
	void push_back(const T&var) {			if (final_end ==finally) {				size_t newcode = capacity() == 0 ? 4 : capacity() * 2;				reserve(newcode);			}			*final_end = var;			++final_end;		void pop_back() {					final_end--;		}
```

插入问题一般先要判断空间是否含有闲置空间，如果没有，就要开辟空间。我们final_end==finally来判断是否含有闲置空间。如果容器含没有空间先开辟4字节空间，当满了后开2_capacoity()空间。在_final_end部插入数据就行了。对final_end加以操作。

### 3.4对insert()插入时迭代器失效刨析

```c
		Iteratot insert(Iteratot iterator,const T&var) {			assert(iterator <= final_end && iterator >= start);			size_t pos = iterator - start;			if (final_end == finally) {								size_t newcode = capacity() == 0 ? 4 : capacity() * 2;				reserve(newcode);				}			//插入操作			auto it = final_end;			while (it >= start+pos) {				*(it+1)=*it;				it--;			}			*iterator = var;			final_end++;						return iterator;		}
```

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

假设这是一段vector空间要在pos插入数据，但是刚刚好final_end和final在同一位置，这个容器满了，要对这这个容器做扩容操作。首先对开辟和这个空间的2呗大小的空间：

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

接着把老空间数据拷贝到新空间中释放老空间。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

由于老空间释放了pos指向的内存不见了，pos指针就成了野指针。这如何解决呢就是在老空间解决之间保存这个指针，接着让他重新指向新空间的原来位置。

而insert()函数返回了这个位置迭代器这为迭代器失效提供了方法，这个方法就是重新赋值。让这个指针重新指向该指向的位置。

### 3.5对erase()数据删除时迭代器失效刨析

```c
	Iteratot erase(Iteratot iterator) {				assert(iterator <= final_end && iterator >= start);				auto it = iterator;				while (it <final_end) {					*it = *(it+1);					it++;				}				final_end--;				return iterator;			}
```

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

vector使用erase删除元素，其返回值指向下一个元素，但是由于vector本身的性质（存在一块连续的内存上），删掉一个元素后，其后的元素都会向前移动，所以此时指向下一个元素的迭代器其实跟刚刚被删除元素的迭代器是一样的。

以下为解决迭代器失效方案：

```c
#include <vector>#include <iostream>using namespace std; int main(){    int a[] = {1, 4, 3, 7, 9, 3, 6, 8, 3, 3, 5, 2, 3, 7};    vector<int> vector_int(a, a + sizeof(a)/sizeof(int));    /*方案一*/    // for(int i = 0; i < vector_int.size(); i++)    // {    //     if(vector_int[i] == 3)    //     {    //         vector_int.erase(vector_int.begin() + i);    //         i--;    //     }    // }  /*方案二*/    // for(vector<int>::iterator itor = vector_int.begin(); itor != vector_int.end(); ++itor)    // {    //     if (*itor == 3)    //     {    //         vector_int.erase(itor);    //         --itor;    //     }     // } /*方案三*/vector<int>::iterator v = vector_int.begin();while(v != vector_int.end()){    if(*v == 3)    {        v = vector_int.erase(v);        cout << *v << endl;    }    else{        v++;    }} /*方案四*/// vector<int>::iterator v = vector_int.begin();// while(v != vector_int.end())// {//     if(*v == 3)//     {//         vector_int.erase(v); //     }//     else{//         v++;//     }// }     for(vector<int>::iterator itor = vector_int.begin(); itor != vector_int.end(); itor++)    {        cout << * itor << "  ";    }    cout << endl;    return 0;}
```

方案一表明vector可以用下标访问元素，显示出其随机访问的强大。并且由于vector的连续性，且for循环中有迭代器的自加，所以在删除一个元素后，迭代器需要减1。

方案二与方案一在迭代器的处理上是类似的，不过对元素的访问采用了迭代器的方法。

方案三与方案四基本一致，只是方案三利用了erase()函数的返回值是指向下一个元素的性质，又由于vector的性质（连续的内存块），所以方案四在erase后并不需要对迭代器做加法。

## 四、例题：杨辉三角

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

在C语言里可以用二维数组进行实现，但C++方式又该如何做呢？

用我们上面说的vector<vector>就行了。

这里的核心问题是：如何去分配空间？像C语言那样静态数组的一口气开完会浪费，开少又不够，vector里的resize可以完美的解决这个问题。

如上图的三角形，第一行有一个，第二行有两个，以此类推，所以，对每一个vector，我们含顺序进行resize，在这里reserve也可以起到开空间的作用，但resize还可以进行初始化，节省很多不必要的操作。

开辟空间完成后，根据杨辉三角的定义，每个数是它左上方和右上方的数的和，进行计算即可。

```
class Solution {public:    vector<vector<int>> generate(int numRows) {        vector<vector<int>> vv(numRows);        for(int i=0;i<numRows;i++)        {            vv[i].resize(i+1,1);        }        for(int i=2;i<numRows;i++)        {            for(int j=1;j<i;j++)            {                vv[i][j]=vv[i-1][j-1]+vv[i-1][j];            }        }        return vv;    }};
```

2023年往期回顾

[C/C++发展方向（强烈推荐！！）](http://mp.weixin.qq.com/s?__biz=Mzg4NDQ0OTI4Ng==&mid=2247487749&idx=1&sn=e57e6f3df526b7ad78313d9428e55b6b&chksm=cfb9586cf8ced17a8c7830e380a45ce080c2b8258e145f5898503a779840a5fcfec3e8f8fa9a&scene=21#wechat_redirect)  

[Linux内核源码分析（强烈推荐收藏！](http://mp.weixin.qq.com/s?__biz=Mzg4NDQ0OTI4Ng==&mid=2247487832&idx=1&sn=bf0468e26f353306c743c4d7523ebb07&chksm=cfb95831f8ced127ca94eb61e6508732576bb150b2fb2047664f8256b3da284d0e53e2f792dc&scene=21#wechat_redirect)[）](http://mp.weixin.qq.com/s?__biz=Mzg4NDQ0OTI4Ng==&mid=2247487832&idx=1&sn=bf0468e26f353306c743c4d7523ebb07&chksm=cfb95831f8ced127ca94eb61e6508732576bb150b2fb2047664f8256b3da284d0e53e2f792dc&scene=21#wechat_redirect)  

[从菜鸟到大师，用Qt编写出惊艳世界应用](http://mp.weixin.qq.com/s?__biz=Mzg4NDQ0OTI4Ng==&mid=2247488117&idx=1&sn=a83d661165a3840fbb23d0e62b5f303a&chksm=cfb95b1cf8ced20ae63206fe25891d9a37ffe76fd695ef55b5506c83aad387d55c4032cb7e4f&scene=21#wechat_redirect)  

[存储全栈开发：构建高效的数据存储系统](http://mp.weixin.qq.com/s?__biz=Mzg4NDQ0OTI4Ng==&mid=2247487696&idx=1&sn=b5ebe830ddb6798ac5bf6db4a8d5d075&chksm=cfb959b9f8ced0af76710c070a6db2677fb359af735e79c6378e82ead570aa1ce5350a146793&scene=21#wechat_redirect)

[突破性能瓶颈：释放DPDK带来无尽潜力](http://mp.weixin.qq.com/s?__biz=Mzg4NDQ0OTI4Ng==&mid=2247488150&idx=1&sn=2cc5ace4391d4eed060fa463fc73dd92&chksm=cfb95bfff8ced2e9589eecd66e7c6e906f28626bef6c1bce3fa19ae2d1404baa25ba3ebfedaa&scene=21#wechat_redirect)

[嵌入式音视频技术：从原理到项目应用](http://mp.weixin.qq.com/s?__biz=Mzg4NDQ0OTI4Ng==&mid=2247488135&idx=1&sn=dcc56a3af5401c338944b9b94f24c699&chksm=cfb95beef8ced2f8231c647033188be3ded88cbacf7d75feab3613f4d7f8fd8e4c8106ac88cf&scene=21#wechat_redirect)

[](http://mp.weixin.qq.com/s?__biz=MzIwNzA1OTA2OQ==&mid=2657215412&idx=1&sn=d3c64ac21056b84814db9903b656853a&chksm=8c8d62a6bbfaebb09df1f727e201c401a1663596a719fa9cb4c7859158e62ebedb5bd5f13b8f&scene=21#wechat_redirect)[C++游戏开发，基于魔兽开源后端框架](http://mp.weixin.qq.com/s?__biz=Mzg4NDQ0OTI4Ng==&mid=2247486920&idx=1&sn=82ca8a72473850e431c01e7c39d8c8c0&chksm=cfb944a1f8cecdb7051863f58d334ff3dea8afae1ecd8bc693287b2f483d2ce8f94eed565253&scene=21#wechat_redirect)

C/C++开发103

面试八股文11

C++标准库1

C/C++开发 · 目录

上一篇大疆一面：请说出水平触发和边缘触发的区别？下一篇大疆常见C++面试题，重点难点全方位解析

Reads 4817

​