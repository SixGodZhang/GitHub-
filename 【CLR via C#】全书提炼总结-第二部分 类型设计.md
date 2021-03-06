# 【CLR via C#】全书提炼总结-第二部分 类型设计

概要:
- [类型基础](#类型基础)
- [基元类型、引用类型和值类型](#基元类型、引用类型和值类型)
- [对象的相等性和同一性](#对象的相等性和同一性)
- [类型和成员基础](#类型和成员基础)
- [友元程序集](#友元程序集)
- [常量和字段](#常量和字段)
- [方法](#方法)
- [参数](#参数)
- [属性](#属性)
- [事件](#事件)
- [泛型](#泛型)
- [接口](#接口)

上一部分程序创建的过程:
![EXELoad](https://github.com/SixGodZhang/Materials/blob/master/Images/EXELoad.png)


## 类型基础
__CLR 在调用方法时加载类型的实例__:
1. CLR 在调用某个方法时,比如Main，第一次会通过JITCompiler将Main方法 编译成本机代码.
2. CLR会注意到Main中所有引用的类型的程序集是否都已经加载，然后根据元数据表，CLR为这些类型创建一个数据结构(如果该类型是第一次加载的话),
3. 创建数据结构的过程: 首先在堆上分配一块内存空间，将该类型的类型对象加载进来,(类型对象包含了类型对象指针，此处指向Type类型对象(所有类型对象都是Type类型的实例)、同步快索引、静态字段、方法的记录项等)
4. 在调用方法时，如果发现有创建类型实例Code，则再为类型的实例对象分配一块内存空间,该空间存储类型对象指针,指向类型对象、同步块索引、实例字段
![clrLoadClass](https://github.com/SixGodZhang/Materials/blob/master/Images/clrLoadClass.png)

## 基元类型、引用类型和值类型
编译器直接支持的数据类型称之为基元类型.
![primitivetype](https://github.com/SixGodZhang/Materials/blob/master/Images/primitivetype.png)
![csharptypes](https://github.com/SixGodZhang/Materials/blob/master/Images/csharptypes.png)
在值类型的使用过程中，需尽量避免装箱的性能消耗。

## 对象的相等性和同一性
相等性是指对象的内容相同,同一性指的是指向同一个对象.
Object中有Equals方法，默认实现的是对象的同一性,因为Object中有专门的方法实现了同一性ReferenceEquals，因此避免使用Object中的Equals方法来判断同一性,
通常,我们选择继承在子类中重写Equals来实现相等性.比如在ValueType中就重写了Equals方法来实现相等性.

在重写Equals方法之后，一定要重写GetHashCode方法,否则编译器将会给出警告.
关于对象的Hash值的用法:(以Dictionary为例)
当集合查找键时，先计算键的hash值，然后根据Hash判断键/值对存在哪个哈希桶中，然后在此哈希桶中以顺序遍历的方式找到该键值对.
在上述过程中，需要注意的就是Hash值的计算方式,它决定了键值对在哈希桶中分布的稠密程度.而Hash值的计算，正和我们上述提到的GetHashCode有关.

## 类型和成员基础
![classmembers](https://github.com/SixGodZhang/Materials/blob/master/Images/classmembers.png)

C# 中 成员的默认访问性
成员名称 | 默认的访问修饰符
--------|--------
class | internal
字段 | private
方法/属性 | private
接口中的方法 | public  abstract 

## 友元程序集
若A程序集想访问B程序集中的声明为internal的类型,即可以将A程序集声明为B程序集的的友元程序集，通过以下Code实现:
``` csharp
[assembly: InternalsVisibleTo("FriendAssemblyDemo_2")]
```
## 常量和字段
常量:常量总是静态成员，而不是实例成员,在定义常量的时候，将会创建元数据.
在代码中引用常量时，将在程序的元数据中查找该符号,提取常量的值，并将其嵌入到IL代码中.因为常量的值是直接嵌入IL代码中的，所以无需为常量奉陪内存.

字段:字段分为类型字段和实例字段.类型字段是在类型对象中分配的.实例字段的内存是在构造类型的实例进行分配的.
字段修饰符：
CLR术语 | C#术语 | 说明
--------|---------|------- 
Static  | static | 表示字段是类型的一部分
Instance | 默认 | 表示字段是实例的一部分
InitOnly | readonly | 只读字段,只能由一个构造器的代码写入

字段使用内联初始化的语法,需要注意代码冗余.

## 方法
__实例构造器和类(引用类型):__
在元数据表中被表示为.ctor
创建引用类型的实例时，首先为实例的数据字段分配内存，然后初始化对象的附加字段(同步块索引和类型对象指针),最后调用类型的实例构造器来设置类型的初始状态.
实例构造器永远不能被继承.
如果类型的修饰符是abstract,则编译器生成的默认构造器的可访问性为protected.
如果基类未提供无参的1构造器，则在子类需要显示调用基类的构造器
如果类的修饰符是static，编译器不会为类生成默认的构造器

构造器的工作原理:
B b = new B();
new :分配内存
1. 先内联初始化一些字段
2. 通过元数据找到父类的构造器，调用其来初始化子类继承的父类实例字段
3. 用默认值初始化类中的实例字段

在同一个类中存在多个构造器时，注意构造器之间的相互依赖关系。
如果不建立依赖关系，在IL代码生成时，可能出现许多冗余代码.

__实例构造器和结构(值类型):__
C# 编译器不允许值类型定义无参构造器.
定义值类型的构造器有两个要求:
1. 在构造器中必须初始化每个字段的值
2. 不能定义无参构造器
__类型构造器:__

__操作符重载:__

__转换操作符方法:__

__扩展方法:__

__分布方法:__
## 参数
## 属性
## 事件
## 泛型
## 接口