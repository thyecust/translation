原文：https://www.figma.com/blog/speeding-up-build-times/
本文是对原文的笔记，一部分是原文。

在2023年，Figma的C++代码量增加了10%，编译时间增加了50%，他们尝试了一些办法
- 买M1 Max，编译时间回到了原本的样子
- 用ccache和远程缓存，也没什么大用
- 开了一个#build-waiting-room的tag，等编译无聊的时候可以进去发发帖子
- 从最大的文件里手懂一个删除所有不必要的#include，编译字节缩小了31%，冷编译时间缩小了25%

总的来说，Figma专注于减少编译字节数。但是他们也尝试了本地缓存、远程缓存和预编译头文件。工程师Charlie Kilpatrick为Bazel Remote Caching设计了一个计划，用于检索一次编译的输入和远程缓存的是否相同。只在需要远程缓存的时候才启用，本地编译时间可以缩短两分钟。

通过减少编译字节数的方案，Figma的编译时间缩短了50%，每天防止了50-100次可能导致编译时间增长的PR。

## DIWYDU

Google有个工具叫Include What You Use (IWYU)，但是这个工具太严厉了，不适用与Figma的仓库，他们开发了一个新工具叫做Don't Include What You Don't Use (DIWYDU)。IWYU要求每个文件使用最精确的include集，DIWYDU则要求每个文件必须用到了自己直接Include集里的一些东西。DIWYDU是用clang的Python binding实现的。

DIWYDU也有一些局限，比如它不适用于分析STL头。例如一个文件直接include了vector.h，同时用到了std::vector这个符号，这个include header应该是必要的。但是DIWYDU的实现会说这个include header是不必要的，因为std::vector并不出现在vector.h里，而是vector.h的某个includeheader里。

另外Python binding也有一些问题，例如会在AST上遇到UNEXPOSED_EXPR类型，得想一些花招应对。

另外，有些include header其实可以替换为forward declaration，或者分成几个小的 header，DIWYDU也不能应对这种情况。

## includes.py

Figma还有个想法，在每个PR的CI里检测文件字节的变化，如果某个PR会对编译时间造成很大的影响，给出一个警告。他们开发了一个Python文件includes.py，它会追随一个文件的include，记录每个header的字节数。它把标准库头文件当成0大小，因为Figma的代码很少直接依赖标准库。最后includes.py会得到一个include的图，然后传递性地计算每个文件的字节数。

## Fwd.h

如果一个工程师遇到了includes.py发出的警告，通常的修复是使用forward declaration。但是这会伤害代码的可读性，也会让代码搜索更难，因为一个符号被多处声明。

Figma在这个问题上的方案是，首先把整个仓库分割成可组装的模块，每个模块是一个文件夹，这个文件夹下放一个Fwd.h文件。这些Fwd.h文件里有这个模块全部的声明，这个模块其他的头文件都要include这个Fwd.h文件。这不仅使得每个符号只会有一次声明，也让工程师不必烦恼用forward declaration还是include了。
