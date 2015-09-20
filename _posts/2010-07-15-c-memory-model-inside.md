---
layout: post
title: "RTTI内存结构分析"
description: ""
category: 
tags: []
---
{% include JB/setup %}
# RTTI内存结构分析 #

    
## 一. 简介 ##
&#160; &#160; &#160; &#160;本文较为深入的研究C++的继承(含多重继承)情况下带虚函数时的实例内存结构，较为深入的剖析了继承实例间是如何组织的，以及动态绑定的实现细节。  
&#160; &#160; &#160; &#160;以下阐述的细节均依据VS2005 生成的DEBUG模式程序在IDA Pro5.2反编译所得，部分数据结构根据程序分析得出。 

## 二. 术语说明 ##
对于以下将频繁用到的术语，我可能简写为如下

classX:RTTI_COL x  *原语*：classX:RTTI Complete Object Locator {for x}  
classX:RTTI_CHD   原语：classX:RTTI class Hierarchy Descriptor  
classX:RTTI_BCD   原语：classX:RTTI Base class Descriptor

以上简写均采用单词首字母缩写，其中Derive表示派生类，x表示Derive的一个基类，classX表示以上任意类


## 三. 分析 ##

#### 1. 派生类内存分布及虚表 ####

![memory model inside](http://linker.hfdn.openstorage.cn/doc/img/blog/2010-07-15-c-memory-model-inside/memory-figure.png)

&#160; &#160; &#160; &#160;&#160; &#160; &#160; &#160;&#160; &#160; &#160; &#160;&#160; &#160; &#160; &#160;&#160; &#160; &#160; &#160;*figure1:派生类实例对象内存分布及虚表映射*

如上图，其中左部为派生类class的实例对象内存格局（继承来自A,B），其中第一项为base A的虚表地址(A,B数据在class实例中布局顺序是按照Derive在声明时继承顺序决定的)，从图中箭头可知base A的虚表地址指向了一个数组，里面存放了Derive对应的虚函数地址：  
① 首先存放重写过的base A声明的的虚函数地址(如果虚函数是从基类继承过来的，那么虚表中存放的也是base A中该函数地址)。  
② 然后存放Derive自己声明的虚函数(可能没有)。
> 
> 注意：vfTable地址指向的虚表是从右边表格黄色以下位置开始，黄色以上是虚表附加信息，不为虚表所有，为本人IDA Pro反汇编分析所得。

其中Derive :RTTI_COL x是定位对象所使用的相关数据结构，其结构经反汇编如下：

#### 2. classX ：RTTI Complete Object Locator {for x} ####

|*Field* | *Length* | *Remark* |
| ------------- | ------------- |-----------|
| reserve_1 | 4 | not used,filled with 0x00 |
| offset_x | 4 | the offset of vfTable( for x) and the base address of classX |
| reserve_3 | 4 | not used,filled with 0x00 |
| pTypeDescriptor | 4 | address of classX:RTTI Type Descriptor |
| pHierarchyDescriptor | 4 | address of classX:RTTI class Hierarchy Descriptor |
     
&#160; &#160; &#160; &#160;&#160; &#160; &#160; &#160;&#160; &#160; &#160; &#160;&#160; &#160; &#160; &#160;*Figure 2:  classX:RTTI Complete Object Locator {for x}反汇编结构*

- **offset_x** 描述基类x在classX实例内存空间中的起始地址相对于classX基址的偏移(也就是x的vfTable字段在classX内存布局中的偏移)  
- **pHierarchyDescriptor** 存放classX：RTTI class Hierarchy Descriptor 结构的地址，该结构描述了Derive类的内存结构层次信息。

> 对于classX为派生类Derive情况时，该结构描述了基类x相对于Derive的信息,此时表格中所有classX均应以Derive替换；但是当classX为基类x情况时，该结构描述了基类自身的信息，此时表格中所有classX均应以x替换.

#### 3. classX:RTTI class Hierarchy Descriptor ####

|*Field* | *Length* | *Remark* |
| ------------- | ------------- |-----------|
| reserve_1 | 4 | unknown |
| reserve_2 | 4 | unknown |
| dwArrayElemCount | 4 | the count of the element inside *pBaseClassArray* |
| pBaseClassArray | 4 | address of *classX:RTTI* Base class Array |

&#160; &#160; &#160; &#160;&#160; &#160; &#160; &#160;&#160; &#160; &#160; &#160;&#160; &#160; &#160; &#160;*Figure 3:  classX:RTTI class Hierarchy Descriptor反汇编结构*

该结构详细描述了类实例中相关重要信息的的索引

- **pBaseClassArray** 存放classX:RTTI Base class Array数组基址。
- **dwArrayElemCount** 存放classX:RTTI Base class Array数组项数,包括自身一项和每个基类一项。

> 对于classX为派生类Derive情况时，该结构描述了派生类Derive的内存结构层次信息,此时表格中所有classX均应以Derive替换；但是当classX为基类x情况时，该结构描述了基类自身的内存结构层次信息，此时表格中所有classX均应以x替换.


#### 4.	classX:RTTI Base class Array ####

此数组存放了该类实例对象所需的所有布局信息。数组中每项均存放了一个类描述符classX:RTTI_BCD的地址，其中实例自身的类描述符地址放在数组第一项，然后存放基类描述符地址

#### 5.	classX:RTTI Base class Descriptor ####

|*Field* | *Length* | *Remark* |
| ------------- | ------------- |-----------|
| pTypeDescriptor | 4 | address of classX:RTTI Type Descriptor |
| dwBaseClassCount | 4 | the count of Base class of classX |
| Offset_X | 4 | the offset of vfTable( for classX) and the base address of Derive |
| reserve_1 | 4 | unknown,filled with 0xffffffff |
| reserve_2 | 4 | unknown,filled with 0x00000000 |
| reserve_3 | 4 | unknown,filled with 0x00000040 |

&#160; &#160; &#160; &#160;&#160; &#160; &#160; &#160;&#160; &#160; &#160; &#160;&#160; &#160; &#160; &#160;*Figure 4:  classX:RTTI Base class Descriptor反汇编结构*

该结构中有几个特别需要注意的地方，首先该结构是从**classX:RTTI class Hierarchy Descriptor –>classX:RTTI Base class Array**引出的，所以此结构中的**Offset_X** 数据都是相对于 **classX:RTTI class Hierarchy Descriptor**中的classX对象而言.  
举例说明，从**Derive：RTTI class Hierarchy Descriptor** 中引出的**x:RTTI Base class Descriptor**中的**Offset_X** 字段是x相对于Derive而言的；但是如果从x:RTTI class Hierarchy Descriptor中引出的x:RTTI Base class Descriptor中的**Offset_X** 字段则是x相对于x而言的

#### 小结 ####
根据上述数据结构，我们可以清晰的将一个类继承关系描述出来，当然只包括虚函数部分，我们甚至可以基于此信息来完成一个PE的类继承视图的扫描器，我们甚至有可能通过对一个基类的扫描而窥知其派生类的继承视图，对，这是有可能的，以上结构信息我相信还有其他用途可供开发研究，再次本人不予赘述

### 补充内容 ###

以下结果是根据IDA PRO5.2分析出的，分析版本为Debug  
&#160; &#160; &#160; &#160;根据逆向分析，可知编译器在虚函数处理过程中完成了不少的工作，当然是编译时工作，根据分析发现虚函数表中函数分布是根据函数声明顺序来存放的，类本身的虚函数置于第一虚表中的末尾位置，这在派生类内存分布及虚表小节中有详细说明，而虚表地址字段又是根据派生类对基类的引用顺序来存放的，那么分析可知，对于某个虚函数而言，无论动态绑定后的结果如何，此虚函数在虚表中的偏移在编译时就能被确定了，那么我们可以认为编译器在对虚函数的绑定技术是基于 虚表基址+偏移地址 取内容来完成的.  

&#160; &#160; &#160; &#160;虚函数的编码问题，根据Debug版本逆向分析可知，对于虚函数中的this引用，编译器采用的是该函数所在虚表地址字段 – 虚表地址字段相对于本实例内存基址的偏移，这样就可得出该实例的内存基址，其中虚表地址字段相对实例基址的偏移在编译时就能确定.  

&#160; &#160; &#160; &#160;thiscall问题，根据被调用的虚函数所在的虚表来确定传入的this指针，虚函数所在虚表在编译时就能确定，此时，将虚表地址字段的地址作为this指针传入(其实也可理解为此虚函数声明所在的类实例的起始地址)。通过传入的this指针，虚函数根据基址相对偏移可以轻松定位到对象的真正基址，从而就能达到通过指针实现动态绑定技术。


附上COL相关结构定义（根据IDA 中数据猜测的结构）：

{% highlight c++ %}
	typedef struct _tagRTTI_Type_Descriptor
	{
		void* pAddressOfProc;
		long reserve_1;
		long signature;   //always 0x56413F2E?
		char szClassName[1];
	}RTTI_Type_Descriptor;

	struct _tagRTTI_Hierarchy_Descriptor;

	typedef struct _tagRTTI_Base_Class_Descriptor
	{
		RTTI_Type_Descriptor* pTypeDescriptor;
		DWORD dwBaseClassCount;
		DWORD dwOffset_x;
		DWORD dwReserve_1;
		DWORD dwReserve_2;
		DWORD dwReserve_3;
		struct _tagRTTI_Hierarchy_Descriptor* pHierarchyDescriptor;
	}RTTI_Base_Class_Descriptor;


	typedef struct _tagRTTI_Hierarchy_Descriptor
	{
		DWORD reserve_1;
		DWORD reserve_2;
		DWORD dwArrayElemCount;
		void* pBaseClassArray;
	}RTTI_Hierarchy_Descriptor;

	typedef struct _tagRTTI_COL_x
	{
		long reserve_1;
		long offsetVTable;
		long reserve_3;
		RTTI_Type_Descriptor* pTypeDescriptor;
		RTTI_Hierarchy_Descriptor* pHierarchyDescriptor;
	}RTTI_COL_X;


{% endhighlight %}


