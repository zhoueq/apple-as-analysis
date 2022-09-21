# 约定

* PPC is "PowerPC", or the architecture of Macs
* 源程序支持多种架构，多用宏定义来做区别
* 规则就是用来打破的，编程倡导在一些规则在这里 **不会** 得到体现

# 整体处理流程

整体处理流程有助于理解整个程序的执行主线，以及理解各模块间的组织关系，特别是各数据结构是如何建立并使用的。

## main

* 主调函数：main

* 主要工作： 命令行参数解析、初始化、汇编、生成二进制文件、收尾

* 相关函数：

  > symbol_begin();      /* symbols.c */
  >
  > sections_begin();    /* sections.c */   在apple as 中，这是一个空函数。
  >
  > read_begin();      /* read.c */
  >
  > md_begin();     /* MACHINE.c */       处理机器指令相关的准备工作
  >
  > input_scrub_begin();    /* input_scrub.c */
  >
  > 
  >
  > perform_an_assembly_pass(argc, argv); /* Assemble it. */
  >
  > write_object(out_file_name);



## perform_an_assembly_pass

* 主调函数：perform_an_assembly_pass

* 主要工作： 读入文件、解析文件

* 相关函数：read_a_source_file

  > 在  read_a_source_file 调用 parse_a_buffer 对一个文件进行汇编
  >
  > 对 GAS 做了拆分优化： 
  >
  > This differs from the GNU version by taking the guts of the GNU read_a_source_file() (with the outer most loop removed) and renaming it parse_a_buffer().



## parse_a_buffer

* 上层函数：read_a_source_file

* 主调函数：**parse_a_buffer**

* 主要工作： 解析汇编文件

* 相关函数：各指令处理函数，参见表 pseudo_table、md_pseudo_table 和 builtin_sections

  > 核心流程：
  >
  > ```C
  > while(input_line_pointer < buffer_limit){
  >     // 以 . 开始的指令：
  >         pop = (pseudo_typeS *)hash_find(po_hash, s+1);
  >         (*pop->poc_handler)(pop->poc_val);
  >   
  >     // 是宏定义
  >         add_to_macro_definition(start_of_line);
  >   
  >     // 处理 : 和 = 两个符号，即标号后的内容及等号后的内容
  >         colon(s, 0);    equals(s);
  >   
  >     // 其它情况：对指令进行汇编
  >         md_assemble(s);
  > }
  > ```
  >
  >  
  >
  > This differs from the GNU version by taking the guts of the GNU read_a_source_file() (with the outer most loop removed) and renaming it parse_a_buffer().



## md_assemble

* 上层函数：parse_a_buffer

* 主调函数：**md_assemble**

* 主要工作： 汇编一条指令

* 相关函数：**opcode_lookup**, **parse_operands**, output_inst

  > 核心流程：根据汇编助词符，从 struct asm_opcode insns 表（arm.c 文件中，参汇编指令的准备）中获取该指令的 opcode，再将 opcode 中的 operands 交由 parse_operands 来进行处理。期间会修改全局变量 static struct arm_it inst; 特别是该结构体的 inst.instruction；
  >
  > 
  >
  > 解析完成后， inst 也完成了部分的修改，此时会调用 opcode 结构中 aencode 所指向的函数，完成指令的编码，即进一步完善 inst 中的 instruction 中各位的值。 aencode 的注释为：Function to call to encode instruction in ARM format。完善过程中所涉及的函数多为机器指令编码相关的函数，此类函数的命名多以 encode_arm_ 开始。
  >
  > 
  >
  > 最后调用 output_inst 完成汇编。



## opcode_lookup

* 上层函数：md_assemble

* 主调函数：**opcode_lookup**

* 主要工作： 根据助记符查找对应的 struct asm_opcode，特别是要处理助符前面的条件.

* 相关函数：hash_find_n

  > 核心流程：
  >
  > ```C
  > /* Subroutine of md_assemble, responsible for looking up the primary
  >    opcode from the mnemonic the user wrote.  STR points to the
  >    beginning of the mnemonic.
  > 
  >    This is not simply a hash table lookup, because of conditional
  >    variants.  Most instructions have conditional variants, which are
  >    expressed with a _conditional affix_ to the mnemonic.  If we were
  >    to encode each conditional variant as a literal string in the opcode
  >    table, it would have approximately 20,000 entries.
  > 
  >    Most mnemonics take this affix as a suffix, and in unified syntax,
  >    'most' is upgraded to 'all'.  However, in the divided syntax, some
  >    instructions take the affix as an infix, notably the s-variants of
  >    the arithmetic instructions.  Of those instructions, all but six
  >    have the infix appear after the third character of the mnemonic.
  > 
  >    Accordingly, the algorithm for looking up primary opcodes given
  >    an identifier is:
  > 
  >    1. Look up the identifier in the opcode table.
  >       If we find a match, go to step U.
  > 
  >    2. Look up the last two characters of the identifier in the
  >       conditions table.  If we find a match, look up the first N-2
  >       characters of the identifier in the opcode table.  If we
  >       find a match, go to step CE.
  > 
  >    3. Look up the fourth and fifth characters of the identifier in
  >       the conditions table.  If we find a match, extract those
  >       characters from the identifier, and look up the remaining
  >       characters in the opcode table.  If we find a match, go
  >       to step CM.
  > 
  >    4. Fail.
  > 
  >    U. Examine the tag field of the opcode structure, in case this is
  >       one of the six instructions with its conditional infix in an
  >       unusual place.  If it is, the tag tells us where to find the
  >       infix; look it up in the conditions table and set inst.cond
  >       accordingly.  Otherwise, this is an unconditional instruction.
  >       Again set inst.cond accordingly.  Return the opcode structure.
  > 
  >   CE. Examine the tag field to make sure this is an instruction that
  >       should receive a conditional suffix.  If it is not, fail.
  >       Otherwise, set inst.cond from the suffix we already looked up,
  >       and return the opcode structure.
  > 
  >   CM. Examine the tag field to make sure this is an instruction that
  >       should receive a conditional infix after the third character.
  >       If it is not, fail.  Otherwise, undo the edits to the current
  >       line of input and proceed as for case CE.  */
  > 
  > static const struct asm_opcode* opcode_lookup (char **str);
  > ```
  >
  > 

## parse_operands

* 上层函数：md_assemble

* 主调函数：**parse_operands**

* 主要工作： 根据汇编指令，解析指令后面的参数

* 相关函数：

  * 宏 po_char_or_fail 对应的 skip_past_char 函数
  * 宏 po_reg_or_fail 对应的 arm_typed_reg_parse 函数
  * 宏 po_reg_or_goto 对应的 arm_typed_reg_parse 函数
  * 宏 po_imm_or_fail 对应的 parse_immediate 函数
  * 宏 po_scalar_or_goto 对应的 parse_scalar 函数
  * 宏 po_misc_or_fail
  * 宏 po_misc_or_fail_no_backtrack

  > 核心流程：根据传进来的 operands，再由一个超长的  switch  来处理operands的各种情况
  >
  > ```C
  > unsigned const char *upat = pattern;
  > // po_char_or_fail 等宏定义
  > ......
  > for (i = 0; upat[i] != OP_stop; i++){
  > 		....
  >     switch (upat[i])
  >     {
  >       /* Registers */
  >     	case OP_oRRnpc:
  >       case OP_RRnpc:
  >       case OP_oRR:
  >       case OP_RR:    po_reg_or_fail (REG_TYPE_RN);	  break;
  >       ......
  >     }
  >   	......
  >    /* Various value-based sanity checks and shared operations.  We
  > 	 do not signal immediate failures for the register constraints;
  > 	 this allows a syntax error to take precedence.	 */
  >    switch (upat[i])
  > 	 {
  >       case OP_oRRnpc:
  >       case OP_RRnpc:
  >       case OP_RRnpcb:
  >       case OP_RRw:
  >       case OP_oRRw:
  >       case OP_RRnpc_I0:
  >         if (inst.operands[i].isreg && inst.operands[i].reg == REG_PC)
  >           inst.error = BAD_PC;
  >         break;
  >      ......
  >      
  >     }
  > } 
  > ```
  >
  > 



## output_inst

* 上层函数：parse_a_buffer

* 主调函数：**output_inst**

* 主要工作： 将汇编好的指令输出到相应的数据结构

* 相关函数：output_relax_insn，frag_more，dwarf2_emit_insn

  > 核心流程：
  >
  > * 在当前的 frags 中给指令分配 inst.size 个字节的空间（调用 frag_more 完成）
  > * 将 inst.instruction 的内容写入刚分配的空间（调用 md_number_to_chars 完成，这里需要区别大小端）
  > * 当 inst.reloc.type 不是 BFD_RELOC_UNUSED 时调用 fix_new_arm 在 notes 中构建一个和重定位相关的 fix 结构 



## layout_addresses

* 上层函数：perform_an_assembly_pass

* 主调函数：**layout_addresses**

* 主要工作： 在 perform_an_assembly_pass 完成汇编工作后，为每个代码片段及符号确定地址

* 相关函数：relax_section、relax_align

  > 核心流程：
  >
  > 创建最后一个空白的 frag。
  >
  > ```C
  > /*
  >  * layout_addresses() is called after all the assembly code has been read and
  >  * fragments, symbols and fixups have been created.  This routine sets the
  >  * address of the fragments and symbols.  Then it does the fixups of the frags
  >  * and prepares the fixes so relocation entries can be created from them.
  >  */
  > void layout_addresses(void)
  > ```
  >
  > 



# 数据结构设计

## 哈希表

```C
struct _obstack_chunk		/* Lives at front of each chunk. */
{
  char  *limit;			/* 1 past end of this chunk */
  struct _obstack_chunk *prev;	/* address of prior chunk or NULL */
  char	contents[4];		/* objects begin here */
};

struct obstack		/* control current object in current chunk */
{
  size_t chunk_size;		/* preferred size to allocate chunks in */
  struct _obstack_chunk* chunk;	/* address of current struct obstack_chunk */
  char	*object_base;		/* address of object we are building */
  char	*next_free;		/* where to add next char to current object */
  char	*chunk_limit;		/* address of char after current chunk */
  int	temp;			/* Temporary for some macros.  */
  int   alignment_mask;		/* Mask of alignment for each object. */
  void *(*chunkfun)(size_t);	/* User's fcn to allocate a chunk.*/
  void (*freefun)(void*);	/* User's function to free a chunk.  */
};

//////////////////////////////////////////////////////////////////////////////////////////////////

/* A hash table.  */
struct hash_control {
  /* The hash array.  */
  struct hash_entry **table;
  /* The number of slots in the hash table.  */
  unsigned int size;
  /* An obstack for this hash table.  */
  struct obstack memory;
};
```

### Obstack

Obstack是C标准库里面对内存管理的GNU扩展（实际上就是GNU C library了）。Obstack===Object stack。没错，Obstack就是一个栈，栈里面的元素是对象object（这里的对象单指数据元素），这些数据是动态的。这种内存管理技术属于基于区域的内存管理（Region-based memory management）。

　　在这种内存管理模式将所有申请的数据对象集中放在一起（地址不一定连续。集中在一起是一种逻辑结构，比如Obstack的栈）。好处就是，可以一次性的释放内存，集中管理。

以下为GNU C的Obstack描述：

　　Obstack是一种内存池，内存池里面包含了数据对象栈。

　　内存池也叫做固定大小的块内存申请（fixed-size blocks allocation），同样也是一种内存管理技术。这种技术允许动态申请内存。（实际上固定的是总大小，动态是指内存实际申请使用）这种方式有点类似malloc或者C++的new操作。但是malloc和new操作都会造成内存碎片化问题。更有效地方式就是使用相同大小内存池了。预先申请，集中管理。应用程序在内存池里面自由的玩耍，从来不上岸。

　　回过头继续讲Obstack。你可以创建任意数量的独立的Obstack，然后在这些Obstack里面申请对象。**这些对象遵循栈的逻辑结构**：最后申请的对象必须第一个释放。不同的Obstack里面的数据是相互独立的，没有任何关系。除了对内存释放顺序的要求之外，obstack是非常通用的：一个Obstack可以包含任意数量任意尺寸的对象。

　　Obstack里面内存申请的实现用的一般都是宏。

　　Obstack的表现形式是一个结构体:struct obstack。这个结构有一个非常小的固定大小，记录了obstack的状态，以及如何找到该Obstack里的对象。但是呢，obstack结构体自身不包含任何对象，别妄想直接探寻struct obstack的内容了。必须使用规定的函数才行。

　　你可以通过声明一个struct obstack类型的变量来创建obstack。或者动态申请struct obstack *obstack = (struct obstack *)malloc(sizeof(struct obstack))。

　　Obstack使用的函数都需要指定使用的是那个obstack。Obstack里面的对象会被打包成一个大的内存块（叫做chunks）。struct obstack结构指针指向当前使用的chunks。

　　每当申请的对象无法塞进之前的chunk时，都会创建一个新的chunk。这些chunk以何种形式连接在一起（链表？树？），不是我们关心的啦。这些交由obstack自动管理。chunk由obstack自动创建，但创建的方式得由你来决定，一般会直接或间接的用到malloc函数。



### 哈希表的创建

创建 ``hash_control`` 结构体，并由 ``obstack_begin`` 宏调用 ``_obstack_begin``函数 初始化哈希表中的 ``memory`` 结构，在初始化过程中调用宏``obstack_chunk_alloc``指定的函数（即 obstack 中 chunkfun 所指向的函数 ）分配``chunk_size``大小（这个大小固定不变）的内存池（内存块），即一个 chunk，完成初始``obstack``的建立。在 obstack 中，每个 chunk 都由结构体 _obstack_chunk 来管理，也就是说，每个chunk 的最开始有如下两个指针：

```C
char  *limit;			/* 1 past end of this chunk */
struct _obstack_chunk *prev;	/* address of prior chunk or NULL */
```

指针之后才是用户可用的内存区域，即 contents 所在的位置。因此，obstack 中的 next_free  和 object_base 都指向 contents。

在 obstack 创建完成后，再由 obstack_alloc 函数在刚创建的 obstack 上为哈希表创建各哈希项 （hash_entry），完成哈希表的创建。

> 实现细节：如果哈希表的初始表项比较多（如默认的 4051 个），那就会自动的再分配一个更大的内存块：
>
> ```C
> int obj_size = (int)(h->next_free - h->object_base);
> /* Compute size for new chunk.  */
> // length 为 新对象 的大小
> new_size = (obj_size + length) + (obj_size >> 3) + 100;
> //TODO: 这里的 (obj_size >> 3) + 100 是什么意思?
> ```
>
> 之后再将当前 chunk 中的已有的对象移到新的 chunk 中。

## 哈希表的使用

### 符号表

```C
/* symbol-name => struct symbol pointer */
struct hash_control *sy_hash = NULL;
```

sy_hash是符号表：名字 ==> 结构体指针，此外，所有的符号结构体也由一个链表串连，symbol_rootP 和 symbol_lastP 分别指向第一个和最后一个符号

### 伪指令 

每个伪指令都对应一个处理函数，该处理函数有一个参数，将这三个由结构体 pseudo_typeS 来处理：

```C
/* set up pseudo-op tables */
static struct hash_control *po_hash = NULL;


/*
 * A pseudo opcode table entry.
 */
typedef struct {
  char *poc_name; /* assembler mnemonic, lower case, no '.' */
  void (*poc_handler)(uintptr_t poc_val); /* Do the work */
  uintptr_t poc_val; /* Value to pass to handler */
} pseudo_typeS;
```

> 注：在解析时， parse_a_buffer 函数中，会识别各个助词符, 识别后再调用对应的处理函数 poc_handler ，此时多是各助记符自己识别指令中剩余的部分。

pseudo_typeS 结构的实例如下：

```C
/*
 * The machine independent pseudo op table.
 */
static const pseudo_typeS pseudo_table[] = {
  { "align",	s_align_ptwo,	1	},
  { "align32",	s_align_ptwo,	4	},
  { "p2align",	s_align_ptwo,	1	},
  { "p2alignw",	s_align_ptwo,	2	},
  { "p2alignl",	s_align_ptwo,	4	},
  { "balign",	s_align_bytes,	1	},
  { "balignw",	s_align_bytes,	2	},
  { "balignl",	s_align_bytes,  4	},
  { "org",	s_org,		0	},
  ...
};
// 类似的表还有：
// md_pseudo_table
// builtin_sections
```

该表在初始化时由 pseudo_op_begin 函数为各结构体建立索引：即以 poc_name 为 key，结构体的地址为 value，插入到哈希表中。

> 关于 builtin_sections，定义如下：
>
> ```C
> struct builtin_section {
>     char *directive;
>     char *segname;
>     char *sectname;
>     uint32_t flags; /* type & attribute */
>     uint32_t default_align;
>     uint32_t sizeof_stub;
> };
> ```
>
> 该表只保存了节、段的信息，但在初始化时，也为其每一项创建了 pseudo_typeS 结构 tem，并以 directive 为 key，tem 的地址为 value，插入到了哈希表 po_hash 中，其中 tem 结构体中的 poc_handler 指向函数  s_builtin_section。

### 宏处理

```C
static struct hash_control *ma_hash = NULL;	/* use before set up: NULL-> address error */
static struct obstack macros;	/* obstack for macro text */
```

### 指令相关

```C
static struct hash_control *arm_ops_hsh;
static struct hash_control *arm_cond_hsh;
static struct hash_control *arm_shift_hsh;
static struct hash_control *arm_psr_hsh;
static struct hash_control *arm_v7m_psr_hsh;
static struct hash_control *arm_reg_hsh;
static struct hash_control *arm_reloc_hsh;
static struct hash_control *arm_barrier_opt_hsh;
```

## 节 section

```C
/*
 * For every section the user mentions in the assembley program, we make one
 * struct frchain.  Each section has exactly one struct frchain and vice versa.
 *
 * Struct frchain's are forward chained (in ascending order of section number).
 * The chain runs through frch_next of each section. 
 *
 * From each struct frchain dangles a chain of struct frags.  The frags
 * represent code fragments, for that section, forward chained.
 */

struct frchain {			/* control building of a frag chain */
    struct frag	    *frch_root;		/* 1st struct frag in chain, or NULL */
    struct frag     *frch_last;		/* last struct frag in chain, or NULL */
    struct frchain  *frch_next;		/* next in chain of struct frchain-s */

    section_t	     frch_section;	/* section info, name, type, etc. */
    uint32_t	     frch_nsect;	/* section number (1,2,3,...) */
    struct fix      *frch_fix_root;	/* section fixups */
    isymbolS	    *frch_isym_root;	/* 1st indirect symbol in chain */
    isymbolS	    *frch_isym_last;	/* last indirect symbol in chain */
    symbolS	    *section_symbol;	/* section symbol for dwarf if set */
    uint32_t	     has_rs_leb128s;	/* section has some rs_leb128 frags */
    uint32_t	     layout_pass;	/* pass order for layout_addresses() */
};

typedef struct frchain frchainS;

// 两个全局变量

/*
 * All sections' chains hang off here.  NULL means no frchains yet.
 */
extern frchainS *frchain_root;

/*
 * The frchain we are assembling into now.  That is, the current section's
 * frag chain, even if it contains no (complete) frags.
 */
extern frchainS *frchain_now;
```

### 节的创建

```C
/*
 * section_new() (for non-zerofill sections) switches to a new section, creating
 * it if needed, and creates a fresh fragment.  If it is the current section
 * nothing happens except checks to make sure the type, attributes and
 * sizeof_stub are the same.  The segment and section names will be trimed to
 * fit in the section structure and is the responsiblity of the caller to
 * report errors if they don't.  For zerofill sections only the struct frchain
 * for the section is returned after possibly being created (these section are
 * never made the current section and no frags are ever touched).
 *
 * Globals on input:
 *    frchain_now points to the (possibly none) struct frchain for the current
 *    section.
 *    frag_now points at an incomplete frag for current section.
 *    If frag_now == NULL, then there is no old, incomplete frag, so the old
 *    frag is not closed off.
 *
 * Globals on output:
 *    frchain_now points to the (possibly new) struct frchain for this section.
 *    frchain_root updated if needed (for the first section created).
 *    frag_now is set to the last (possibly new) frag in the section.
 *    now_seg is set to the Mach-O section number (frch_nsect field) except
 *     it is not set for section types of S_ZEROFILL or S_THREAD_LOCAL_ZEROFILL.
 */

frchainS *section_new(char *segname, char *sectname, 
                      uint32_t type, uint32_t attributes, uint32_t sizeof_stub)
```



## 代码片段 fragment

所有的代码片段 frag 都保存于名为 frags 的 obstack 里。全局亦是 frag_now 始终指向当前的代码片段。

```C
struct obstack frags = { 0 };	/* All, and only, frags live here. */
fragS *frag_now = NULL;	/* -> current frag we are building. */
```

在节中有两个指针分别指向该节中的第一个片段和最后一个片段。

```C
struct frag	    *frch_root;		/* 1st struct frag in chain, or NULL */
struct frag     *frch_last;		/* last struct frag in chain, or NULL */
```

代码片段的定义如下：

```C
/*
 * A code fragment (frag) is some known number of chars, followed by some
 * unknown number of chars. Typically the unknown number of chars is an
 * instruction address whose size is yet unknown. We always know the greatest
 * possible size the unknown number of chars may become, and reserve that
 * much room at the end of the frag.
 * Once created, frags do not change address during assembly.
 * We chain the frags in (a) forward-linked list(s). The object-file address
 * of the 1st char of a frag is generally not known until after relax().
 * Many things at assembly time describe an address by {object-file-address
 * of a particular frag}+offset.

 */
struct frag			/* a code fragment */
{
    uint64_t fr_address;	/* Object file address. */
    uint64_t last_fr_address;	/* When relaxing multiple times, remember the */
				/* address the frag had in the last relax pass*/
    struct frag *fr_next;	/* Chain forward; ascending address order. */
				/* Rooted in frch_root. */

    int32_t fr_fix;		/* (Fixed) number of chars we know we have. */
				/* May be 0. */
    int32_t fr_var;		/* (Variable) number of chars after above. */
				/* May be 0. */
    struct symbol *fr_symbol;	/* For variable-length tail. */
    int32_t fr_offset;		/* For variable-length tail. */
    char *fr_opcode;		/* ->opcode low addr byte,for relax()ation*/
    relax_stateT fr_type;	/* What state is my tail in? */
    relax_substateT fr_subtype;	/* Used to index in to md_relax_table for */
				/*  fr_type == rs_machine_dependent frags. */
#ifdef ARM
    /* Where the frag was created, or where it became a variant frag.  */
    char *fr_file;
    unsigned int fr_line;
    /* Flipped each relax pass so we can easily determine whether
       fr_address has been adjusted.  */
    unsigned int relax_marker:1,
		 pad:31;
#endif /* ARM */
    char fr_literal[1];		/* Chars begin here. */
				/* One day we will compile fr_literal[0]. */
};

/* We want to say fr_literal[0] below */
#define SIZEOF_STRUCT_FRAG \
 ((uintptr_t)zero_address_frag.fr_literal - (uintptr_t)&zero_address_frag)

```

在 frag 结构中，虽然结构中定义的是 char fr_literal[1]，占 1 字节空间，但 SIZEOF_STRUCT_FRAG 的计算出来是 **不包含** **不包含** **不包含** 这 1 个字节的，即 fr_literal 即为 frag 结构外的地址，后面会看到，frag 结构本身为数据头，而fr_literal 实际指向 frag 的数据体。



### 片段的创建

在创建“节”时，会创建该节的第一个 frag。

之后在汇编过程中，每碰到一个类似 align 的伪指令，就会新创建一个 frag，

> 如前所述，该 frag 会保存于 frags 这个 obstack 里，在汇编之后的指令时，这些指令对应的数据都会跟随在 frag 之后，也就是 frag 结构最后的成员 fr_literal所指的地方开始，该数据的大小在汇编时是不知道的，只有到建立下一下 frag 时才确定。
>
> 
>
> 在汇编当前片段的代码时，对每一条汇编指令 opcode，都将生成一个对应的机器指令 inst，即在当前的 frag 中给指令分配 inst.size 个字节的空间（调用 frag_more 完成），再将其写入到 frag 里 （参总体流程中 output_inst 的描述）。frag_now 结构体只是一个数据“头”，数据“体”在汇编过程中不断的增长，直到要建立新的 frag。

汇编完整个文件后，在



### 代码对齐

frag_now 的注释有这样一个描述：

```C
/*
 * frag_now points at the current frag we are building. This frag is incomplete.
 * It is, however, included in frchain_now. Frag_now->fr_fix is not the total
 * bytes in use for the frag.  For that use:
 * frag_now->fr_fix + obstack_next_free(&frags) - frag_now->fr_literal.
 */
```

而在 frag_new 函数中，fr_fix 的值计算代码为。

```C
/*
 *			frag_new()
 *
 * Call this to close off a completed frag, and start up a new (empty)
 * frag, in the same subsegment as the old frag.
 * [frchain_now remains the same but frag_now is updated.]
 * Because this calculates the correct value of fr_fix by
 * looking at the obstack 'frags', it needs to know how many
 * characters at the end of the old frag belong to (the maximal)
 * fr_var: the rest must belong to fr_fix.
 * It doesn't actually set up the old frag's fr_var: you may have
 * set fr_var == 1, but allocated 10 chars to the end of the frag:
 * in this case you pass old_frags_var_max_size == 10.
 *
 * Make a new frag, initialising some components. Link new frag at end
 * of frchain_now.
 */
void
frag_new(
int old_frags_var_max_size)	/* Number of chars (already allocated on obstack
				   frags) in variable_length part of frag. */
{
    ......
    /* cctools-port: Added (intptr_t) cast to silence warning */
    frag_now->fr_fix = (int)(intptr_t)((char *) (obstack_next_free (&frags)) -
			     (long)(frag_now->fr_literal) -
			     old_frags_var_max_size);
 		......
}
```

要理解这里的计算逻辑，就需要确切理解 “对齐” 指令的含义。这里需要了解至少三个函数 s_align、frag_align 和 frag_var。其中，在 s_align 中有

```C
power_of_2_alignment = (int)get_absolute_expression();
```

因此，对伪指令 .align 2 来说，power_of_2_alignment 的值将是 2。而对于 arm，有：

```C
temp_fill = 0xe1a00000; /* arm nop */
fill_size = 4; /* 4 byte fill size */
```

之后再调用 md_number_to_chars(fill, temp_fill, fill_size);  将 temp_fill 写入到 char fill[4] 中。准备好这一切后，开始真正的对齐操作，即调用

```C
/* Only make a frag if we HAVE to. . . */
if(power_of_2_alignment != 0)
	    frag_align(power_of_2_alignment, fill, fill_size, max_bytes_to_fill);
```

仍以  .align 2 为例，相当于 frag_align(2, fill, 4, 0)，注 max_bytes_to_fill 在没有给定的情况下默认为 0。而在该函数中，max_chars = 4 + （4-1）= 7；即为了以 4 字节对齐，max_char 的值最大就为 7 （最差的情况，当前地址空 3 个字节后才为 4 的倍数，自己再占用 4 字节）。 

```c
/*
 *			frag_align()
 *
 * Make an rs_align frag for:
 *   .align power_of_2_alignment, fill_expression, fill_size, max_bytes_to_fill
 * the fill_size must be 1, 2 or 4.  An rs_align frag stores the
 * power_of_2_alignment in the fr_offset field of the frag, the fill_expression
 * in the fr_literal bytes, the fill_size in the fr_var field and the
 * max_bytes_to_fill in the fr_subtype field.  We call frag_var() with max_chars
 * parameter large enough to hold the fill_expression of fill_size plus the
 * maximum number of partial bytes that may be needed to be zeroed before the
 * fill.
 *
 * Call to close off the current frag with a ".align", then start a new
 * (so far empty) frag, in the same subsegment as the last frag.
 */
void
frag_align(
int power_of_2_alignment,
char *fill,
int fill_size,
int max_bytes_to_fill)
{
    void *fr_literal;
    int max_chars;
	max_chars = fill_size + (fill_size - 1);

	fr_literal = (void *)
	    frag_var(rs_align,				/* type */
		     max_chars,				/* max_chars */
		     fill_size,				/* var */
		     (relax_substateT)max_bytes_to_fill,/* subtype */
		     (symbolS *)0,			/* symbol */
		     (int32_t)power_of_2_alignment,	/* offset */
		     (char *)0);			/* opcode */
    if(fill_size == 1 || fill_size == 2 || fill_size == 4)
	memcpy(fr_literal, fill, fill_size);
    else
	as_bad("Invalid width for fill expression.");
}


/*
 *			frag_var()
 *
 * Start a new frag unless we have max_chars more chars of room in the current frag.
 * Close off the old frag with a .fill 0.
 *
 * Set up a machine_dependent relaxable frag, then start a new frag.
 * Return the address of the 1st char of the var part of the old frag
 * to write into.
 */
char *
frag_var(
relax_stateT type,
int max_chars,
int var,
relax_substateT subtype,
symbolS *symbol,
int32_t offset,
char *opcode)
{
    register char  *retval;

#ifdef ARM
    as_file_and_line (&frag_now->fr_file, &frag_now->fr_line);
#endif /* ARM */
    frag_grow (max_chars); // 检查是否有足够的增长空间
  
    // 返回值，当前 obstack 中空闲的位置，返回后 frag_align 会根据这个值将 fill 的内容填充进去
    retval = obstack_next_free (&frags);  
  
    // 空闲的地址增加 max_chars，即当前的 frag 多增加max_chars
    obstack_blank_fast (&frags, max_chars); 

    // fill_size 保存于 fr_var，即注释中讲的，实际占用空间，由于对齐，实际分配空间需要变动
    // 另 frag_now->fr_fix 的值由 frag_new 函数计算得出。
    frag_now->fr_var = var;          
  
    frag_now->fr_type = type;        // rs_align
    frag_now->fr_subtype = subtype;  // max_bytes_to_fill
    frag_now->fr_symbol = symbol;    // 0
    frag_now->fr_offset = offset;    // power_of_2_alignment
    frag_now->fr_opcode = opcode;    // 0

    // 新的 frag，这里的 max_chars 是为了计算当前 frag_now 中的 fr_fix 的值，与 frag 的大小无关
    frag_new (max_chars);            
#ifdef ARM
    as_file_and_line (&frag_now->fr_file, &frag_now->fr_line);
#endif /* ARM */
    return (retval);
}				/* frag_var() */
```

frag_var 返回后，frag_align 再根据返回的 retval 将 temp_fill = 0xe1a00000; /* arm nop */ 的值填充到上一个 frag，完成对齐操作。下图展示了汇编程序碰到第一个 .align 2 伪指令时，完成对齐操作前后 frags 上的值。 

```C
       section
     frchain_root                            obstack  
|--------------------|                        frags
|     frch_root      |---------------->|--------------------|<----frag_now (old)
|--------------------|                 |      ......        |
|      ......        |                 |--------------------|
|--------------------|                 |  fix = 0; var = 4; |
                                       |--------------------|
                                       |      ......        |
       (old) frag_now.fr_literal------>|--------------------|<----retval
                                       | 0xe1a00000 (4字节)  |
                                       |--------------------|
                                       |    对齐留空 (3字节)  |
                                       |--------------------|<----frag_now (new)
                                       |      ......        |
                                       |--------------------|
                                       |  fix = 0; var = 0; |
                                       |--------------------|
                                       |      ......        |
       (new) frag_now.fr_literal------>|--------------------|<----frags->next_free
                                       |     free space     |
                                       |                    |
                                         
```

在上图中，frag_now (old) 里的 fr_fix 的值为 0，那该值是什么含义？简单说，fr_fix 就是该代码片段对应“数据体”的固定大小，而 fr_var 则为可变大小。上图展示的是对第一个 .align 2 伪指令的处理，而此时的 frag_now 是随 section 的建立而建立的，即没有处理任何实际的指令，因此 fr_fix 的值为 0。而新建立的 frag_now (new) 在初始化时，结构体内的数据全置为 0。



这里有一个词 “relaxable frag”，需要解释。











### 









# 词法分析

```C
#define is_name_beginner(c)     ( lex_type[(c) & 0xff] & LEX_BEGIN_NAME )
#define is_part_of_name(c)      ( lex_type[(c) & 0xff] & LEX_NAME       )

```

遇到 .section 时，在哈希表 po_hash 中查找 相应的 “value” 并得到相应的处理函数，如 s_section，再由该函数对 .section 指令后的进行解析处理。

```C
/*
 *   s_section() implements the pseudo op:
 *	.section segname , sectname [[[ , type ] , attribute] , sizeof_stub]
 */
// .section   __TEXT,__text,regular,pure_instructions

/*
 * s_globl() implements the pseudo op:
 *	.globl name [ , name ]
 */
// .globl	_fib              
```

section_new 有一个查找过程，如果是 zerofill sections，则并不创建新的 frag。

```C

```



# 伪指令处理

函数：md_assemble

```C
/*
 * pseudo_op_begin() creates a hash table of pseudo ops from the machine
 * independent and machine dependent pseudo op tables.
 */
static void pseudo_op_begin(void);
// 该函数在初始化时被 read_begin 函数调用
// 对应的表：
// pseudo_table, md_pseudo_table
```

即各指令及对应的解析函数可以通过 pseudo_table, md_pseudo_table 这两个表快速查看。



## .file 的处理

```C
/* Handle two forms of .file directive:
   - Pass .file "source.c" to s_app_file
   - Handle .file 1 "source.c" by adding an entry to the DWARF-2 file table
   If an entry is added to the file table, return a pointer to the filename. */
char * dwarf2_directive_file (uintptr_t dummy ATTRIBUTE_UNUSED);
  
/*
 * s_app_file() implements the pseudo op:
 *	.file name [ level_number ]
 * the level number is generated by /lib/cpp and is just ignored.
 */
void s_app_file();
```

## .text 的处理

在函数 static uint32_t s_builtin_section(const struct builtin_section *s) 中，调用 section_new 函数，解释如下

> section_new() (for non-zerofill sections) switches to a new section, creating it if needed, and creates a fresh fragment.  If it is the current section nothing happens except checks to make sure the type, attributes and sizeof_stub are the same. 



# 汇编指令处理

## 汇编指令结构

```c
struct asm_opcode
{
  /* Basic string to match.  */
  const char *template;

  /* Parameters to instruction.	 */
  unsigned char operands[8];

  /* Conditional tag - see opcode_lookup.  */
  unsigned int tag : 4;

  /* Basic instruction code.  */
  unsigned int avalue : 28;

  /* Thumb-format instruction code.  */
  unsigned int tvalue;

  /* Which architecture variant provides this instruction.  */
  const arm_feature_set *avariant;
  const arm_feature_set *tvariant;

  /* Function to call to encode instruction in ARM format.  */
  void (* aencode) (void);

  /* Function to call to encode instruction in Thumb format.  */
  void (* tencode) (void);
};

```

## 汇编指令准备

这里的准备即指令这里定义了若干宏来处理：

```C
static const struct asm_opcode insns[] =
{
	......
 tCE(and,	0000000, and,      3, (RR, oRR, SH), arit, t_arit3c),
 tC3(ands,	0100000, ands,	   3, (RR, oRR, SH), arit, t_arit3c),
 tCE(eor,	0200000, eor,	   3, (RR, oRR, SH), arit, t_arit3c),
  ......
}

//上面三个宏展开后的结构体为

 { "and", { OP_RR,OP_oRR,OP_SH, }, OT_csuffix, 0x0000000, T_MNEM_and, &arm_ext_v1, &arm_ext_v4t, do_arit, do_t_arit3c },
 { "ands", { OP_RR,OP_oRR,OP_SH, }, OT_cinfix3, 0x0100000, T_MNEM_ands, &arm_ext_v1, &arm_ext_v4t, do_arit, do_t_arit3c },
 { "eor", { OP_RR,OP_oRR,OP_SH, }, OT_csuffix, 0x0200000, T_MNEM_eor, &arm_ext_v1, &arm_ext_v4t, do_arit, do_t_arit3c },
```

其中：

* OP_xx, OT_xx, T_MNEM_xx 都是枚举类型；
* arm_ext_v 开始的为定义的结构体，分别对应指令结构体中的 avariant 和 tvariant
* do_arit 和 do_t_arit3c 分别为定义的编码函数，其实现参 arm.c

## 机器指令结构

```C
struct arm_it
{
  const char *  error;
  uint32_t instruction;
  int           size;
  int		size_req;
  int		cond;
  /* "uncond_value" is set to the value in place of the conditional field in
     unconditional versions of the instruction, or -1 if nothing is
     appropriate.  */
  int		uncond_value;
  struct neon_type vectype;
  /* Set to the opcode if the instruction needs relaxation.
     Zero if the instruction is not relaxed.  */
  uint32_t	relax;
  struct
  {
    bfd_reloc_code_real_type type;
    expressionS              exp;
    int                      pc_rel;
    /* HACK_GUESS, force relocation entry to support scatteed loading */
    int                      pcrel_reloc;
  } reloc;

  struct
  {
    unsigned reg;
    signed int imm;
    struct neon_type_el vectype;
    unsigned present	: 1;  /* Operand present.  */
    unsigned isreg	: 1;  /* Operand was a register.  */
    unsigned immisreg	: 1;  /* .imm field is a second register.  */
    unsigned isscalar   : 1;  /* Operand is a (Neon) scalar.  */
    unsigned immisalign : 1;  /* Immediate is an alignment specifier.  */
    unsigned immisfloat : 1;  /* Immediate was parsed as a float.  */
    /* Note: we abuse "regisimm" to mean "is Neon register" in VMOV
       instructions. This allows us to disambiguate ARM <-> vector insns.  */
    unsigned regisimm   : 1;  /* 64-bit immediate, reg forms high 32 bits.  */
    unsigned isvec      : 1;  /* Is a single, double or quad VFP/Neon reg.  */
    unsigned isquad     : 1;  /* Operand is Neon quad-precision register.  */
    unsigned issingle   : 1;  /* Operand is VFP single-precision register.  */
    unsigned hasreloc	: 1;  /* Operand has relocation suffix.  */
    unsigned writeback	: 1;  /* Operand has trailing !  */
    unsigned preind	: 1;  /* Preindexed address.  */
    unsigned postind	: 1;  /* Postindexed address.  */
    unsigned negative	: 1;  /* Index register was negated.  */
    unsigned shifted	: 1;  /* Shift applied to operation.  */
    unsigned shift_kind : 3;  /* Shift operation (enum shift_kind).  */
  } operands[6];
};

// 下面是全局变量，在汇编时（函数 md_assemble 执行时）根据指令修改该结构
static struct arm_it inst;

// 修改过程如下：
inst.instruction = opcode->avalue;
if (opcode->tag == OT_unconditionalF)
	inst.instruction |= 0xF << 28;
else
	inst.instruction |= inst.cond << 28;
inst.size = INSN_SIZE;
```

## 汇编指令参数解析

### 寄存器类型及结构

```C
/* ARM register categories.  This includes coprocessor numbers and various
   architecture extensions' registers.	*/
enum arm_reg_type
{
  REG_TYPE_RN,
  REG_TYPE_CP,
  REG_TYPE_CN,
  REG_TYPE_FN,
  REG_TYPE_VFS,
  REG_TYPE_VFD,
  REG_TYPE_NQ,
  REG_TYPE_VFSD,
  REG_TYPE_NDQ,
  REG_TYPE_NSDQ,
  REG_TYPE_VFC,
  REG_TYPE_MVF,
  REG_TYPE_MVD,
  REG_TYPE_MVFX,
  REG_TYPE_MVDX,
  REG_TYPE_MVAX,
  REG_TYPE_DSPSC,
  REG_TYPE_MMXWR,
  REG_TYPE_MMXWC,
  REG_TYPE_MMXWCG,
  REG_TYPE_XSCALE,
};

/* Structure for a hash table entry for a register.
   If TYPE is REG_TYPE_VFD or REG_TYPE_NQ, the NEON field can point to extra
   information which states whether a vector type or index is specified (for a
   register alias created with .dn or .qn). Otherwise NEON should be NULL.  */
struct reg_entry
{
  const char        *name;
  unsigned char      number;
  unsigned char      type;
  unsigned char      builtin;
  struct neon_typed_alias *neon;
};

// 所有寄存器保存于全局数组中：
static const struct reg_entry reg_names[] =
{
  /* ARM integer registers.  */
  REGSET(r, RN), REGSET(R, RN),

  /* ATPCS synonyms.  */
  REGDEF(a1,0,RN), REGDEF(a2,1,RN), REGDEF(a3, 2,RN), REGDEF(a4, 3,RN),
  REGDEF(v1,4,RN), REGDEF(v2,5,RN), REGDEF(v3, 6,RN), REGDEF(v4, 7,RN),
  ......
}
```

### 寄存器的解析

该部分的解析在 parse_operands 函数中。具体来说，是根据 operand 的类型，使用 po_reg_or_fail 等宏定义及下面的函数完成解析。

```C
/* Like arm_reg_parse, but allow allow the following extra features:
    - If RTYPE is non-zero, return the (possibly restricted) type of the
      register (e.g. Neon double or quad reg when either has been requested).
    - If this is a Neon vector type with additional type information, fill
      in the struct pointed to by VECTYPE (if non-NULL).
   This function will fault on encountering a scalar.
*/

static int arm_typed_reg_parse (char **ccp, enum arm_reg_type type,
                     enum arm_reg_type *rtype, struct neon_type_el *vectype)；
```

该函数调用 parse_typed_reg_or_scalar 完成解析。

```C
/* Parse either a register or a scalar, with an optional type. Return the
   register number, and optionally fill in the actual type of the register
   when multiple alternatives were given (NEON_TYPE_NDQ) in *RTYPE, and
   type/index information in *TYPEINFO.  */

static int parse_typed_reg_or_scalar (char **ccp, enum arm_reg_type type,
                           enum arm_reg_type *rtype,
                           struct neon_typed_alias *typeinfo)
```

该函数首先调用 arm_reg_parse_multi 函数得到寄存器对应的结构体，再检查寄存器后是否为 ‘[’，是则解析，不是则修改 ccp 指向剩余的串，并返回寄存器的“数值” reg->number;

```C
/* Generic register parser.  CCP points to what should be the
   beginning of a register name.  If it is indeed a valid register
   name, advance CCP over it and return the reg_entry structure;
   otherwise return NULL.  Does not issue diagnostics.	*/

static struct reg_entry* arm_reg_parse_multi (char **ccp);
```

arm_reg_parse_multi 函数会在 **arm_reg_hsh** 哈希表中查找 *ccp 对应的寄存器结构体，找到则返回相应的 reg_entry。此外，还有如下函数：

```C
/* As arm_reg_parse_multi, but the register must be of type TYPE, and the
   return value is the register number or FAIL.  */
static int arm_reg_parse (char **ccp, enum arm_reg_type type);
```

该函数除调用 arm_reg_parse_multi 外，还会调用 arm_reg_alt_syntax，而 arm_reg_alt_syntax 为 “叶子”函数。



### 地址解析

地址的类型参下面注释所述：

```C
/* Parse all forms of an ARM address expression.  Information is written
   to inst.operands[i] and/or inst.reloc.

   Preindexed addressing (.preind=1):

   [Rn, #offset]       .reg=Rn .reloc.exp=offset
   [Rn, +/-Rm]	       .reg=Rn .imm=Rm .immisreg=1 .negative=0/1
   [Rn, +/-Rm, shift]  .reg=Rn .imm=Rm .immisreg=1 .negative=0/1
		       .shift_kind=shift .reloc.exp=shift_imm

   These three may have a trailing ! which causes .writeback to be set also.

   Postindexed addressing (.postind=1, .writeback=1):

   [Rn], #offset       .reg=Rn .reloc.exp=offset
   [Rn], +/-Rm	       .reg=Rn .imm=Rm .immisreg=1 .negative=0/1
   [Rn], +/-Rm, shift  .reg=Rn .imm=Rm .immisreg=1 .negative=0/1
		       .shift_kind=shift .reloc.exp=shift_imm

   Unindexed addressing (.preind=0, .postind=0):

   [Rn], {option}      .reg=Rn .imm=option .immisreg=0

   Other:

   [Rn]{!}	       shorthand for [Rn,#0]{!}
   =immediate	       .isreg=0 .reloc.exp=immediate
   label	       .reg=PC .reloc.pc_rel=1 .reloc.exp=label

  It is the caller's responsibility to check for addressing modes not
  supported by the instruction, and to set inst.reloc.type.  */

static parse_operand_result
parse_address_main (char **str, int i, int group_relocations, group_reloc_type group_type);
```

该函数在处理寄存器时会调用寄存器解析函数 arm_reg_parse、表达式解析函数 my_get_expression



### 表达式解析

在其它文件中直接调用的接口类函数如下：

```C
// read.c 中
/*
 * get_absolute_expression() gets an absolute expression and returns the value
 * of that expression.
 */
signed_target_addr_t get_absolute_expression(void);
```

```C
// arm.c 中
/* Prefix characters that indicate the start of an immediate value.  */
#define is_immediate_prefix(C) ((C) == '#' || (C) == '$')

static int my_get_expression (expressionS * ep, char ** str, int prefix_mode);
```

以上函数都会调用 expr.c 中的 expression 函数，定义如下，而在 expr.c 内部，实际由 expr 函数完成解析，其中 expr 会调用 operand 函数辅助解析。

```C
segT	/* Return resultP -> X_seg */
expression(
expressionS *resultP) /* deliver result here */
  
/* Expression parser. */

/*
 * We allow an empty expression, and just assume (absolute,0) silently.
 * Unary operators and parenthetical expressions are treated as operands.
 * As usual, Q==quantity==operand, O==operator, X==expression mnemonics.
 *
 * Most expressions are either register (which does not even reach here)
 * or 1 symbol. Then "symbol+constant" and "symbol-symbol" are common.
 *
 * After expr(RANK,resultP) input_line_pointer -> operator of rank <= RANK.
 * Also, we have consumed any leading or trailing spaces (operand does that)
 * and done all intervening operators.
 */
static
segT	/* Return resultP -> X_seg */
expr(
operator_rankT	rank, /* larger # is higher rank */
expressionS *resultP) /* deliver result here */
```

```C
/*
 * Summary of operand().
 *
 * in:	Input_line_pointer points to 1st char of operand, which may
 *	be a space.
 *
 * out:	A expressionS. X_seg determines how to understand the rest of the
 *	expressionS.
 *	The operand may have been empty: in this case X_seg == SEG_NONE.
 *	Input_line_pointer -> (next non-blank) char after operand.
 *
 */
static segT operand(expressionS *expressionP);
```

### 符号处理

每次碰到一个符号，会调用 symbol_find_or_make 函数

```C
/*
 *			symbol_find_or_make()
 *
 * If a symbol name does not exist, create it as undefined, and insert
 * it into the symbol table. Return a pointer to it.
 */
symbolS* symbol_find_or_make(char *name)；
```

所有符号的字符串保存于 notes，notes 的定义如下：

```C
/* FixS & symbols live here */
struct obstack notes = { 0 };

/*
 *			symbol_new()
 *
 * Return a pointer to a new symbol.
 * Die if we can't make a new symbol.
 * Fill in the symbol's values.
 * Add symbol to end of symbol chain.
 *
 *
 * Please always call this to create a new symbol.
 *
 * Changes since 1985: Symbol names may not contain '\0'. Sigh.
 */
symbolS *
symbol_new(
char	       *name,	/* We copy this: OK to alter your copy. */
unsigned char	type,	/* As in <nlist.h>. */
char		other,	/* As in <nlist.h>. */
short		desc,	/* As in <nlist.h>. */
valueT		value,	/* As in <nlist.h>, often an address. */
			/* Often used as offset from frag address. */
struct frag    *frag)	/* For sy_frag. */
```

对于新的符号，先将 符号的字符串保存于 notes，之后在 notes 中创建该符号对应的 symbol 结构体，并将其加入到哈希表 sy_hash 中。

如果是标号（参函数 colon），则创建新标号的 symbol，之后插入到符号表，并将 symbol_lastIndexedP 指向该符号。



### 移位解析

```C
/* Parse a <shifter_operand> for an ARM data processing instruction
   (as for parse_shifter_operand) where group relocations are allowed:

      #<immediate>
      #<immediate>, <rotate>
      #:<group_reloc>:<expression>
      <Rm>
      <Rm>, <shift>

   where <group_reloc> is one of the strings defined in group_reloc_table.
   The hashes are optional.

   Everything else is as for parse_shifter_operand.  */

static parse_operand_result parse_shifter_operand_group_reloc (char **str, int i);

/* Parse a <shifter_operand> for an ARM data processing instruction:

      #<immediate>
      #<immediate>, <rotate>
      <Rm>
      <Rm>, <shift>

   where <shift> is defined by parse_shift above, and <rotate> is a
   multiple of 2 between 0 and 30.  Validation of immediate operands
   is deferred to md_apply_fix.  */

static int parse_shifter_operand (char **str, int i);

```





BFD_RELOC_ARM_OFFSET_IMM 19 str    fp, [sp, #-4]!

BFD_RELOC_ARM_ADRL_IMMEDIATE 19 

BFD_RELOC_ARM_IMMEDIATE 17  add    fp, sp, #0

sub    sp, sp, #28     BFD_RELOC_ARM_IMMEDIATE 17 

mov    r3, #2            BFD_RELOC_ARM_IMMEDIATE 17 

beq    .L4                BFD_RELOC_ARM_PCREL_BRANCH 5       inst.reloc.pc_rel = 1;

add    r3, r2, r3      BFD_RELOC_ARM_SHIFT_IMM 20

.word a               r_type  0 

# ARM指令格式与Fixup

```C
|31      28|27 26| |24         21|20|19   16|15      12|11                             0|
|---------------------------------------------------------------------------------------|
|   Cond   | 0  0|I|   opcode    | S|   Rn  |    Rd    |            Operand2            |
|---------------------------------------------------------------------------------------|
```

* 每条ARM指令占有4个字节，其指令长度为32位
* cond（bit[31:28]） 指令执行的条件码
* type（bit[27:26]） 指令类型码
* opcode（bit[24:21]） 指令操作码
* S  （bit[20]） 决定指令的操作结果是否影响CPSR 
* Rn （bit[19:16]） 包含第一个操作数的寄存器编码
* Rd （bit[15:12]） 目标寄存器编码
* Operand2（bit[11:0） 指令第二个操作数
* I 为 1表示 Operand2 是立即数； I 不 0 表示Operand2是移位寄存器

```C
// 类型为 BFD_RELOC_ARM_IMMEDIATE (值为 17）时的处理 
// 示例指令：sub sp, sp, #28     
//         sub sp, fp, #4
  
newimm = encode_arm_immediate (value);
temp = md_chars_to_number (buf, INSN_SIZE);
newimm |= (temp & 0xfffff000);
md_number_to_chars (buf, (valueT) newimm, INSN_SIZE);
```



**Single Data Transfer (LDR, STR)**

```C
|31      28|27 26|25|24|23|22|21|20|19   16|15      12|11                             0|
|--------------------------------------------------------------------------------------|
|   Cond   | 0  1| I| P| U| B| W| L|   Rn  |    Rd    |            Operand2            |
|--------------------------------------------------------------------------------------|

// The offset may be added to (U=1) or subtracted from (U=0) the base register Rn. 
// 类型为 BFD_RELOC_ARM_ADRL_IMMEDIATE / BFD_RELOC_ARM_LITERAL md_apply_fix  (值为 19）时的处理 
// 示例指令：ldr	r3, [fp, #-8]  
#define INDEX_UP	0x00800000

sign = value >= 0;
if (value < 0)
	 value = - value;

newval = md_chars_to_number (buf, INSN_SIZE);
// 0xff7ff000 的二进制形式：1111 1111 0111 1111 1111 0000 0000 0000
//                            上面的 0 刚好是第23位，即 U，根据手册 U 为 1 表示加，为 0 表示减
// 0x00800000 的二进制形式：0000 0000 1000 0000 0000 0000 0000 0000
//                        只保留了第 23 位
// 所以这里将 23 位空出来，值由 value 决定，加减由 sign 决定
newval &= 0xff7ff000;
newval |= value | (sign ? INDEX_UP : 0);
md_number_to_chars (buf, newval, INSN_SIZE);
```





```C
|31      28|27    25|24|23                                                            0|
|--------------------------------------------------------------------------------------|
|   Cond   | 1  0  1| L|                         offset                                |
|--------------------------------------------------------------------------------------|
// L: 0 = Branch; 1 = Branch with Link;
// 示例指令: bl	myFunc
/* For ARM, PC-relative fixups applied to instructions
   are generally relative to the location of the fixup plus 8 bytes.
   Thumb branches are offset by 4, and Thumb loads relative to PC
   require special handling.  */
// 所以在函数返回时有：return (int32_t)base + 8;
int32_t md_pcrel_from_section (fixS * fixP, segT seg);

newval = md_chars_to_number (buf, INSN_SIZE);
// shifted left two bits, 24 位的偏移值
newval |= (value >> 2) & 0x00ffffff;
```

为什么 +8 呢？在手册中有如下描述：

> Branch instructions contain a signed 2’s complement 24 bit offset. This is **shifted left two bits**, sign extended to 32 bits, and added to the PC. The instruction can therefore specify a branch of +/- 32Mbytes.
>
> The branch offset must **take account of the prefetch operation**, which causes the PC to be 2 words (8 bytes) ahead of the current instruction.









# 待整理

 

Obstack初始化：

\#include<obstack.h>

int obstack_init(struct obstack *obstack-ptr)

 

 

　　obstack_init实际上是一个宏实现。obstack_init会自动调用obstack_chunk_alloc函数来申请chunk，注意这还是个宏，需要你指定这个宏指向的函数。如果内存申请失败了会调用obstack_alloc_failed_handler指向的函数，没错依旧是宏。如果chunk里面的对象都被释放了，obstack_chunk_free指向的函数被用来返回chunk占用的空间。目前版本的obstack_init永远只返回1（之前的版本会在失败的时候返回0）

实例:

 

\#include<obstack.h>

\#include<stdlib.h>

\#define obstack_chunk_alloc malloc

\#define obstack_chunk_free free

 

//静态申请obstack

static struct obstack myobstack;

obstack_init(&myobstack);

 

//动态申请obstack

struct obstack *myobstack_ptr = (struct obstack*) malloc (sizeof (struct obstack))

obstack_init(myobstack_ptr)




 　　obstack chunk的大小默认情况下为4096。如果你想自定义chunk的大小，需要使用宏obstack_chunk_size

int obstack_chunk_size (struct obstack* obstack-ptr)

 　注意这实际上是个左值（C++里面有麻烦的左值引用，右值引用 囧rz）。obstack-ptr如之前的myobstack所示，指明了是哪个obstack。如果你适当的提高chunk的大小，有利于提高效率。减小则没啥好处。。。

if (obstack_chunk_size (myobstack_ptr) < new-chunk-size)

  obstack_chunk_size (myobstack_ptr) = new-chunk-size

 

在Obstack里面申请对象：

 　最直接的方法使用aobstack_alloc函数

void * obstack_alloc (struct obstack *obstack-ptr, int size)

 

　　调用方式和malloc差不多, 对新申请的空间不初始化。如果chunk被object填满了，obstack_alloc会自动调用obstack_chunk_alloc。

 

struct obstack string_obstack;

/*

....

初始化内容

....

*/

char * copystring (char *string)

{

  size_t len = strlen (string) + 1;

  char *s = (char*) obstack_alloc (&string_obstack, len);

  memcpy (s, string , len);

  return s;

}

 

 

　　如果想在申请时初始化，需要用到obstack_copy或者obstack_copy0

void * obstack_copy (struct obstack *obstack-ptr, void *address, int size)

//使用从地址address开始复制size大小的数据初始化，新申请的object内存大小也为size

 

void *obstack_copy0 (struct obstack *obstack-ptr, void *address, int size)

//同obstack_copy但结尾处额外添加一个空字符，在复制以NULL结尾的字符串时很方便。

 

 

　　释放Obstack里面的对象：

　　同样很简单，我们使用函数obstack_free

void obstack_free (struct obstack *obstack-ptr, void *object)

 

　　如果object是空指针的话，所有的对象都被释放，留下一个未初始化的obstack,然后obstack库会自动释放chunk。如果你指定了object地址，那自这个object之后的所有对象都被释放。值得注意的是：如果你想释放所有object但同时保持obstack有效，可以指定第一个object的地址

obstack_free (obstack_ptr, first_object_allocated_ptr);

 

　　

可增长内存空间的对象：

　　由于obstack chunk的内存空间是连续的，在其中的对象空间可以一步步搭建一直增加到结尾。这种技术称之为growing object。尽管obstack可以使用growing object，但是你无法为已申请好的对象使用这项技术（理由是如果这对象之后还有对象，会造成冲突）

 

void obstack_blank (struct obstack *obstack-ptr, int size)

//添加一个为初始化的growing object

 

void obstack_grow (struct obstack *obstack-ptr, void *data, int size)

//添加一个初始化的空间，类似与obstack_copy

 

void obstack_grow0 (struct obstack *obstack-ptr, void *data, int size)

//obstack_grow类似与obstack_copy, 那obstack_grow0你说类似与谁？

 

void obstack_lgrow (struct obstack *obstack-ptr, charc)

//一次添加一个字符（字节）

 

void  obstack_ptr_grow (struct obstack *obstack-ptr, void *data)

//一次添加一个指针。。。大小为（sizeof(void *)

 

void obstack_int_grow (struct obstack *obstack-ptr, int data)

//一次添加一个int型数据，大小为sizeof(int)

 

　　在growing object中最重要的函数是obstack_finish, 因为只有调用了本函数之后才会返回growing object 的最终地址。

void *obstack_finish (struct obstrack *obstrack-ptr)

 

　　如果你想获取growing object当前大小，可以使用obstack_object_size

int obstack_object_size (struct obstack *obstrack-ptr)

//只能用在obstack_finish之前，否则只会返回0

 

　　如果你想取消一个正在增长的对象，必须先结束他，然后再释放。示例如下：

obstack_free (obstack_ptr, obstack_finish (obstack_ptr))

 

　　上面的函数在添加数据时会检查是否由足够的空间。如果你只增加一丁点的空间，检查这一步骤显然是多余的。我们可以省略这一步骤，更快的growing。这里关于growing object额外在介绍一个函数：

int obstack_room (struct obstack *obstack-ptr)

//返回可以安全添加的字节数

 

　　其余的快速添加函数只需要在之前的growing object函数添加后缀_fast即可（如果不清楚可以查看GNU C 关于这部分的介绍）

 

Obstack 的状态：

以下函数用于获取obstack的状态：

void * obstack_base (struct obstack *obstack-ptr)

//返回正在增长的对象的假定起始地址。为啥是假定呢，因为如果你增长的过大，当前chunk的空间不够，obstack就会新创建一个chunk，这样地址就变了。

 

 

void *obstack_next_free (struct obstack *obstack-ptr)

//返回当前chunk未被占用的第一个字节的地址

 

int obstack_object_size (struct obstack *obstack-ptr)

//返回当前growing object的大小。等同与：

//obstack_next_free (obstack-ptr) -obstack_base (obstack-ptr)

 

Obstack 里的数据对齐问题

int obstack_alignment_mask (struct obstack *obstack-ptr)

 

　　宏展开后是一个左值。如果是函数实现，返回的是掩码。你给他复制也应该是掩码。掩码的值应该是2的n次方减一，换算后也就是说地址必须是2的n次方的倍数。如果你改变了掩码，只有下次申请object的时候才会生效（特例是growing object：立即生效，调用obstack_finish后就能看到效果了）

 本文地址http://www.cnblogs.com/san-fu-su/p/5739780.html

 











 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 