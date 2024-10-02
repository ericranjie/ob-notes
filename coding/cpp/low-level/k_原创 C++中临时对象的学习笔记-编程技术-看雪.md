> [https://bbs.kanxue.com/thread-107034.htm](https://bbs.kanxue.com/thread-107034.htm)

在函数调用的时候，无论是参数为对象还是返回一个对象，都将产生一个临时对象。这个笔记就是为了学习这个临时对象的产生过程而写。

本代码的详细例子见实例代码Ex.01

Ok，先让我们定义一个类:

```cpp
class CExample {
public:
	int m_nFirstNum;
	int m_nSecNum;

	int GetSum();
	bool SetNum(int nFirst, int nSec);
	CExample(){} 				// 空构造，不实现任何功能
	virtual ~CExample(){}			// 空析构
};

// 定义的函数实现部分
int CExample::GetSum() {
	return m_nFirstNum+m_nSecNum;
}
```

先让我们看一下对象的创建过程

```nasm
CExample objExp1;
// 	00401393   lea         ecx,[ebp-18h]			//  第一个对象
// 	00401396   call        @ILT+20(CExample::CExample)
// 	0040139B   mov         dword ptr [ebp-4],0		用来统计当前对象个数

CExample objExp2;
// 	004013A2   lea         ecx,[ebp-24h]			//  第二个对象
// 	004013A5   call        @ILT+20(CExample::CExample)
// 	004013AA   mov         byte ptr [ebp-4],1

CExample objExp3;
// 	004013AE   lea         ecx,[ebp-30h]			//  第三个对象
// 	004013B1   call        @ILT+20(CExample::CExample)
// 	004013B6   mov         byte ptr [ebp-4],2
```

上面创建了三个对象，它们的过程都非常相似，将一个局部变量地址给了ECX寄存器，然后调用构造函数。

先让我们看看，构造函数都干啥了:

```nasm
12:   CExample::CExample()
13:   {
00401540   push        ebp
00401541   mov         ebp,esp
00401543   sub         esp,44h
00401546   push        ebx
00401547   push        esi
00401548   push        edi
00401549   push        ecx             				        // 保存寄存器环境
0040154A   lea         edi,[ebp-44h]
0040154D   mov         ecx,11h
00401552   mov         eax,0CCCCCCCCh
00401557   rep stos    dword ptr [edi]
00401559   pop         ecx							         // 填充完CC以后，恢复ECX内容
0040155A   mov         dword ptr [ebp-4],ecx
0040155D   mov         eax,dword ptr [ebp-4]			         // 取到this指针
00401560   mov         dword ptr [eax],offset CExample::`vftable'   // 让this指针指向虚表
15:   }
00401566   mov         eax,dword ptr [ebp-4]
00401569   pop         edi
0040156A   pop         esi
0040156B   pop         ebx
0040156C   mov         esp,ebp
0040156E   pop         ebp
0040156F   ret
```

我们知道，我们再C代码中，实现的是空构造，没有添加任何功能，可是反汇编的时候，发现，函数应该有个参数（是this指针），定位虚表的时候，是又构造完成让this指向虚表的工作的。

1、 传递一个对象的过程:

```cpp
bool SetExpFun(CExample objExp)
{
	g_objExp.SetNum(objExp.m_nFirstNum, objExp.m_nSecNum);
	return true;
}
```

这是我们样例程序中，一个对象作为参数的情况。我们编写如下的调用代码:

```cpp
	SetExpFun(objExp1);
```

反汇编代码如下:

```nasm
004013C8   sub         esp,0Ch					    //	申请临时对象空间
	004013CB   mov        ecx,esp					    //	让ECX指向临时申请的对象
	004013CD   mov        dword ptr [ebp-34h],esp		    //   赋值一份this
	004013D0   lea         eax,[ebp-18h]				    //   获取第一个对象的this指针
	004013D3   push        eax							//   传递参数
	004013D4   call        @ILT+45(CExample::CExample)   //   使用了拷贝构造所以有上面的参数
	004013D9   mov        dword ptr [ebp-48h],eax		    //   产生一个临时对象并保存它的this指针
	004013DC   call        @ILT+15(SetExpFun) (00401014)  //  调用函数
	004013E1   add         esp,0Ch
```

上面代码中，有两处函数调用，一个是我们已经非常熟悉的调用构造函数，另一个事调用我们需要的setExpFun函数，当然，通过上面的注释，我们很容易就能知道，在这里创建了一个临时的对象，而且貌似调用构造函数的时候还传递了一个参数（参数是我们定义的第一个对象: objExp1）。

是的，很明显这里是个拷贝构造，让我们先来看下它的调用过程。

```nasm
拷贝构造
	{
		004011F0   push         ebp
		004011F1   mov         ebp,esp
		004011F3   sub          esp,44h
		004011F6   push         ebx
		004011F7   push         esi
		004011F8   push         edi
		004011F9   push         ecx					; 保存临时对象的this指针
		004011FA   lea          edi,[ebp-44h]
		004011FD   mov         ecx,11h
		00401202   mov         eax,0CCCCCCCCh
		00401207   rep stos       dword ptr [edi]
		00401209   pop          ecx					; 找到调用时传递的临时对象的this指针
		0040120A   mov         dword ptr [ebp-4],ecx
		0040120D   mov         eax,dword ptr [ebp-4]
		00401210   mov  		ecx,dword ptr [ebp+8]	; 参数对象的this指针，ECX中是虚表
		00401213   mov         edx,dword ptr [ecx+4]	; 取出参数对象的第一个成员
		00401216   mov         dword ptr [eax+4],edx	; 并赋值给临时对象的第一个成员
		00401219   mov         eax,dword ptr [ebp-4]
		0040121C   mov         ecx,dword ptr [ebp+8]
		0040121F   mov         edx,dword ptr [ecx+8]	; 取到参数对象的第二个成员
		00401222   mov         dword ptr [eax+8],edx	; 并赋值给临时对象的第二个成员
		00401225   mov         eax,dword ptr [ebp-4]	; 设置临时对象的虚表
		00401228   mov         dword ptr [eax],offset CExample::`vftable'
		0040122E   mov         eax,dword ptr [ebp-4]	; 返回一个临时对象
		00401231   pop          edi
		00401232   pop          esi
		00401233   pop          ebx
		00401234   mov         esp,ebp
		00401236   pop          ebp
		00401237   ret           4
	}
```

从上面的代码不难看出，我们这个拷贝构造直接在参数中改写的数据，等出来这个函数，我们main函数中：

```nasm
004013C8   sub         esp,0Ch
```

申请的临时对象空间中就是一个完整的对象了。

好现在我们继续跟踪调用传参的代码:

```nasm
	16:   bool SetExpFun(CExample objExp)
	17:   {
			004012C0   push        ebp
			004012C1   mov        ebp,esp
			004012C3   push        0FFh
			004012C5   push        offset __ehhandler$?SetExpFun@@YA_NVCExample@@@Z
			004012CA   mov        eax,fs:[00000000]
			004012D0   push        eax
			004012D1   mov        dword ptr fs:[0],esp
			004012D8   sub         esp,44h
			004012DB   push        ebx
			004012DC   push        esi
			004012DD   push        edi
			004012DE   lea          edi,[ebp-50h]
			004012E1   mov         ecx,11h
			004012E6   mov         eax,0CCCCCCCCh
			004012EB   rep stos      dword ptr [edi]
			004012ED   mov        dword ptr [ebp-4],0				; 计数对象数量
			18:       g_objExp.SetNum(objExp.m_nFirstNum, objExp.m_nSecNum);
			004012F4   mov         eax,dword ptr [ebp+0Ch]			; 直接引用临时对象的成员
			004012F7   push         eax
			004012F8   mov         ecx,dword ptr [ebp+10h]
			004012FB   push         ecx
			004012FC   mov         ecx,offset g_objExp			; 传递this指针
			00401301   call          @ILT+0(CExample::SetNum)
			19:       return true;
			00401306   mov         byte ptr [ebp-10h],1
			0040130A   mov         dword ptr [ebp-4],0FFFFFFFFh	; 清空临时对象计数
			00401311   lea          ecx,[ebp+8]					; 取到临时对象的this指针
			00401314   call         @ILT+40(CExample::~CExample)
			00401319   mov         al,byte ptr [ebp-10h]
	20:   }
	0040131C   mov         ecx,dword ptr [ebp-0Ch]
	0040131F   mov         dword ptr fs:[0],ecx
	00401326   pop         edi
	00401327   pop         esi
	00401328   pop         ebx
	00401329   add         esp,50h
	0040132C   cmp         ebp,esp
	0040132E   call        __chkesp (00401610)
	00401333   mov         esp,ebp
	00401335   pop         ebp
	00401336   ret
```

2、 返回一个对象的过程:

```cpp
CExample GetExpFun()
{
	return g_objExp;
}
```

编写如下的调用代码:

```cpp
	// 下面是返回对象的情况
	objExp2 = GetExpFun();
```

调试下这个程序:

```nasm
59:       objExp2 = GetExpFun();
	004013E4   lea         ecx,[ebp-40h]		; 返回的临时对象空间是进入main函数的时候，提前分配好的。
	004013E7   push        ecx					;		先将对象压栈
	004013E8   call        @ILT+25(GetExpFun)	;		调用函数
		11:   CExample GetExpFun()
		12:   {
		00401190   push        ebp
		00401191   mov         ebp,esp
		00401193   sub         esp,44h
		00401196   push        ebx
		00401197   push        esi
		00401198   push        edi
		00401199   lea         edi,[ebp-44h]
		0040119C   mov         ecx,11h
		004011A1   mov         eax,0CCCCCCCCh
		004011A6   rep stos    dword ptr [edi]
		004011A8   mov         dword ptr [ebp-4],0
		13:       return g_objExp;
		004011AF   push        offset g_objExp (0042af80)
		004011B4   mov         ecx,dword ptr [ebp+8]			; 引用传进来的参数对象指针
		004011B7   call        @ILT+45(CExample::CExample)	; 调用构造创建对象
		004011BC   mov         eax,dword ptr [ebp-4]
		004011BF   or          al,1
		004011C1   mov         dword ptr [ebp-4],eax			; 更新对象个数
		004011C4   mov         eax,dword ptr [ebp+8]			; 返回……
		14:   }
		004011C7   pop         edi
		004011C8   pop         esi
		004011C9   pop         ebx
		004011CA   add         esp,44h
		004011CD   cmp         ebp,esp
		004011CF   call        __chkesp
		004011D4   mov         esp,ebp
		004011D6   pop         ebp
		004011D7   ret

	004013ED   add         esp,4
	004013F0   mov         dword ptr [ebp-4Ch],eax			; 保存临时对象的指针
	004013F3   mov         edx,dword ptr [ebp-4Ch]
	004013F6   mov         dword ptr [ebp-50h],edx
	004013F9   mov         byte ptr [ebp-4],3
	004013FD   mov         eax,dword ptr [ebp-50h]		; 这里重载的 = 运算符，因此将副本压栈做复制操作
	00401400   push        eax
	00401401   lea         ecx,[ebp-24h]					; 得到第二个对象的this指针
	00401404   call        @ILT+10(CExample::operator=)
	00401409   mov         byte ptr [ebp-4],2
	0040140D   lea         ecx,[ebp-40h]					; 使用完成，释放临时对象
	00401410   call        @ILT+40(CExample::~CExample)

	printf("%d\\r\\n", objExp2.GetSum());
```

OK,只要捣鼓明白了这个临时对象，那我们的好多问题都可以解决了。