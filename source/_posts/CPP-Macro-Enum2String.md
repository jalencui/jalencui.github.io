---
title: MSVC Compiler Prefined Macro -- 枚举(enum class)转字符串
author: Jalen Cui
comments: true
categories: CPP
date: 2023-01-23 04:26:46
tags:
---

# 预备知识  

<!--
> block reference used head of paragragh, and can be nested.
-->
* Standard predefined identifier: 预定义标识符  
    + `__func__`: 返回所在函数闭包对应的函数名. 本质上是常量字符数组, 由编译器提供(ISO C99 & ISO C++11), 格式: `const char __func__[]`
* Standard predefined macro: 预定义宏
> MSVC supports the predefined preprocessor macros required by the ANSI/ISO C99, C11, and C17 standards, and the ISO C++14, C++17, and C++20 standards. 

    + 可提供通用的辅助性信息
        - `__LINE__`: 提供当前函数所在的行号, 格式: 数字
        - `__FILE__`: 当前函数所在的文件, 格式: 字符串
        - `__DATE__`: 当前日期, 格式: `Jan 19 2023`
        - `__TIME__`: 当前时间信息, 格式: `22:35:41`
    + 提供具体到函数调用的信息本文将利用 `__FUNCSIG__` 提供的信息, 模拟反射机制来将 enum 变量转为字符串 
        - `__FUNCTION__`: 提供所在闭包对应函数的函数名, 格式: 返回 `__func__` 的值
        - `__FUNCDNAME__`: 提供函数修饰名, 格式: `?main@@YAHXZ` 或 `main` 或 `_main`（可理解为符号表中导出的符号）
        - `__FUNCSIG__`: 提供函数的完整声明(同g++支持的宏:`__PRETTY_FUNCTION__`), 格式: `int __cdecl main(void)`
* 编译方式: 导出符号的修饰规则, 会影响到部分预定义宏的返回结果
    + `extern "C"`: 不带修饰符. 此时, `__FUNCTION__` 与 `__FUNCDNAME__` 的值相同 
    + 其他: 导出符号被修饰, 如: `?main@@YAHXZ`, 可拆分为 `?` + `main` + `@@` + `参数表代号`
        - 修饰符: 用于函数重载
        - 代号: XDEFHIJKMN_NU, 比如: @@YAHXZ可拆分为
            - @@YA:起始位置, 也有@@YG, @@YI
            - H: 返回类型为 `int`
            - X: 参数列表为空, 即 `void`
            - Z: 结束位置, 也有@Z
* 调用约定:   
    + `_cdecl`: C Declaration 的缩写, 即采用 C 语言的声明方式（C缺省调用方式）  
        - 参数从右--->左依次入栈
        - 手动清栈, 即由调用者来恢复堆栈
    + `_stdcall`: Standard Call 的缩写, 是 C++ 的标准调用方式. MSVC 宏有 `PASCAL`, `WINAPI`, `CALLBACK` 
        - 参数从右--->左依次入栈
        - 自动清栈, 即由被调用函数来恢复堆栈
    + `_fastcall`
        - 参数借由CPU寄存器(ecx, edx)和堆栈来处理参数信息, 从右--->左依次入栈
        - 自动清栈: 同`_stdcall`, 减轻调用者负担
# 简介  
业务开发过程中, 会遇到需要获取 <krd>Enum</krd> 的相关信息, 具体应用场景如下: 
* 配置信息: 枚举值本身在代码逻辑中用于归类; 枚举名在初始化阶段作为配置信息的索引
* 多语言: 完整的枚举名列表用于前端UI展示; 枚举值用于用户交互进行语言切换  

本文目标是模拟反射机制(c#, java), 获取对应枚举的相关信息. 由预备知识可知, `__FUNCSIG__` 可获取当前函数的完整声明信息(包括调用约定信息). 不由产生两个疑问: 
* `__FUNCSIG__` 是如何获得函数的完整声明?
* 函数的声明如何能获得任意类型的 `enum class` 的相关信息?  

好的问题远比答案重要: 在编译阶段, 模板函数会对所有的枚举类型进行实例化. 而实例化过程中, 传入的枚举类型及枚举值就是我们所需要的相关信息. 本文目标是提供操作枚举类型的工具函数, 如下:
* `get_enum_name`: 通过枚举值, 获取对应名称
* `get_all_enum_names`: 获取枚举类型中所有值对应的名称列表
* ~~`get_enum_value`: 通过名称, 获取对应枚举值~~
* ~~`check_enum_valid`: 确认是否是有效的枚举(value or name)~~
# 准备阶段
考虑到兼顾不同编译器的情况, 需要对不同编译器提供的预定义宏进行适配. 个人比较喜欢 `__PRETTY_FUNCTION__`, 如下:   
* > [msvc](https://learn.microsoft.com/en-us/cpp/preprocessor/predefined-macros?view=msvc-170): `__FUNCSIG__` Defined as a string literal that contains the signature of the enclosing function.
* > [gcc](https://gcc.gnu.org/onlinedocs/gcc/Function-Names.html): In C, `__PRETTY_FUNCTION__` is yet another name for `__func__`, except that at file scope (or, in C++, namespace scope), it evaluates to the string "top level". In addition, in C++, `__PRETTY_FUNCTION__` contains the signature of the function as well as its bare name.  

<table style="margin-bottom: 8px;">
    <tbody>
        <tr style="border: none">
            <td style="background: black; border: none; width: 100vw; border-radius: 4px">
                <ol style="margin:0;">
                    <li align="left"><b>
                        <font size=2 color=silver>#if&ensp;!defined(</font><font size=2 color=peru>__PRETTY_FUNCTION__</font><font size=2 color=silver>)&ensp;&&&ensp;!defined(</font><font size=2 color=gainsboro>__GNUC__</font><font size=2 color=silver>)</font>
                    </b></li>
                    <li align="left"><b>
                        <font size=2 color=silver>#&emsp;&emsp;define</font>
                        <font size=2 color=mediumpurple>__PRETTY_FUNCTION__</font>
                        <font size=2 color=peru>__FUNCSIG__</font>
                    </b></li>
                    <li align="left"><b>
                        <font size=2 color=silver>#endif</font>
                    </b></li>
                </ol>
            </td>
        </tr>
    </tbody>
</table>  

同时, 提供工具函数用于遍历枚举类的所有枚举值: 根据阈值 `boundary`, 生成枚举值所在范围列表 [-boundary, boundary)  

<table style="margin-top: 8px">
    <tbody>
        <tr style="border: none">
            <td style="background: black; border: none; width: 100vw; border-radius: 4px">
                <ol style="margin:0;">
                        <li align="left"><b>
                        <font size=2 color=steelblue>template</font><font size=2 color=silver>&ensp;<</font><font size=2 color=steelblue>int</font><font size=2 color=silver>...</font><font size=2 color=gainsboro>&ensp;sequence</font><font size=2 color=silver>></font>
                    </b></li>
                    <li align="left"><b>
                        <font size=2 color=steelblue>constexpr&ensp;auto</font><font size=2 color=lemonchiffon>&ensp;make_sequence_impl</font><font size=2 color=silver>(std::</font><font size=2 color=mediumseagreen>index_sequence</font><font size=2 color=silver><</font><font size=2 color=gainsboro>sequence</font><font size=2 color=silver>...>&ensp;seq)&ensp;{</font>
                    </b></li>
                    <li><b>
                        <font size=2 color=mediumorchid>&emsp;&emsp;&ensp;return</font><font size=2 color=silver>&ensp;std::</font><font size=2 color=mediumseagreen>integer_sequence</font><font size=2 color=silver><</font><font size=2 color=steelblue>int</font><font size=2 color=silver>,&ensp;(</font><font size=2 color=gainsboro>sequence&ensp;-</font><font size=2 color=silver>&ensp;seq.</font><font size=2 color=gainsboro>size</font><font size=2 color=silver>())...,</font><font size=2 color=gainsboro>&ensp;sequence</font><font size=2 color=silver>...>();</font>
                    </b></li>
                    <li align="left"><b>
                        <font size=2 color=silver>}</font>
                    </b></li>                       
                    <li align="left"><b>
                        <font size=2 color=steelblue>template</font><font size=2 color=silver>&ensp;<</font><font size=2 color=steelblue>int</font><font size=2 color=gainsboro>&ensp;boundary</font><font size=2 color=silver>,</font><font size=2 color=steelblue>&ensp;typename</font><font size=2 color=mediumseagreen>&ensp;indices</font><font size=2 color=silver>&ensp;=&ensp;std::</font><font size=2 color=mediumseagreen>make_index_sequence</font><font size=2 color=silver><</font><font size=2 color=gainsboro>boundary</font><font size=2 color=silver>>></font>
                    </b></li>
                    <li align="left"><b>
                        <font size=2 color=steelblue>constexpr&ensp;auto</font><font size=2 color=lemonchiffon>&ensp;make_sequence</font><font size=2 color=silver>()&ensp;{</font>
                    </b></li>
                    <li align="left"><b>
                        <font size=2 color=mediumorchid>&emsp;&emsp;&ensp;return</font><font size=2 color=gainsboro>&ensp;make_sequence_impl</font><font size=2 color=silver>(</font><font size=2 color=mediumseagreen>indices</font><font size=2 color=silver>());</font>
                    </b></li>
                    <li align="left"><b>
                        <font size=2 color=silver>}</font>
                    </b></li>
                </ol>
            </td>
        </tr>
    </tbody>
</table>  

# 代码实现
* `get_enum_name`: 由于 `std::regex` 不支持 look behind, 此处藉由 `QRegualr` 实现对编译期常量<b><font color=gainsboro>`V`</font></b>和类型<b><font color=mediumseagreen>`E`</font></b>的捕获.

<table>
    <tbody>
        <tr style="border: none">
            <td style="background: black; border: none; width: 100vw; border-radius: 4px">
                <ol style="margin:0;">
                    <li><b>
                        <font size=2 color=steelblue>template</font><font size=2 color=silver>&ensp;<</font><font size=2 color=steelblue>typename</font><font size=2 color=MediumSeaGreen>&ensp;E</font><font size=2 color=silver>,</font><font size=2 color=MediumSeaGreen>&ensp;E</font><font size=2 color=gainsboro>&ensp;V</font><font size=2 color=silver>></font>
                    </b></li>
                    <li align="left"><b>
                        <font size=2 color=steelblue>constexpr</font><font size=2 color=silver>&ensp;std::</font><font size=2 color=MediumSeaGreen>string</font><font size=2 color=LemonChiffon>&ensp;get_enum_name</font><font size=2 color=silver>()&ensp;{</font>
                    </b></li>
                    <b><li align="left">
                        <font size=2 color=silver>&emsp;&emsp;&ensp;std::</font><font size=2 color=mediumseagreen>string</font><font size=2 color=skyblue>&ensp;name</font><font size=2 color=silver>(</font><font size=2 color=mediumpurple>__PRETTY_FUNCTION__</font><font size=2 color=silver>);</font>
                    </b></li>
                    <b><li align="left">
                        <font size=2 color=mediumseagreen>&emsp;&emsp;&ensp;QRegularExpression</font><font size=2 color=skyblue>&ensp;reg</font><font size=2 color=silver>(</font><font size=2 color=peru>"(?<=::)[a-zA-Z_]+[</font><font size=2 color=navajowhite>\\</font><font size=2 color=peru>w]*(?=>)"</font><font size=2 color=silver>);</font>
                    </b></li>
                    <li><b>
                        <font size=2 color=mediumseagreen>&emsp;&emsp;&ensp;QRegularExpressionMatch</font><font size=2 color=skyblue>&ensp;match</font><font size=2 color=silver>&ensp;=</font><font size=2 color=skyblue>&ensp;reg</font><font size=2 color=silver>.</font><font size=2 color=lemonchiffon>match</font><font size=2 color=silver>(</font><font size=2 color=skyblue>name</font><font size=2 color=silver>.</font><font size=2 color=lemonchiffon>c_str</font><font size=2 color=silver>());</font>
                    </b></li>
                    <li><b>
                        <font size=2 color=mediumseagreen>&emsp;&emsp;&ensp;QString</font><font size=2 color=skyblue>&ensp;matched</font><font size=2 color=silver>&ensp;=</font><font size=2 color=lemonchiffon>&ensp;""</font><font size=2 color=silver>;</font>
                    </b></li>
                    <li><b>
                        <font size=2 color=mediumorchid>&emsp;&emsp;&ensp;if</font>
                        <font size=2 color=silver>(</font><font size=2 color=skyblue>match</font><font size=2 color=silver>.</font><font size=2 color=lemonchiffon>hasMatch</font><font size=2 color=silver>())&ensp;{</font>
                    </b></li>
                    <li><b>
                        <font size=2 color=skyblue>&emsp;&emsp;&emsp;&emsp;matched</font><font size=2 color=silver>&ensp;=</font><font size=2 color=skyblue>&ensp;match</font><font size=2 color=silver>.</font><font size=2 color=lemonchiffon>captured</font><font size=2 color=silver>(</font><font size=2 color=powderblue>0</font><font size=2 color=silver>);</font>
                    </b></li>
                    <li><b>
                        <font size=2 color=silver>&emsp;&emsp;&ensp;}</font>
                    </b></li>
                    <li><b>
                        <font size=2 color=mediumorchid>&emsp;&emsp;&ensp;return</font><font size=2 color=skyblue>&ensp;matched</font><font size=2 color=silver>.</font><font size=2 color=lemonchiffon>size</font><font size=2 color=silver>()&ensp;></font><font size=2 color=powderblue>&ensp;0</font><font size=2 color=silver>&ensp;?</font><font size=2 color=skyblue>&ensp;matched</font><font size=2 color=silver>.</font><font size=2 color=lemonchiffon>toStdString</font><font size=2 color=silver>()&ensp;:</font><font size=2 color=burlywood>&ensp;""</font><font size=2 color=silver>;</font>
                    </b></li>
                    <li><b>
                        <font size=2 colro=silver>}</font>
                    </b></li>
                </ol>
            </td>
        </tr>
    </tbody>
</table>  

* `get_all_enum_names`:  

<table>
    <tbody>
        <tr style="border: none">
            <td style="background: black; border: none; width: 100vw; border-radius: 4px">
                <ol style="margin:0;">
                    <li><b>
                        <font size=2 color=steelblue>template</font><font size=2 color=silver>&ensp;<</font><font size=2 color=steelblue>typename</font><font size=2 color=mediumseagreen>&ensp;T</font><font size=2 color=silver>,</font><font size=2 color=steelblue>&ensp;int</font><font size=2 color=silver>...</font><font size=2 color=gainsboro>&ensp;Ints</font><font size=2 color=silver>></font>
                    </b></li>
                    <li><b>
                        <font size =2 color=steelblue>constexpr&ensp;auto</font><font size =2 color=lemonchiffon>&ensp;get_all_enum_names</font><font size =2 color=silver>(std::</font><font size =2 color=mediumseagreen>integer_sequence</font><font size =2 color=silver><</font><font size =2 color=steelblue>int</font><font size=2 color=silver>,</font><font size=2 color=gainsboro>&ensp;Ints</font><font size=2 color=silver>...>)&ensp;{</font>
                    </b></li>
                    <li><b>
                        <font size=2 color=silver>&emsp;&emsp;&ensp;std::</font><font size=2 color=mediumseagreen>array</font><font size=2 color=silver><</font><font size=2 color=silver>std::</font><font size=2 color=mediumseagreen>string</font><font size=2 color=silver>,</font><font size=2 color=steelblue>&ensp;sizeof</font><font size=2 color=silver>...(</font><font size=2 color=gainsboro>Ints</font><font size=2 color=silver>)></font><font size=2 color=skyblue>&ensp;names</font><font size=2 color=silver>{</font><font size=2 color=gainsboro>&nbsp;get_enum_name</font><font size=2 color=silver><</font><font size=2 color=mediumseagreen>T</font><font size=2 color=silver>,</font><font size=2 color=steelblue>&ensp;static_cast</font><font size=2 color=silver><</font><font size=2 color=mediumseagreen>T</font><font size=2 color=silver>>(</font><font size=2 color=gainsboro>Ints</font><font size=2 color=silver>)>()...&nbsp;};</font>
                    </b></li>
                    <li><b>
                        <font size=2 color=silver>&emsp;&emsp;&ensp;std::</font><font size=2 color=mediumseagreen>vector</font><font size=2 color=silver><</font><font size=2 color=silver>std::</font><font size=2 color=mediumseagreen>string</font><font size=2 color=silver>></font><font size=2 color=skyblue>&ensp;valid</font><font size=2 color=silver>{};</font>
                    </b></li>
                    <li><b>
                        <font size=2 color=mediumorchid>&emsp;&emsp;&ensp;for</font><font size=2 color=silver>&ensp;(</font><font size=2 color=steelblue>auto const</font><font size=2 color=silver>&</font><font size=2 color=skyblue>&ensp;name</font><font size=2 color=silver>&ensp;:</font><font size=2 color=skyblue>&ensp;names</font><font size=2 color=silver>)&emsp;{</font>
                    </b></li>
                    <li><b>
                        <font size=2 color=mediumorchid>&emsp;&emsp;&emsp;&emsp;&ensp;if</font><font size=2 color=silver>&ensp;(</font><font size=2 color=skyblue>name</font><font size=2 color=silver>.</font><font size=2 color=gainsboro>size</font><font size=2 color=silver>()&ensp;==</font><font size=2 color=powderblue>&ensp;0</font><font size=2 color=silver>)</font><font size=2 color=mediumorchid>&ensp;continue</font><font size=2 color=silver>;</font>
                    </b></li>
                     <li><b>
                        <font size=2 color=skyblue>&emsp;&emsp;&emsp;&emsp;&ensp;valid</font><font size=2 color=silver>.</font><font size=2 color=gainsboro>push_back</font><font size=2 color=silver>(</font><font size=2 color=skyblue>name</font><font size=2 color=silver>);</font>
                    </b></li>               
                    <li><b>
                        <font size=2 color=silver>&emsp;&emsp;&ensp;}</font>
                    </b></li>
                    <li><b>
                        <font size=2 color=mediumorchid>&emsp;&emsp;&ensp;return</font><font size=2 color=skyblue>&ensp;valid</font><font size=2 color=silver>;</font>
                    </b></li>
                    <li><b>
                         <font size=2 color=silver>}</font>
                   </b></li>
                </ol>
            </td>
        </tr>
    </tbody>
</table>  

# 接口封装
<table>
    <tbody>
        <tr style="border: none">
            <td style="background: black; border: none; width: 100vw; border-radius: 4px">
                <ol style="margin:0;">
                    <li><b>
                        <font size=2 color=steelblue>template</font><font size=2 color=silver>&ensp;<</font><font size=2 color=steelblue>typename</font><font size=2 color=mediumseagreen>&ensp;T</font><font size=2 color=silver>,</font><font size=2 color=steelblue>&ensp;int</font><font size=2 color=gainsboro>&ensp;boundary</font><font size=2 color=silver>></font>
                    </b></li>
                    <li><b>
                        <font size=2 color=steelblue>auto</font><font size=2 color=gainsboro>&ensp;enum_names</font><font size=2 color=silver>&ensp;=</font><font size =2 color=gainsboro>&ensp;get_all_enum_names</font><font size =2 color=silver><</font><font size =2 color=mediumseagreen>T</font><font size =2 color=silver>>(</font><font size=2 color=gainsboro>make_sequence</font><font size=2 color=silver><</font><font size=2 color=gainsboro>boundary</font><font size=2 color=silver>>());</font>
                    </b></li>
                </ol>
            </td>
        </tr>
    </tbody>
</table>  
