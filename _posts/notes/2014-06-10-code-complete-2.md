---
layout:	post
title:  《代码大全2》读书笔记
---

《代码大全》是有关编程分割和软件构建的绝好指导书。

# 第一部分 打好基础

## chp01 欢迎进入软件构建的世界

+ 构建活动主要关注于 编码与调试，也部分包含 详设、单测、集成测试。
+ 不同程序员的效率相差20倍。你的效率高么?

## chp02 软件隐喻

+ writing code
+ growing a system
+ system accretion
+ build software

## chp03 前期准备

程序员是软件食物链的最后一环。架构师吃掉需求，设计师吃掉架构，程序员消化设计


## chp04 关键的构建决策

选择代码，定好编程规范

# 第二部分 高质量代码

## chp05 软件构建中的设计
设计不是在谁的头脑中直接跳出来的，它是在不断的设计评估、非正式讨论、写实验代码以及修改试验代码中演化和完善的。

软件本质 复杂，管理复杂度

理想的设计特征

+ 最小复杂度
+ 易于维护
+ 松散耦合
+ 可扩展性
+ 可重用性
+ 搞扇入
+ 低扇出
+ 可移植性
+ 精简性
+ 层次性
+ 标准技术

设计的层次

+ 系统
+ 子系统（业务规则，数据库访问）
+ 类
+ 子程序（类的私有方法）
+ 子程序内部（数据结构，算法，编程）

对状态变量的使用

1. 不要使用布尔，使用枚举
2. 使用access routine来取代对状态变量的直接检查。

设计模式：关于设计启发的总结  P108

Refs：《设计模式》《重构》《怎样解题》（波利亚）

设计实践：
+ 迭代
+ Bottom Up
+ Top Down

## chp06 Working Classes

抽象数据类型：深入挖掘能在问题领域工作的能量吧！

好的抽象：

+ 类的接口需要提供一致的抽象，函数之间最好是有联系的，内聚的。
+ 提供成对的服务。
+ 把不相关的信息转移到其他类中。
+ 尽可能让接口可编程，而不是表达语义。 少用语义假设
+ 不要在修改的时候引入新接口，破坏原有抽象。大杂烩
+ 抽象性和内聚性往往一起出现。

良好的封装:

+ 尽可能限制可访问性
+ 不要公开暴露成员数据
+ 不要将private实现public
+ 不要对使用者做假设
+ 避免使用友元
+ 不要因为一个函数只使用了公共函数，就设为public
+ 让代码阅读起来比编写方便
+ 警惕从语义上破坏封装
+ 当心耦合

包含：

+ 包含才是面向对象编程中的主力技术
+ 数据成员 7+-2

继承：

+ 使用public继承表示 is-a
+ 最好不要用，如果要用，详细的写说明
+ Liskov替换原则
+ 不要覆盖一个非virtual函数，即使你可以那么做，因为那意味着不合理的使用
+ 不要超过3层继承，子类不要超过7个
+ 派生后，将某个子函数覆盖，让他什么都不做。可以考虑让父类使用组合来灵活描述行为。猫和爪子
+ 使用多态。
+ 使用private，而不是protected

成员函数和数据成员：

+ 子程序少
+ 一些默认产生的成员函数可以禁用。像赋值操作符和拷贝构造（TODO Demo）
+ A调用B的方法就行了，别用b.f1().f2().f3()，说明B的封装不好。（Demeter 法则）

构造函数

+ 在所有的构造函数初始化所有的数据成员
+ private 构造函数来实现单例模式
+ 优先考虑 深拷贝

为什么使用类：

应该避免的类：

+ 万能类
+ 没有行为的类
+ 动词命名的类 StringBuilder
     

## chp07 高质量的子程序

创建程序的正当理由：

+ 降低复杂度，提供抽象来简化代码
+ 引入中间、易懂的抽象
+ 避免代码重复
+ 支持子类化
+ 隐藏细节
+ 隐藏指针操作
+ 提高可移植性
+ 简化复杂的布尔判断，封进一个函数中
+ 改善性能（放到一个地方优化）
+ 确保所有的子程序都很小
+ 提高可读性

内聚：

+ 功能上的内聚

好的子程序命名

+ 描述所做的事情
+ 尽量精确，避免模糊的字
+ 长度 9-15字符
+ 动宾结构/面向对象就是动词
+ 对仗词
+ 命名规范。遵守规范，前后统一。

子程序的行数：

+ 50-150行
+ 不要超过200行

如何使用子程序参数：

+ 按照输入-修改-输出的顺序排列参数
+ 若干子程序风格保持一致
+ 使用所有的参数
+ 把状态或者出错变量放在最后
+ 不要改变子程序的输入参数。将其用作工作变量。
+ 参数个数7个以内
+ 输入、修改、输出的参数显示命名。in_xxx, m_xxx , out_xxx。
+ 为接口提供维持其抽象的变量列表或者对象。（有时传对象，有时传若干变量 p179）
+ 检查参数类型，必要时使用Assert。

返回值：

+ 检查所有的返回路径
+ 不要返回局部数据的引用或指针。可以将变化记录在类的数据中，然后使用对应的Getter来访问。

宏的替代方案：

+ const 用户定义常量
+ inline
+ template ，以类型安全的方式定义各种标准操作
+ enum 
+ typedef 用户简单的类型替换

内链

+ 节制使用，因为写在.h中，破坏封装


## chp08 防御式编程

在开车的时候，你要承担起保护自己的责任，哪怕别的司机犯错。

Gabage In

+ 检查外部数据
+ 检查函数输入
+ 处理错误输入数据

断言，断言主要用于开发和维护阶段。

下面是C++语言的支持message的断言宏。

	#define ASSERT (condition, message) {						\
		if (condition) {										\
			LogError("Assertion Failed .", condition, message);	\
		}														\
		exit(EXIT_FAILURE);										\
	}															\
	

错误处理技术

+ 返回中立值
+ 换用下一个正确数据
+ 返回前一次相同的结果
+ 换用最接近的合法值
+ 记录日志
+ 返回一个错误码
	+ 设置状态变量的值
	+ 用状态值作为函数的返回值
	+ 抛出一个异常
+ 调用错误处理子程序或对象
+ 显示出错信息
+ 用最妥当的方法在局部处理
+ 关闭程序

健壮性 vs 正确性

高层设计对错误处理的影响。确定一种通用的错误处理策略，是架构层次的设计决策。

请在每一个系统调用后检查错误码。

异常

> 把异常当做正常处理逻辑的一部分，这样的程序都会遭遇可读性和可维护性的问题（意大利面）。

+ 异常表示通知使用方，发生了不可忽略的错误。
+ 只在真正例外的情况下才使用异常。 异常弱化了封装，提高了复杂度。
+ 不能用异常来推卸责任
+ 避免在构造函数和析构函数中抛出异常（C++）
+ 异常要和对应抛出函数的抽象在一个层次
+ Message要全
+ 避免使用空的catch(){}语句
+ 了解函数库可能会抛出的异常（C++）
+ 集中的异常报告中心（就像CloudAtlas代码中统一处理所有SQLException一样）
+ 整个项目使用异常标准一致
+ 考虑异常的替代方案

辅助调试的代码

+ 使用ant、make等工具
+ 使用预处理器，在make的时候加上-DDEBUG参数，然后在代码里 #if defined (DEBUG) #endif

有多少防御式代码保留在产品中

+ 保留检查重要错误的代码
+ 去掉检查细微错误的代码
+ 去掉可以导致程序硬性崩溃的代码。比如丢失用户数据等是不可取的
+ 记录错误信息，方便追问题。日志！
+ 错误消息是可读的、友好的。


## chp09 伪代码编程过程

创建一个类的步骤：

+ 创建类的总体设计
	+ 定义类的特定职责
	+ 定义类要隐藏的秘密
	+ 精确的定义类的接口所代表的抽象概念。
	+ 关键公共方法
	+ 重要数据成员
+ 创建子程序
+ 复审、测试整个类

创建子程序的步骤：

使用伪代码方法开发子程序，自然变成了注释

检查代码：

+ 脑海里检查程序中的错误
+ 编译子程序。 晚点编译有好处，“只要再编译一次我就可以通过啦！”
	+ 警告级别调到最高
	+ 使用验证工具 lint等
	+ 消除所有错误
+ 逐行执行
+ 测试代码
+ 按照上一章的标准复查代码

其他方法

+ 测试先行开发
+ 契约式设计
+ 重构


# 第三部分 变量

## chp10 使用变量的一般事项

变量使用原则：

+ 在申明变量的时候初始化
+ 在靠近第一次使用变量的地方声明和初始化
+ final(Java)/const(C++)
+ 在类的构造函数中初始化该类的数据成员

减少作用域的原则

+ 在循环开始前初始化循环中的变量，而不是在子程序开始的时候
+ 直到变量快被使用的时候再赋值
+ 把相关语句放在一起
+ 把相关语句提取出成单独的子程序
+ 开始的时候就采用最严格的可见性

变质的变量：

+ delete后置为 NULL，访问的时候判断NOT NULL
+ 加入断言
+ 养成在使用数据前声明和初始化的习惯

为变量指定单一用途：

+ 不要在两个场合使用同一个“临时”变量。（x，tmp）
+ 不要给变量加上隐含意义。（sessionID如果是-1，代表非法，>1 才是正确的）
+ 确保使用了所有已声明的变量

## chp11 变量名的力量

一个好名字通常表达的是“What”，而不是“How”

20个字符以下，足够描述清楚

变量名中的计算值限定词放在最后：

	Min，Max，Total，Average

循环下标： 

+ i k j
+ 如果要在循环之外使用，需要更有意义的名字

状态变量：

+ 不含有flag
+ 枚举类型、具名常量（或者全局变量）

枚举类型：

+ 使用统一前缀来表示这一组变量属于同一个组 Color_RED, Color_GREEN
+ 枚举类型本身 Color这样的驼峰就可以。

全局变量：

+ 全大写

非正式命名规则

+ 区分类和对象 类是首字母大写驼峰
+ 全局变量 g_
+ 成员变量 m_
+ typedef 类型声明 t_
+ 具名常量 c_ C++中是全部大写
+ 枚举值 全大写
+ 子程序 首字母大写

C++ 命名规范示例 P277


## chp12 基本数据类型

数值概论

+ 避免使用“Magic Number”
+ 循环的时候可以使用 1
+ 预防除零错误
+ 使用显示类型转换
+ 注意编译器警告

整数

+ 整数的除法 7/10 结果是0 而不是0.7
+ 检查整数溢出
+ 检查中间结果溢出

浮点数

+ 避免数量级相差太大的两个浮点数加减
+ 避免等量判断
+ 处理舍入误差问题
+ 检查语言和函数库的特定数据类型支持

字符和字符串

+ 避免使用神秘字符和字符串（比如说报警邮件里的字符串，最好可配置。国际化，翻译存放在字符串资源文件中的字符串要比翻译存在于代中的字符串容易的多）
+ 注意 off-by-one
+ 单语言就ISO8859
+ 多语言就Unicode
+ 采用一致的字符串类型转换策略。在靠近I/O的位置做转换

C-String

+ LENGTH +1 
+ \0填充
+ strcpy strncpy strcmp strncmp

布尔

+ 用布尔变量取代一坨&& || 
+ C没有Bool 
	
	enum Boolean {
		True = 1,
		False = (!True)
	}

枚举类型 

+ 提高可读性
+ 提高可靠性，编译器保证
+ 把可能的修改几种在枚举类型中，简化修改
+ 布尔表达不了的，用枚举来


## chp13 不常见的数据类型

使用指针的一般技巧

+ 把指针操作限制在子程序或者类里面，用NextLink(),InsertLink()等封装起来。
+ 同时声明和定义指针。
+ 在分配的作用域中释放指针。
+ 在使用湿疹之前检查指针。
+ 先检查指针所引用的变量再使用它。
+ 可以在内存前后放置狗牌字段。
+ 在删除指针后将其置为NULL

C++ 指针

+ 指针 & 引用
+ 传参时，用指针来实现“传引用”。用cosnt引用来实现“按值传递”。
+ 使用auto_ptr，在离开作用域的时候自动释放内存。
+ 灵活运用智能指针

C 指针

+ 使用显示指针类型而不是默认类型
+ 避免强制类型转换
+ 参数传递要用星号
+ 在分配内存的时候使用sizeof()确定变量大小。

只有万不得已才使用全局数据

+ 首先每一个变量都设为全局的，仅当需要才设为全局
+ 区分全局变量和类变量
+ 使用访问器子程序来封装全局变量（p340）


# 第四部分 语句

## chp14 组织直线型代码

对于必须有明确顺序的语句：

+ 设法组织代码，使依赖关系变明显。（init()）
+ 利用子程序参数明确显示依赖关系。
+ 用注释说明几个语句的顺序关系
+ 检查依赖关系。（断言/错误处理）

对于没有明确顺序的语句：

+ 就近原则
+ 缩短一个变量的“生存”周期，代码块，方便抽成子程序


## chp15 条件语句

利用布尔函数调用简化负责的检测
把最常见的情况放在前面

case语句：

+ 简化每一种条件的处理 如果很长，抽成子函数
+ 如果case面对的条件比较复杂（比如说是字符串而不是int），使用if-else-...
+ default语句用于真正default的，或者用户错误处理
+ 默认都有break，如果要穿过，切记要说明


## chp16 控制循环

## chp17 不常见的控制结构

## chp18 表驱动法

表驱动法是一种编程模式，从表里面查找信息而不使用逻辑语句（if/else）

直接访问表

	int day_of_month (int month){
		int days_array[] = {31,30....};
		return days_array[month-1];
	}



## chp19 一般控制问题

布尔表达式：

+ 使用肯定的表达式
+ 运用表达式的等量替换
+ 不要吝啬括号，能让表达式更清晰
+ 按照数轴的顺序编写数值表达式 `i < MIN_ELEMENTS && MAX_ELEMENTS < i`

与0比较的知道原则：

+ while(balance != 0)
+ while(*charPtr != '\0')
+ while(buffer != NULL)

空语句：

+ 使用DoNothing() 宏，来代替';'

深层嵌套

+ 不要超过3层嵌套

结构化编程的中心论点：任何一种控制流都可以由顺序、选择和迭代这三种结构生成。

McCabe子程序复杂度：

+ 一旦有if while repeat for and or，都加一个复杂度，switch每个case都加1
+ 0-5 还不错。6-10 得想办法简化子程序 10+ 需要重构


# 第五部分 代码改进

## chp20 软件质量概述

Review 是比Test 更有效率的方法。Review一般能直接发现问题以及原因。而Test一般只能发现现象。

软件质量的普遍原理：改善质量以降低开发成本

## chp21 协同构建

协同构建的方式：

+ 结对编程
+ 正式检查
	+ CMM 软件成熟度模型
+ 走查
+ Code Reading



## chp22 开发者测试

单测框架

覆盖率统计

持续集成

## chp23 调试

调试本身并不是改进代码质量的方法，而是诊断代码缺陷的一种方法。

经验丰富的程序员找bug的效率是缺乏经验的20倍。

通过调试让自己进步：

+ 理解程序
+ 明确犯的错误
+ 从代码阅读者的角度分析代码质量
+ 审视自己解决问题的办法
+ 审视自己修正缺陷的方法。

## chp24 重构

## chp25 代码调整策略

TODO

## chp26 代码调整技术

TODO

# 第六部分 系统考虑

## chp27 程序规模对构建的影响

## chp28 管理构建

## chp29 集成

## chp30 编程工具

# 第七部分 软件工艺

## chp31 布局与风格

## chp32 自说明代码


## chp33 个人性格

很多软件开发者花时间去准备应付上一次战争，却不花时间去准备下一次战争。

## chp34 软件工艺的话题

测试仅仅说明软件所用的特定方法有缺陷，并不能让软件更有用、更快、更可读或更有扩展性。

首先为人写程序，其次才是机器。

习惯成自然，专业的程序员总是写可读性好的代码。

借助规范的力量来节省个人的注意力与精力。

基于问题域编程。顶层代码不应该充斥文件、栈、队列、数组、字节等计算机科学的解决方案。分层来解决这个问题。

一天写了一大堆代码，然后花两个星期去调试绝不是巧妙的干活方法。

工具箱：

+ 每种方法都有适用的场景。
+ 工作时自己判断挑选最好的工具。

## chp35 More Info

程序员的书单，每一本都要读！