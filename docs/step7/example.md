# step7 实验指导

本实验指导使用的例子为：

```C
int main() {
    int x = 1;
    if (x) x = 2; else x = 3;
    return x;
}
```

## 词法语法分析
针对 if 语句，我们需要设计 AST 节点来表示它，给出的参考定义如下（框架中已经提供）：

| 节点 | 成员 | 含义 |
| --- | --- | --- |
| `If` | 分支条件 `cond`，真分支 `then`，假分支 `otherwise` | if 分支语句 |

仿照 if 节点，还需要类似地实现条件表达式节点。

### 悬吊 else 问题

这一节引入的 if 语句既可以带 else 子句也可以不带，但这会导致语法二义性：`else` 到底和哪一个 `if` 结合？
例如 `if(a) if(b) c=0; else d=0;`，到底是 `if(a) {if(b) c=0; else d=0;}` 还是  `if(a) {if(b) c=0;} else d=0;`？
这个问题被称为 **悬吊 else（dangling else）** 问题。

如果程序员没有加大括号，那么我们需要通过一个规定来解决歧义。
我们人为规定：`else` 和最近的 `if` 结合，也就是说上面两种理解中只有前者合法。
为了让 parser 能遵守这个规定，一种方法是设置产生式的优先级，优先选择没有 else 的 if。
按照这个规定，parser 看到 `if(a) if(b) c=0; else d=0;` 中第一个 if 时，选择没有 else 的 if；
而看到第二个时只能选择有 else 的 if ，也就使得 `else d=0;` 被绑定到 `if(b)` 而不是 `if(a)` 了。

> 需要说明的是 bison 默认在 shift-reduce conflict 的时候选择shift，从而对悬挂else进行就近匹配。

## 语义分析

本步骤中语义分析没有特别需要增加的内容，只需要在扫描到 if 语句和条件表达式时递归地访问其子结点即可。请注意 if 语句**不总是**有 else 分支，所以在递归到子结点时，请先判断子结点是否存在。

## 中间代码生成
从本步骤开始，由于 MiniDecaf 程序出现了分支结构，我们需要开始考虑跳转语句了。在 Step1-4 中，TAC 代码中的标签只有标志 main 函数入口这一个功能。而现在，我们需要使用标签来指示跳转指令的目标位置。我们用 _Lk 来表示跳转用标签，以此和函数入口标签区分开来。

为了实现 if 语句，我们需要设计两条中间代码指令，分别表示条件跳转和无条件跳转，给出的参考定义如下：

> 请注意，TAC 指令的名称只要在你的实现中是一致的即可，并不一定要和文档一致。

| 指令 | 参数 | 作用 |
| --- | --- | --- |
| `BEQZ` | `T0, Label` | 若 T0 的值为0，则跳转到 LABEL 标签处 |
| `JUMP` | `Label` | 跳转到 LABEL 标签处 |

现在让我们来看看示例所对应的 TAC 代码：

```assembly
main:
    _T1 = 1
    _T0 = _T1
    BEQZ _T0, _L1
    _T2 = 2
    _T0 = _T2
    JUMP _L2
_L1:
    _T3 = 3
    _T0 = _T3
_L2:
    return _T0
```

在这段 TAC 代码中，x 对应的临时变量为 _T0。如果 x 的值为真（不等于0），那么应当执行 then 分支 `x = 2;`，否则执行 else 分支 `x = 3;`。因此，我们设置了两个跳转标签 _L1 和 _L2，分别表示 else 分支开始位置和整个 if 语句的结束位置。如果 x 为假，那么应当跳转到 _L1 处，我们使用一条 BEQ 指令来执行。如果 x 为真，那么按顺序执行 then 分支的代码，并在该分支结束时，用一条 JMP 指令跳转到 if 语句的结束位置，从而跳过 else 分支。在 TAC 生成过程中，每当扫描到 if 语句时，都需要调用 TAC 的底层接口，新建两个跳转标签，并按照这种方式生成中间代码。

当然，如果一条 if 语句没有 else 分支，那么只需要一个跳转标签即可。例如我们将例子中的 if 语句修改为 `if (x) x = 2;`，则对应的 TAC 代码可简化为：

```assembly
main:
    _T1 = 1
    _T0 = _T1
    BEQ _T0, _L1
    _T2 = 2
    _T0 = _T2
_L1:
    return _T0
```

同样地，条件表达式也可以使用类似的方法完成中间代码生成。要注意的是，条件表达式是一种特殊的**表达式**，因此有返回值。同学们在实现的时候不要忘记为其分配临时变量。

## 目标代码生成
Step7 中目标代码生成主要是指令的选择以及 label 的声明，RISC-V 提供了与中间代码中 BEQZ 和 JUMP 类似的指令：

```assembly
step7:             # RISC-V 汇编标签
    beqz t1, step7 # 如果 t1 为 0，跳转到 step7 标签处
    j step7        # 无条件跳转到 step6 标签处
```

# 思考题

1. 你使用语言的框架里是如何处理悬吊 else 问题的？请简要描述。

2. 在实验要求的语义规范中，条件表达式存在短路现象。即：

```c
int main() {
    int a = 0;
    int b = 1 ? 1 : (a = 2);
    return a;
}
```

会返回 0 而不是 2。如果要求条件表达式不短路，在你的实现中该做何种修改？简述你的思路。

# 总结
本节主要就是引入了跳转，后面 Step8 循环语句还会使用。

