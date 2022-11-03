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

在encode_arm_shifter_operand(1)中，判断完操作数类型后，对立即数的操作是将第25位置一，表示shifter_operand中为立即数，对寄存器则需要多一步处理。之后判断是否为移位寻址，根据移位类型在第5-6位赋对应值。值得注意的是汇编器对于RRX（带扩展的循环右移）做了特判。（默认为移位寻址处理了，对于寄存器直接寻址，可能会在移位操作数那赋0。）

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

