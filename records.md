# Stage 1 Record

## 2023-10-10

重新开始学习rust语法，边看rust圣经边做rustling。

耗时：2.5h

## 2023-10-12

做了enums、hashmaps、modules、strings部分

耗时2h

## 2023-10-13

error_handling、options

​	

## 2023-10-14   —— 2023-10-16

这几天比较零散，断断续续做了错误处理、泛型、模板、生命周期、特征部分的rustling。这个时候对rust的Option和Result类型有了更深了理解。

Option: 

- 一个可以是空的变量，用来解决其他语言的空指针的问题

- 本质上是一个枚举类型，包含两个成员，一个成员是Some(T)，一个成员是None
- 编译器强制我们需要对其进行模式匹配来处理None值，避免访问空指针的问题，

Result:

- 本质上也是一个枚举类型enum Result<T,E>，成员为Ok(T)，Err(E)。
- 强制我们必须处理函数可能返回的错误，否则会产生pancic

## 2023-10-17 —— 2023-10.19

断断续续完成最后的rustling 大约20道题的部分，主要是迭代器和类型转换部分写了比较久，map、fold、collect使用不够熟练，一开始还是沿用其他语言的思想，写的不够rust   :( 一堆match 真的很丑陋。

