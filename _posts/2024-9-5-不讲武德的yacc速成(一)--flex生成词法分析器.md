---
layout: post
title: "不讲武德的yacc速成(一)--flex生成词法分析器的结构"
date:   2024-9-5
tags: [编译技术]
comments: true
author: 久菜合子
---


&emsp;前排提示1：如果发现有什么和你理解的不一样，请狠狠拷打作者
&emsp;前排提示2：本文将**无法**帮你完成icoding编译原理实验
### Step0 flex介绍
&emsp;&emsp;&emsp;众所周知，词法分析是编译器前端的起点。以SysY(可以简单理解为C语言的一个子集)为例, 词法分析将所有制表符和空格(字符串内除外)消除, 将剩余字符划分为被称为token的区域......总之, 经过词法分析器的作用, 源程序向着机器友好的进了一步。
&emsp;&emsp;&emsp;token(词法单元)由一个词法单元名和一个可选属性值组成(龙书第二版p69)。所以token是一个键值对, 大概为\<id, attribute\>, 注意其中attribute值可以为None, 也可以是一个结构化数据。
&emsp;&emsp;&emsp;而关于id, 一般认为源代码中的标识符就可以当作id, 但这引发一个问题, 应该为这个id划分多少空间? gcc中标识符不限制长度, MSVC限制2048个字节, 无论是那一个把标识符字符串直接当作id都有些不可取, 所以取而代之本人认为将字符(串)指针当作id更为合适。并且将指针当作id也该能方便编译器剥除(stripped)符号表, 因为注意到IDA反编译无符号表文件时, 产生的id都是位置相关的, 如sub_40036、UNK_40800之类。
&emsp;&emsp;&emsp;龙书89页介绍了词法分析器生成工具Lex。 而Flex是Lex的升级版，效率更高且bug少，同时Flex兼容Lex的规则。
### Step1 flex在yacc工具链中的角色
&emsp;&emsp;&emsp;首先知道Flex是一个翻译工具，而不是C/C++的第三方库，你无法以简单的预编译include的方式使用Flex。
&emsp;&emsp;&emsp;Flex/Lex源文件以 .l 结尾用于标识。龙书p89介绍了Lex程序的文件流
![](https://www.helloimg.com/i/2024/09/07/66dba6da11312.png)
&emsp;&emsp;&emsp;由框图可知，Lex生成的a.out为一个可执行文件。但在实际使用时，单个可执行文件可能无法完全满足需求。所以为了提升复用性，打开a.out这个黑盒，Flex编译器提供了其他目标文件的选择，可以指定outfile 和 headfile的生成，至于生成的outfile 和 headfile 是什么**语法结构（文件类型）**，这里先按下不表。
&emsp;&emsp;&emsp;
### Step2 lex.l文件格式
&emsp;&emsp;&emsp;lex.l文件分为三个部分，每个部分使用 "%%" 分割，也就是会有两处 "%%" 将整个文本分为三部分
&emsp;&emsp;&emsp;先贴一张示范文件，来自龙书p91
![yacc1-2](https://www.helloimg.com/i/2024/09/07/66dba9d3c0fd5.png)
#### 第一部部分----预编译区域
&emsp;&emsp;&emsp;通过 "%{" "%}" 括起来的区域，也就是图中的
```c
%{
    /* definintions of manifest constants*/
%}
```
&emsp;&emsp;&emsp;这里存放的是供lex.l使用的各种外部定义，相当于一般C程序中的预编译环节，并且语法也是一样的。
&emsp;&emsp;&emsp;"%{"  "%}"之间的内容，将被交给flex去解析和寻找，例如你可以include头文件，define，枚举，定义。
###### 举个例子
```c
%option yylineno
%option caseful
%option noyywrap
%option outfile="scanner.cpp" header-file="scanner.hpp"

%{
    #include <iostream>
    #define ONE 1
    enum example{
        ADD, SUB, MOD
    };
%}

%%
%%
```
```sh
$ flex ./lex.l
$ ls
lex.l  scanner.cpp  scanner.hpp # scanner.cpp scanner.hpp 就是生成的文件
```
###### 针对示例的一点补充
```c
%option yylineno // 启用全局yylineno，记录已经扫描过的行数，是报错提示中相当重要的一部分
%option caseful // 区分大小写，默认打开
%option noyywrap // 取消yywrap()函数，后面讲
%option outfile="scanner.cpp" header-file="scanner.hpp" // 文件
```
&emsp;&emsp;&emsp;%option 是对flex编译的一些设置，这些设置是不必要的，或者其中一部分可以使用命令行时指定。
&emsp;&emsp;&emsp;关于最后的outfile 和 header-file ，看似c++实为c语言，无论是.cpp还是.hpp里，flex**自动生成**的部分中都没有c++的特殊语法。
&emsp;&emsp;&emsp;那有人就要问了，\<iostream\>不是c++的用法吗？你说没有c++用法，我缺的c++这块谁给我补啊？
&emsp;&emsp;**直接上图**
![yacc1-3](https://www.helloimg.com/i/2024/09/07/66dbb18d8253c.png)
&emsp;&emsp;&emsp;从469行到473行就是设置的预编译内容，这一块由于**自动生成的**，而是flex拷贝的。预编译内容flex用不上，但后面我们会用。
&emsp;&emsp;&emsp;该部分的最后一点，也是非常重要的一点，有关于lex.l中怎么使用注释，**先说明上面的注释全是错的**。
&emsp;&emsp;**先上示例**
```c
1 /*  大家好啊    */    
2 // 我是说的编译器     
3       /*  今天来点大家想看的东西   */
```
&emsp;&emsp;&emsp;以上三行注释只有第一行是可行的，也就是只能用/*  */，并且所有的注释都必须顶格开始写。除此之外，"{" "}" 之间的注释与C语言规则相同，因为大括号里的内容会被C的编译器编译。
#### 第二部分----定义token的模式和动作
&emsp;&emsp;&emsp;首先是定义token的模式，这部分一般位于预编译下面以及第一组 "%%" 上面，还是一样先给出示例
###### 示例，不一定都对
```c
/*  basic definitions */
delim [ \t\n]
letter [A-Za-z]
dec_digit [0-9]
oct_digit [0-7]
hex_digit [0-9|a-f|A-F]
/* regular  */
ID ({letter}|(_))({letter}|(_)|{dec_digit})*
WASTE {delim}+
/*  terminal    */
ADD "+"
SUB "-"
T_while "while"
``` 
&emsp;&emsp;&emsp;可以看到，这就是正则表达式的使用，应该不难。pattern之间也可以相互调用。
&emsp;&emsp;&emsp;OK，现在你以及掌握了token模式的定义方式，是时候接受更高层次的挑战了。
###### 🐂刀小试--帮我康康下面的这些定义有没有问题
```c
DEC_INT [1-9][0-9]*
OCT_INT {octprefix}[0-7]*
HEX_INT {hexprefix}[0-9a-fA-F]+

INT {DEC_INT}|{OCT_INT}|{HEX_INT}

DEC_FLOAT_HEAD {DEC_INT}
DEC_FLOAT_MID 0*{DEC_INT}?
DEC_FLOAT_TAIL [eE][+-]?(0*){DEC_INT}
DEC_FLOAT_1 {DEC_FLOAT_HEAD}*{COMMA}{DEC_FLOAT_MID}{DEC_FLOAT_TAIL}* 
DEC_FLOAT_2 {DEC_FLOAT_HEAD}{DEC_FLOAT_TAIL}

HEX_FLOAT_HEAD {HEX_INT}
HEX_FLOAT_MID [0-9a-fA-F]*
HEX_FLOAT_TAIL [pP][+-]?[0-9a-fA-F]
HEX_FLOAT_1 {HEX_FLOAT_HEAD}*{COMMA}{HEX_FLOAT_MID}{HEX_FLOAT_TAIL}*
HEX_FLOAT_2 {HEX_FLOAT_HEAD}{HEX_FLOAT_TAIL}

DEC_FLOAT {DEC_FLOAT_1}|{DEC_FLOAT_2}
HEX_FLOAT {HEX_FLOAT_1}|{HEX_FLOAT_2}

FLOAT {DEC_FLOAT}|{HEX_FLOAT}
```
&emsp;&emsp;&emsp;接下来是token的动作，也就是定义词法分析器匹配到了符合模式的token之后要干什么。这部分在两个 "%%" 之间定义
###### 还是先上示例
```c
%%
{WASTE} {/*什么也不做，就达到了消除空格和制表符的效果，编译技术真是易如反掌啊哈哈（不是*/}
{ID} {yyval = (int)installID(); return(ID);/*这里的ID是一个枚举量，前面的ID是pattern*/}
{ADD} {yyval = ADD; return(ADD);}
%%
```
&emsp;&emsp;&emsp;如你所见，每行由一个token pattern 和一对大括号组成，大括号里就是动作，什么都不做当然也是一种动作。
&emsp;&emsp;&emsp;你肯定很好奇yyval是什么，yyval、yytext、yyleng等许多值，都是词法分析器的全局变量。yyval默认类型为int，用于存储一些关键类型（比如指针或者枚举量），可以使用```#define YYSTYPE newType```来修改它的数据类型。yytext是一个指向词素开头的指针，yyleng存放当前识别到的词素的长度。以上全局变量对于**动作**的设计有一些帮助。

#### 第三部分----设计动作
&emsp;&emsp;&emsp;第二部分中只是在引用动作，而没有具体设计动作。
&emsp;&emsp;&emsp;第三部分在第二个 "%%" 之后，这些 "动作" 一般封装成函数，方便调用。
###### 举例子
```c
%%
int installID(){
    std::cout<<"get an ID "<<endl;
    return ID;
}

int yywrap(){
    return 1;
}
```
&emsp;&emsp;&emsp;这样的设计看起来很简单对吧。没错，所以一般不这样做。
&emsp;&emsp;&emsp;首先，大部分编辑器对.l文件不支持，所以在.l文件里编辑极为痛苦，我们期望尽可能减少编辑.l文件的时间。
&emsp;&emsp;&emsp;其次，编辑动作函数也不是一件容易的事情，思考如何让输出的token流方便后续语法分析和语义分析，会花费大量的时间。
&emsp;&emsp;&emsp;最后，很现实的一点是，yacc的另一个工具bison(用于生成语法分析器)可以实现动作的自动生成。比如已知bison生成了 "parse.hpp" "parse.cpp"(不全是动作的定义)，那么在预编译部分直接```#include```就可以使用这些动作。

#### ---END