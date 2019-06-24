# 简述golang指针理解



## golang 传值， 按引用指针传递数据理解

按值传递：
    也就是按值传递(by value)。

    内存地址不同 ----> 即互相不影响

按引用指针传递(by reference)：

    new 一个新指针， 但是指向同一个内存地址。


## golang  unsafe 指针使用

golang指针有以下安全限制：

1. Go的指针不能进行数学运算。

2. 不同类型的指针不能相互转换。

3. 不同类型的指针不能使用==或!=比较。

4. 不同类型的指针变量不能相互赋值。

但是通过使用unsafe包， 可以突破以上限制。

转换方式为：
uintptr <-------->unsafe.Pointer(类似c 的void *) <---------> *T(其他类型指针)

uintptr是golang的内置类型，是能存储指针的整型。
uintptr的底层类型是int，它和unsafe.Pointer可相互转换。

而uintptr是用于「指针运算」的，GC 不把 uintptr 当指针，也就是说 uintptr 无法持有对象，uintptr类型的目标会被回收。

uintptr和unsafe.Pointer的区别就是：unsafe.Pointer只是单纯的通用指针类型，用于转换不同类型指针，它不可以参与指针运算


参考文章：https://mp.weixin.qq.com/s/r0gTZR9r5OLUe3zpYInyFg   

