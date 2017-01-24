---
title: PHP7 opcode查看器
date: 2015-12-18 19:12:50
tags:
---

如果要进一步理解PHP的运行原理，就涉及到了PHP的opcode：opcode是一种中间代码，和Java的ByteCode或者.NET的MSL类似。

从底层来看，PHP执行的基本原理是： 读取php文件 –> 编译为中间代码opcode –> 执行。这个模式在PHP5与PHP7之间并没有变化，我就不在这班门弄斧了，有兴趣的同学可以看鸟哥的深入理解PHP原理之Opcodes和Sara Golemon大师的Understanding Opcodes。

有了上面的基础，好奇的同学肯定对PHP代码会编译成什么样的Opcode感兴趣了，怎么能看到opcode呢？首先，可以通过调试源码的方式看到。调试的方法在PHP7源码的编译与调试有介绍，将断点打在Zend/zend.c的zend_execute_scripts里，其中的zend_compile_file(file_handle, type)是编译，zend_execute(op_array, retval)是执行（在PHP-7.0.0中为zend.c的1422行和1428行，见php-src/Zend/zend.c），zend_compile_file返回的op_array就是opcode数组了。不过这样看太痛苦了，PHP有一个很好的opcode查看器VLD，通过扩展的方式显示opcode的分析结果，参考作者的说明可以很方便的看到opcode。

vld集成了很多功能，输出的内容也很多，opcode到底是怎么实现的并不容易看懂。为了更直观的理解opcode，我写了一个简单的PHP7扩展opcode查看器。

这是一个简单php扩展，除去PHP扩展必须的代码,只有100多行，应该很容易理解。基本原理是这样：

使用PHP_RINIT_FUNCTION，在Request init的时候将php的zend_compile_file函数替换为自己的opdump_compile_file函数。
opdump_compile_file函数调用php的zend_compile_file函数，翻译、存储返回结果op_array后，将op_array返回。
PHP7的扩展与PHP5的扩展创建与使用方法相同，可以参考鸟哥的文章用C/C++扩展你的PHP。介绍一下opdump的原理：

zend_compile_file返回一个zend_op_array，定义在zend_compile.h，比较长：

```c
struct _zend_op_array {
    /* Common elements */
    zend_uchar type;
    zend_uchar arg_flags[3]; /* bitset of arg_info.pass_by_reference */
    uint32_t fn_flags;
    zend_string *function_name;
    zend_class_entry *scope;
    zend_function *prototype;
    uint32_t num_args;
    uint32_t required_num_args;
    zend_arg_info *arg_info;
    /* END of common elements */

    uint32_t *refcount;

    uint32_t this_var;

    uint32_t last;
    zend_op *opcodes;

    int last_var;
    uint32_t T;
    zend_string **vars;

    int last_brk_cont;
    int last_try_catch;
    zend_brk_cont_element *brk_cont_array;
    zend_try_catch_element *try_catch_array;

    /* static variables support */
    HashTable *static_variables;

    zend_string *filename;
    uint32_t line_start;
    uint32_t line_end;
    zend_string *doc_comment;
    uint32_t early_binding; /* the linked list of delayed declarations */

    int last_literal;
    zval *literals;

    int  cache_size;
    void **run_time_cache;

    void *reserved[ZEND_MAX_RESERVED_RESOURCES];
};
```

opcode存放在opcodes数组中，每行是一句code， last是code的行数。每行code是zend_op:

```c
struct _zend_op {
    const void *handler;
    znode_op op1;          //操作数1
    znode_op op2;          //操作数2
    znode_op result;       //返回值
    uint32_t extended_value;
    uint32_t lineno;       //行号
    zend_uchar opcode;     //操作类型
    zend_uchar op1_type;  //操作数1类型
    zend_uchar op2_type;  //操作数2类型
    zend_uchar result_type;  //返回值类型
};
```
其中opcode是操作类型，定义在Zend/zend_vm_opcodes.h，op1_type、op2_type、result_type定义在zend_compile.h:

```c
#define IS_CONST    (1<<0)
#define IS_TMP_VAR  (1<<1)
#define IS_VAR      (1<<2)
#define IS_UNUSED   (1<<3)  /* Unused variable */
#define IS_CV       (1<<4)  /* Compiled variable */

#define EXT_TYPE_UNUSED (1<<5)
```

使用内置的zend_get_opcode_name显示opcode，再将op1，op2，result稍作处理显示出来。

这样，一个简易的opdump就完成了，代码中将opcode输出到了php文件相同目录下的filename.php.opcodes文件中。测试一个简单的php文件：

```php
$a = "hello world";
echo $a;
```
输出的opcode：

``` 
ZEND_ASSIGN CV+96   string:hello world  unknown
ZEND_ECHO   CV+96   unused  unused
ZEND_RETURN long:1  unused  unused
```
CV表示compiled varaible, +96表示偏移量。上面的意思是把string:hello world赋值到CV+96的位置，再对CV+96调用echo，最后返回long:1。

实现：https://github.com/zanexie/php7-opdump 

如果要更进一步的了解opcode，可以参考vld的代码。
