# 关于do_XXX()的解析

### 一些前置

1. 该汇编器对于各种立即数的处理都是等到之后做的

## 需要实现的do_XXX()

原项目中的函数很多，根据要实现的arm指令挑出了几个需要实现的

```c
static void do_mov(void);
static void do_ldst(void);
static void do_cmp(void);
static void do_arit(void);
static void do_ldmstm(void);
static void do_shift(void);
static void do_bx(void);
static void do_blx(void);
static void do_branch(void);
static void do_bl(void);
static void do_nop(void);
static void do_push_pop(void);
static void do_mul(void);
```

其中`do_arit()`是出现在`and sub add`等二元运算的操作符中

## 关于arm汇编的机器指令

ARM汇编指令集与机器码 https://blog.csdn.net/DXCyber409/article/details/92838715

一个简单常见的指令结构，如下图

![cfa2c29ea0bdcb5a76acae3e76d1426d.jpg](https://img.gejiba.com/images/cfa2c29ea0bdcb5a76acae3e76d1426d.jpg)

常见指令格式`MNEMONIC{S}{condition} {Rd}, Operand1, Operand2`

对照指令格式和图，可以更好理解。

## 一些样例

#### Mov指令的构造

mov指令比较简单，只需关注两点，目的寄存器和源操作数，分别存储在Rd和shifter_operand字段，其中源操作数可以是立即数或寄存器，

```c
static void do_mov(void)
{
  inst.instruction |= inst.operands[0].reg << 12; // 左移12位作为RD 
  encode_arm_shifter_operand(1);                  // 处理0-11位
}
```

在encode_arm_shifter_operand(1)中，判断完操作数类型后，对立即数的操作是将第25位置一，表示shifter_operand置为立即数，对寄存器则需要多一步处理。之后判断是否为移位寻址，根据移位类型在第5-6位赋对应值。值得注意的是汇编器对于RRX（带扩展的循环右移）做了特判。（默认是作为移位寻址处理的，对于寄存器直接寻址，可能会在移位操作数那赋0。）

```c
static void encode_arm_shifter_operand(int i)
{
  if (inst.operands[i].isreg) // 判断是否为寄存器
  {
    inst.instruction |= inst.operands[i].reg;
    encode_arm_shift(i);
  }
  else 
    inst.instruction |= INST_IMMEDIATE; // 是立即数 但是并没有在0-11位中设置值，应该是在后续fix-up中处理
}

static void encode_arm_shift(int i)
{
  if (inst.operands[i].shift_kind == SHIFT_RRX) // 是带扩展的循环右移
    inst.instruction |= SHIFT_ROR << 5;
  else
  {
    inst.instruction |= inst.operands[i].shift_kind << 5; // 有可能是左移或者不移
    if (inst.operands[i].immisreg)                        // 判断作为移位的第二个操作数是否为寄存器
    {
      inst.instruction |= SHIFT_BY_REG; // 第四位赋1 表示是寄存器
      inst.instruction |= inst.operands[i].imm << 8; //8-11位RS，存放移位寄存器
    }
    else
      inst.reloc.type = BFD_RELOC_ARM_SHIFT_IMM; //移位操作数是立即数  mov r1,r1 直接到这了。
  }
}
```

#### ADD等运算的构造

相比只有一个操作数的MOV，ADD有两个操作数，但是OP1被限制为寄存器，OP2则可以是寄存器也可以是立即数。

```c
static void do_arit(void)
{
  if (!inst.operands[1].present)
    inst.operands[1].reg = inst.operands[0].reg;
  inst.instruction |= inst.operands[0].reg << 12; //左移12位作为RD，
  inst.instruction |= inst.operands[1].reg << 16; //左移16位作为RN，
  encode_arm_shifter_operand(2);				  //处理OP2
}
```

对OP1的操作也是直接左移对应位数，只对OP2进行判断处理，和MOV那一样，就不多说了。

#### LDR运算的构造

LDR指令比较复杂。首先LDR有两种用法，一是LDR伪指令，二是LDR类型数据加载指令

[Arm入门第六讲 伪指令与Load/Store架构 - Android_IBinary - 博客园 (cnblogs.com)](https://www.cnblogs.com/AndroidBinary/p/14999032.html#二丶loadstore架构)

##### 伪指令

> ```c
> LDR{COND}{.W} register, =[expr | label-expr]
> cond 可选的指令执行条件
> .W 可选用来指定指令宽度（Thumb-2指令支持）
> reg: 目标寄存器
> expr: 32位常量表达式，汇编器会根据expr的取值情况，对LDR伪指令做如下处理。
>     1.expr表示的地址值 没有超过MOV 或者MOVN伪指令的地址取值范围，那么LDR在底层会用MOV和MVN来替换LDR
>     2.如果超出了MOV MVN 那么汇编器会将数据防暑数据缓存池（理解为内存）同时用一条基于PC的LDR伪指令来读取这个常数。
>     
> label-expr方式：
>     1.一个程序相关或声明位外部的表达式，那么汇编器会将label-expr表达式的值方式数据缓存池（内存）使用一条程序相关的LDR伪指令将该值取出放入寄存器。
>     
> 
> 2.一个程序相关或声明为外部表达式，汇编器会将LABEL_EXPR表达式的值放入数据缓存池，然后使用一条程序相关的LDR伪指令将该值取出并放入寄存器。 当lable-expr被声明位外部的表达式的时候，汇编器将在目标文件中插入链接重定位伪操作，并且由连接器在连接的时候生成改地址。
> ```

##### 加载指令

> ```c
> LDR{条件} 目的寄存器，<存储器地址>
> ```
>
> LDR指令是从存储器地址读取一个32位的双字数据传送到目的寄存器中。
>
> 这个指令通常用于将内存中的数据读取到通用寄存器，然后处理数据。如果目的寄存器是PC（R15 == EIP）寄存器，那么内存中读取的地址就会当做目标地址，从而实现程序的跳转。

##### do_ldst()

可以看到也是分了两种情况。伪指令使用`move_or_literal_pool`，LOAD指令使用`encode_arm_addr_mode_2`。

```C
static void do_ldst(void)
{
  inst.instruction |= inst.operands[0].reg << 12; // 左移12位作为RD opn1
  if (!inst.operands[1].isreg)                    // 是伪指令的情况
    if (move_or_literal_pool(0, /*thumb_p=*/false, /*mode_3=*/false))
      return;
  // 是LOAD指令的情况
  encode_arm_addr_mode_2(1, /*is_t=*/false);
}
```

##### move_or_literal_pool

首先判断opn2是否为常量表达式，接着判断该立即数的合法性。对于合法立即数，用MOV和MVN。

```c
  if (inst.reloc.exp.X_op == O_constant) // 如果opn2是常量表达式
  {
    int value = (int)encode_arm_immediate((unsigned int)inst.reloc.exp.X_add_number); // 判断立即数是否合法
    if (value != FAIL)                                                                // 如果是合法立即数 转MOV
    {
      /* This can be done with a mov instruction.  */
      inst.instruction &= LITERAL_MASK;                                   // 保留RD 和 cond其他全部清0
      inst.instruction |= INST_IMMEDIATE | (OPCODE_MOV << DATA_OP_SHIFT); // 1101 左移21位
      inst.instruction |= value & 0xfff;                                  // 保留12位
      return 1;
    }
    // 如果是非法立即数 还是说负数？
    value = (int)encode_arm_immediate((unsigned int)~inst.reloc.exp.X_add_number); // 取反 负数是要处理一下？
    if (value != FAIL)                                                             // 负数取反后变成合法立即数
    {
      /* This can be done with a mvn instruction.  */
      inst.instruction &= LITERAL_MASK;
      inst.instruction |= INST_IMMEDIATE | (OPCODE_MVN << DATA_OP_SHIFT); // 用MVN指令，其他和MOV一样
      inst.instruction |= value & 0xfff;
      return 1;
    }
  }
```

对于非法立即数和标号情况，都通过函数add_to_lit_pool处理，处理完后对对应字段赋值。

```c
  if (add_to_lit_pool() == FAIL)
  {
    inst.error = _("literal pool insertion failed");
    return 1;
  }
  inst.operands[1].reg = REG_PC; // 寄存器变PC
  inst.operands[1].isreg = 1;
  inst.operands[1].preind = 1; // 预索引
  inst.reloc.pc_rel = 1;
  // thump默认false
  inst.reloc.type = (mode_3 ? BFD_RELOC_ARM_HWLITERAL : BFD_RELOC_ARM_LITERAL);
```

##### add_to_lit_pool

关于这个函数，我认为难点还是在于这个池子，在该项目中如何表示，结构体中会有哪些属性。

```c
//文字池结构体
struct literal_pool
{
  expressionS literals[MAX_LITERAL_POOL_SIZE]; //用于存放对应的常量表达式
  unsigned int next_free_entry;// 指向数组下一个空位
  unsigned int id;
  symbol *symbol; 			   //文字池标号
  segT section; 			   //用来查找对应节的文字池
  subsegT sub_section; 		   //和section同时使用
  struct literal_pool *next;
};
```

然后就是具体操作了，通过一番操作，取出了有效的信息，池子标号和数据相对偏移。之后就可以根据相应属性做下一步处理。

```c
static int add_to_lit_pool(void)
{
  literal_pool *pool; // 池子
  unsigned int entry;

  pool = find_or_make_literal_pool(); // 找池子

  /* Check if this literal value is already in the pool.查一下这个常数或者标号是不是已经在池子里了  */
  for (entry = 0; entry < pool->next_free_entry; entry++)
  {
      //常量表达式
    if ((pool->literals[entry].X_op == inst.reloc.exp.X_op) && (inst.reloc.exp.X_op == O_constant) &&
        (pool->literals[entry].X_add_number == inst.reloc.exp.X_add_number))
      break;
	  //标号
    if ((pool->literals[entry].X_op == inst.reloc.exp.X_op) && (inst.reloc.exp.X_op == O_symbol) &&
        (pool->literals[entry].X_add_number == inst.reloc.exp.X_add_number) &&
        (pool->literals[entry].X_add_symbol == inst.reloc.exp.X_add_symbol) &&
        (pool->literals[entry].X_op_symbol == inst.reloc.exp.X_op_symbol))
      break;
  }

  /* Do we need to create a new entry? 如果常数不在池子里 */
  if (entry == pool->next_free_entry)
  {
    if (entry >= MAX_LITERAL_POOL_SIZE) // 
    {
      inst.error = _("literal pool overflow");
      return FAIL;
    }

    pool->literals[entry] = inst.reloc.exp; // 把常数放进去
    pool->next_free_entry += 1;
  }

  inst.reloc.exp.X_op = (segT)O_symbol;               // 告诉后面取值要X_add_symbol + X_add_number.即取池子地址加上偏移
  inst.reloc.exp.X_add_number = ((int)entry) * 4 - 8; //  entry当前的池子的下标，*4 - 8是搞啥
  inst.reloc.exp.X_add_symbol = pool->symbol; 		  // 把池子的标号放到这，方便后面找池子

  return SUCCESS;
}
```

##### encode_arm_addr_mode_2()

