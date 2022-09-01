---
title: 重新定义Hello World
date: 2022-08-20 11:26:14
tags:
---

目录
- 原始的 Hello World 程序
- 配置开发环境 
- 单元测试
- 代码实现
- 代码调试
- 安全测试 fuzz
- 性能测试
- 性能剖析
- 性能优化


### 原始的 Hello World

原始的 Hello World 程序是 Brian Kernighan 和 Dennis Ritchie(K&R) 在 1978 年为《The C Programming Language》一书写的例子，后来成为了所有程序员的入门必学的例子。


    #include <stdio.h>

    main()
    {
        printf("hello, world\n");
    }

可以看到，这段代码和现在的 C 代码有很大的差别，main 函数没有参数也没有返回值，因为那会 C 语言的规范还没有确定，硬件也比较受限。而且原始的 hello world 程序全是小写，hello 后面有个逗号，但结尾没有句号，这应该是随手示例的。

要编译这段古老的代码，我们要使用 C90 标准：

    $ gcc -std=gnu90 k-r-hello-world.c
    $ ./a.out
    hello, world


参考链接：
- [Original "Hello, World!" in K&R C](https://riptutorial.com/c/example/3675/original--hello--world---in-k-r-c)

### 配置开发环境

工欲善其事必先利其器，假设我们使用 VIM 开发 C 语言，首先要在 `~/.vimrc` 里设置一些简单的配置:

    set nocp nu et ts=4 sw=4 sta hls si noeb vb t_vb=
    autocmd FileType c nnoremap <buffer> <F5> :!gcc -g -Wall  % && ./a.out<CR>

第一行是通用的设置，要使用 VIM 写代码我一般都会先设置它，就一行也很好好记

- `nocp`: 让 VIM 不在兼容模式下运行 
- `nu`: 显示行号
- `et`: expandtab 的缩写，表示输入 tab 时插入空格
- `ts=4`: tabstop 的缩写，输入 tab 插入 4 个空格
- `sw=4`: shiftwidth 缩写，用shift+>> 时调整缩进时移动 4 个空格
- `sta`: smarttab 的缩写，基于已有行的缩进来确定，在新行的开始位置
- `hls`: hlsearch 的缩写，高亮显示搜索的关键字 
- `si`: smartindent 的缩写，每一行都和前一行有相同的缩进量
- `noeb` vb t_vb=: 关闭按错键时的响铃声和屏幕闪烁，否则太烦了

第二行是设置当文件是 C 语言文件时，在正常模式下按 F5 键会编译和运行程序，相当于一个快速运行快捷键。

- `-g` 是开启调试模式，会生成调试符号，出错时的错误信息能看到行号。
- `-Wall` 是开启所有警告，以最严格的模式来写代码，养成好的习惯。

写一个新的 `hello-world.c` 再按 F5 试试：

    #include <stdio.h>

    int main()
    {
        printf("hello world.\n");
        return 0;
    }

好了，现在我们就可以快乐的写代码了，如果你记不住 VIM 的配置，那可以简化一下：

    set nu et ts=4

然后新开一个窗口进行编译和测试

    gcc hello-world.c && ./a.out

还有一个小技巧是写 C 代码的时候，有时候需要查看函数的帮助，可以将光标移动到函数上，按 `shit+k` 查看帮助，按 q 退出，这是 VIM 内置的。

一个好的开发工具往往会事半功倍，自己试一下在自己熟悉的编程工具里如何完成如下任务：
- 跳转到上一次编辑的位置 
- 跳转到一个函数的定义
- 查找函数的所有引用
- 跳到匹配括号的开始和结尾
- 快速注释和取消注释一行或多行

### 单元测试

下面我们写一个有意义的程序，把一个路径和一个文件名连接在一起，比如把 `/tmp` 和 `girl.jpg` 连接在一起就是 `/tmp/girl.jpg`。

大家一定听过 TDD 测试驱动，就是写代码之前先写测试，但写测试前得先有测试框架，我们可以使用 CMocka，原因是简单够用，下面是安装过程：

    git clone --single-branch --depth 1 https://git.cryptomilk.org/projects/cmocka.git
    cd cmocka
    mkdir build && cd build
    cmake ..
    make -j8
    sudo make install

编写一个 `cmocka-test.c` 文件，测试下官方例子：

    #include <stdarg.h>
    #include <stddef.h>
    #include <setjmp.h>
    #include <stdint.h>
    #include <cmocka.h>

    /* A test case that does nothing and succeeds. */
    static void null_test_success(void **state) {
        (void) state; /* unused */
    }

    int main(void) {
        const struct CMUnitTest tests[] = {
            cmocka_unit_test(null_test_success),
        };

        return cmocka_run_group_tests(tests, NULL, NULL);
    }

这时候按 F5 是无法编译了，大概会看到这样的错误：

    /tmp/cc1roVOc.o: In function `main':
    cmocka-test.c:(.text+0x6f): undefined reference to `_cmocka_run_group_tests'
    collect2: error: ld returned 1 exit status

这因为我们编译的时候没有链接 `cmocka` 的库，所以要修改下 `~/.vimrc` 的配置

    autocmd FileType c nnoremap <buffer> <F5> :!gcc -g -Wall %  -lcmocka && ./a.out<CR>

这时候按 F5 应该能够看到单元测试的输出，可以看到运行了一个测试，并且通过了。

    [==========] tests: Running 1 test(s).
    [ RUN      ] null_test_success
    [       OK ] null_test_success
    [==========] tests: 1 test(s) run.
    [  PASSED  ] 1 test(s).
        
如果运行程序的时候报如下的错误：

    ./a.out: error while loading shared libraries: libcmocka.so.0: cannot open shared object file: No such file or directory

这是因为运行时找不到动态链接库，我们要修改下环境变量来解决，在 shell 里修改 `LD_LIBRARY_PATH` 并重新用 VIM 打开代码运行：

    export LD_LIBRARY_PATH=/usr/local/lib/:$LD_LIBRARY_PATH

要永久修改共享库查找路径的话可以在 `/etc/ld.so.conf` 里指定。

好了，单元测试库是可以用了，我们就可以写代码了，写代码之前我们还要先定义接口，就是函数的签名，参数和返回值。
本着接口和实现分离的原则，我们一般会把函数声明放到头文件里，新建一个 `join-path.h` 文件：

    int join_path(char *buf, int buflen, const char *base, const char *file);

我们做如下约定

- 函数名是 `join_path`
- `buf` 是拼接结果的缓冲区
- `buflen` 是缓冲区长度，防止拼接后缓冲区溢出
- `base` 是目录的路径，如 `/tmp`
- `file` 是文件名，如 `girl.jpg`
- 返回值为 int，0 表示成功，非 0 表示出错，一般 unix 下是这样约定的。

新建代码实现文件 `join-path.c`，先写个空的实现函数，没有实际逻辑：

    #include <stdio.h>

    int join_path(char *buf, int buflen, const char *base, const char *file) {
        return 0;
    }

新建 `join-path-test.c` 文件来写测试代码

    #include <stdarg.h>
    #include <stddef.h>
    #include <setjmp.h>
    #include <stdint.h>
    #include <cmocka.h>
    #include "join-path.h"
    #include "join-path.c"

    static void join_test01(void **state) {
        char buf[100];
        int ret = join_path(buf, 100, "/tmp", "girl.jpg");
        assert_int_equal(0, ret);
        assert_string_equal("/tmp/girl.jpg", buf);
    }

    int main(void) {
        const struct CMUnitTest tests[] = {
            cmocka_unit_test(join_test01),
        };

        return cmocka_run_group_tests(tests, NULL, NULL);
    }

大部分代码和 CMocka 的实例代码是一样的，不同的地方是：
- include 了被测代码的头文件和实现文件，因为我们是测试自己的代码，可以直接引入源码文件，这样 F5 就可以看到测试结果, 不需要修改链接选项。
- 定义了一个 join_test01 的函数，相当于第一个测试用例：
    - 定义一个长度 为 100 个 buffer；
    - 调用 join_path 函数；
    - 断言函数返回 0；
    - 断言缓冲区为预期的结果 `/tmp/girl.jpg`。
- 在 main 函数里执行上面定义的测试。

好了，接口声明，代码实现和第一个单元测试都有了，我们 F5 来运行测试，预期会失败，但我们后面再让他成功。

    [==========] tests: Running 1 test(s).
    [ RUN      ] join_test01
    111
    [  ERROR   ] --- "/tmp/girl.jpg" != ""
    [   LINE   ] --- join-path-test.c:13: error: Failure!
    [  FAILED  ] join_test01
    [==========] tests: 1 test(s) run.
    [  PASSED  ] 0 test(s).
    [  FAILED  ] tests: 1 test(s), listed below:
    [  FAILED  ] join_test01

     1 FAILED TEST(S)

在 VIM 同时打开多个文件的话，可以按 `:bn` 和 `:bp` 进行切换，比如我们就打开了 3 个文件：`join-path.h`，`join-path.c` 和 `join-path-test.c`。

我们要对单元测试进行更多的熟悉，尝试做如下任务：
- 了解更多的断言，如断言数组中包含指定元素；
- 在每个单元测试前面进行准备工作，后面做资源回收工作；
- 对被测函数依赖的函数进行 mock；
- 断言某个函数以指定的参数被调用；
- 临时屏蔽某个测试用例；
- 评估被测代码的测试覆盖率；

### 代码实现

终于可以正式的开始写代码实现了，第一个可以想到的就是用 `snprintf` 函数实现：

    int join_path(char *buf, int buflen, const char *base, const char *file) {
        snprintf(buf, buflen, "%s/%s", base, file);
        return 0;
    }

F5 运行测试

    [==========] tests: Running 1 test(s).
    [ RUN      ] join_test01
    [       OK ] join_test01
    [==========] tests: 1 test(s) run.
    [  PASSED  ] 1 test(s).

妥了，测试通过了，喝杯咖啡奖励一下自己。接着做更多的测试，写第 2 个用例，并在 main 函数里增加一行以便测试它：

    static void join_test02(void **state) {
        char buf[3];
        int ret = join_path(buf, 3, "/tmp", "girl.jpg");
        assert_int_equal(0, ret);
        assert_string_equal("/tmp/girl.jpg", buf);
    }

    int main(void) {
        const struct CMUnitTest tests[] = {
            cmocka_unit_test(join_test01),
            cmocka_unit_test(join_test02),
        };

        return cmocka_run_group_tests(tests, NULL, NULL);
    }

F5 测试，发现这次用例失败了：

    [==========] tests: Running 2 test(s).
    [ RUN      ] join_test01
    [       OK ] join_test01
    [ RUN      ] join_test02
    [  ERROR   ] --- "/tmp/girl.jpg" != "/t"
    [   LINE   ] --- join-path-test.c:20: error: Failure!
    [  FAILED  ] join_test02
    [==========] tests: 2 test(s) run.
    [  PASSED  ] 1 test(s).
    [  FAILED  ] tests: 1 test(s), listed below:
    [  FAILED  ] join_test02

     1 FAILED TEST(S)

因为这次的缓冲区长度我们故意给了 3 个，根本放不下拼接后的结果，所以我们预期这时候返回值不应该为 0 ，我们修改下测试用例:

    static void join_test02(void **state) {
        char buf[3];
        int ret = join_path(buf, 3, "/tmp", "girl.jpg");
        assert_int_not_equal(0, ret);
    }

这时候运行测试肯定还是失败，我们的主要工作流程就是：
- 编写新的单元测试，默认执行会失败
- 修改代码，让失败的单元测试成功

上面的用例失败，说明我们没有对参数做严格的检查，我们来完完善它：
    
    #include <string.h>

    int join_path(char *buf, int buflen, const char *base, const char *file) {
        if (strlen(base)+strlen(file)+1+1>buflen) return -1;
        snprintf(buf, buflen, "%s/%s", base, file);
        return 0;
    }

我们在 base 加上 file 再加上路径连接符 `/` 再加上 C 语言字符串终止符 `0` 的长度大于缓冲区长度时返回 -1。F5 测试：

    [==========] tests: Running 2 test(s).
    [ RUN      ] join_test01
    [       OK ] join_test01
    [ RUN      ] join_test02
    [       OK ] join_test02
    [==========] tests: 2 test(s) run.
    [  PASSED  ] 2 test(s).

妥了， 2 个测试用例成功了，但我们不能靠假设编程，每个参数都会有各种可能的值传进来，还有别的边界条件要测吗？想一想：

- 如果参数 buf, base, file 有一个是 NULL 程序能否正常运行，应如何修复。
- 如果参数 buflen 是负数，程序能否正常运行，应如何修复。
- 如果参数 base 或 file 不是一个以 `\0` 结尾的字符串，会发生什么，应如何修复。
- 如何缓冲区的大小并没有 buflen 那么长，会发生什么，我们能修复吗？


### 代码调试

写 C 程序最常遇到的错误是什么？那肯定是内存错误，因为 C 语言里内存都要手动管理，各种数组也没有越界检查，一不小心就会出现内存错误。

我们新建 `join-path-debug.c` 来学习如何调试代码：

    #include <stdio.h>
    #include "join-path.c"
    int main()
    {
        char buf[3];
        join_path(buf, 100, "/tmp", "girl.jpg");
        return 0;
    }

手工编译并运行，直接报错了，类型是调用栈错误，也是一种典型的内存错误，其它常见的还有段错误等。 

    $ gcc join-path-debug.c
    $ ./a.out
    *** stack smashing detected ***: <unknown> terminated
    Aborted (core dumped)

如果我们的代码量很大，执行程序异常崩溃，我们如何定位原因呢？这就得用到 gdb 调试工具了。首先编译时要加 -g 参数

    gcc -g join-path-debug.c

然后用 gdb 来运行它：

    
    $ gdb ./a.out
    GNU gdb (Ubuntu 8.1.1-0ubuntu1) 8.1.1
    Reading symbols from ./a.out...done.

输入 r 会运行程序，这里会看到遇到了内存错误，程序被终止了。

    (gdb) r
    Starting program: a.out
    *** stack smashing detected ***: <unknown> terminated

    Program received signal SIGABRT, Aborted.
    __GI_raise (sig=sig@entry=6) at ../sysdeps/unix/sysv/linux/raise.c:51
    51      ../sysdeps/unix/sysv/linux/raise.c: No such file or directory.

输入 bt 可以查看线程调用栈, 程序是被 libc 的栈检查机制终止的，最顶层的栈帧是 join-path-debug.c 的第 8 行 main 函数里引起的错误

    (gdb) bt
    #0  __GI_raise (sig=sig@entry=6) at ../sysdeps/unix/sysv/linux/raise.c:51
    #1  0x00007ffff7a227f1 in __GI_abort () at abort.c:79
    #2  0x00007ffff7a6b837 in __libc_message (entry=do_abort, fmt=fmt@entry=0x7ffff7b98869 "*** %s ***: %s terminated\n") at ../sysdeps/posix/libc_fatal.c:181
    #3  0x00007ffff7b16b31 in __GI___fortify_fail_abort (need_backtrace=need_backtrace@entry=false, msg=msg@entry=0x7ffff7b98847 "stack smashing detected") at fortify_fail.c:33
    #4  0x00007ffff7b16af2 in __stack_chk_fail () at stack_chk_fail.c:29
    #5  0x00005555555547cc in main () at join-path-debug.c:8

用 `f 5` 切换栈帧，并用 `l` 查看附近的代码

    (gdb) f 5
    #5  0x00005555555547cc in main () at join-path-debug.c:8
    8       }
    (gdb) l
    3       int main()
    4       {
    5           char buf[3];
    6           join_path(buf, 100, "/tmp", "girl.jpg");
    7           return 0;
    8       }

我们发现第 8 行代码并没有实际代码执行，怎么会出错呢，这其实是正常的。C 程序遇到的好多错误就是这样，出错的地点和引起错误的地点有可能离的很远，因为引起内存错误的代码可能执行了很久，才会被其它的代码使用这块内存的时候发现。

现在我们看到的例子是个特例，因为在第 8 行往上看一下就能找到问题：代码定义的缓冲区长度只有 3， 但给 join_path 传入的 buflen 参数是 100，所以 join_path 的参数检查并没有失败，这时候运行就会报错了。

那我们还有没有更可靠的方法来定位这种错误呢？有，那就是用 Address Sanitizer（Asan），这个工具已经集成到 gcc 里了，我们直接加几个参数就可以使用：


    $ gcc -g -Wall -fsanitize=address -fno-omit-frame-pointer join-path-debug.c
    $ ./a.out
    =================================================================
    ==13565==ERROR: AddressSanitizer: stack-buffer-overflow on address 0x7fff97c3b743 at pc 0x7fb8cbce4f09 bp 0x7fff97c3b5d0 sp 0x7fff97c3ad60
    WRITE of size 14 at 0x7fff97c3b743 thread T0
        #0 0x7fb8cbce4f08 in __interceptor_vsnprintf (/usr/lib/x86_64-linux-gnu/libasan.so.4+0xa0f08)
        #1 0x7fb8cbce5286 in snprintf (/usr/lib/x86_64-linux-gnu/libasan.so.4+0xa1286)
        #2 0x55bb3a82cb80 in join_path join-path.c:6
        #3 0x55bb3a82cc2c in main join-path-debug.c:6
        #4 0x7fb8cb874c86 in __libc_start_main (/lib/x86_64-linux-gnu/libc.so.6+0x21c86)
        #5 0x55bb3a82ca29 in _start (a.out+0xa29)

    Address 0x7fff97c3b743 is located in stack of thread T0 at offset 35 in frame
        #0 0x55bb3a82cb9c in main join-path-debug.c:4

      This frame has 1 object(s):
        [32, 35) 'buf' <== Memory access at offset 35 overflows this variable

这次妥了，直接定位到 join-path.c 的第 6 行触发了栈溢出，而且还显示这块内存是在 join-path-debug.c 的第4 行在栈上分配的，有了这些信息候再结合代码和调用栈就很容易定位错误了。

如果还是没有定位到错误，因为一些运行时的信息不确定，那可以继续用 gdb 来调试，去掉 asan 编译选项重新编译程序并用 gdb 启动

    $ gcc -g -Wall join-path-debug.c
    $ gdb ./a.out

根据 asan 的信息，在 join-path.c 的第 6 行增加断点：

    (gdb) b join-path.c:6
    Breakpoint 1 at 0x745: file join-path.c, line 6.

运行程序，可以看到在断点处停住了，可以很清楚的看到各个参数的值：

    (gdb) r
    Starting program: a.out

    Breakpoint 1, join_path (buf=0x7fffffffe385 "\177", buflen=100, base=0x555555554863 "/tmp",
        file=0x55555555485a "girl.jpg") at join-path.c:6
    6           snprintf(buf, buflen, "%s/%s", base, file);

也可以用 p 命令查看具体某个变量的值

    (gdb) p base
    $1 = 0x555555554863 "/tmp"
    (gdb) p buflen
    $2 = 100
    (gdb) p file
    $3 = 0x55555555485a "girl.jpg"
    (gdb) p buf
    $4 = 0x7fffffffe385 "\177"

这里看到 buflen 为 100，我们尝试跳到上一层堆栈找找有没有 buf 分配的代码：
    
    (gdb) bt
    #0  join_path (buf=0x7fffffffe385 "\177", buflen=100, base=0x555555554863 "/tmp", file=0x55555555485a "girl.jpg")
        at join-path.c:6
    #1  0x00005555555547b3 in main () at join-path-debug.c:6
    (gdb) f 1
    #1  0x00005555555547b3 in main () at join-path-debug.c:6
    6           join_path(buf, 100, "/tmp", "girl.jpg");
    (gdb) l
    1       #include <stdio.h>
    2       #include "join-path.c"
    3       int main()
    4       {
    5           char buf[3];
    6           join_path(buf, 100, "/tmp", "girl.jpg");
    7           return 0;
    8       }

这里定位到 join-path-debug.c 的第 5 行定义 buf 只有 3 个的长度，找到了问题的原因。

假如说是一个正在运行的进程突然崩溃了，我们如何来定位原因呢？这时候就要调试 core dump文件了，首先在服务器上要做一些设置允许程序崩溃是产生 dump 文件。

    sudo mkdir -p /data/coredump/
    sudo chmod o+w /data/coredump
    sudo bash -c 'echo /data/coredump/core.%e.%p> /proc/sys/kernel/core_pattern'
    ulimit -c unlimited

上面代码是让 core dump 文件保存在 `/data/coredump` 下，并且 dump 文件名称包含程序名称和进程 ID，然后设置 dump 文件大小为无限制。

直接运行代码，就会产生 core dump 了。

    $ ./a.out
    *** stack smashing detected ***: <unknown> terminated
    Aborted (core dumped)
    $ ls /data/coredump/
    core.a.out.14375

拿到 dump 文件后需要有原始的文件才能进行调试，先启动 gdb:

    $ gdb ./a.out
    GNU gdb (Ubuntu 8.1.1-0ubuntu1) 8.1.1
    Reading symbols from ./a.out...done.

然后用 core-file 指令加载 core dump:

    (gdb) core-file /data/coredump/core.a.out.14375
    [New LWP 14375]
    Core was generated by `./a.out'.
    Program terminated with signal SIGABRT, Aborted.
    #0  __GI_raise (sig=sig@entry=6) at ../sysdeps/unix/sysv/linux/raise.c:51
    51      ../sysdeps/unix/sysv/linux/raise.c: No such file or directory.

查看调用栈

    (gdb) bt
    #0  __GI_raise (sig=sig@entry=6) at ../sysdeps/unix/sysv/linux/raise.c:51
    #1  0x00007fadd6b0f7f1 in __GI_abort () at abort.c:79
    #2  0x00007fadd6b58837 in __libc_message (action=action@entry=do_abort, fmt=fmt@entry=0x7fadd6c85869 "*** %s ***: %s terminated\n") at ../sysdeps/posix/libc_fatal.c:181
    #3  0x00007fadd6c03b31 in __GI___fortify_fail_abort (need_backtrace=need_backtrace@entry=false, msg=msg@entry=0x7fadd6c85847 "stack smashing detected") at fortify_fail.c:33
    #4  0x00007fadd6c03af2 in __stack_chk_fail () at stack_chk_fail.c:29
    #5  0x000055df3b1a47cc in main () at join-path-debug.c:8

live debug 能单步调试，设置断点等，但调试 core dump 基本上就是看一下调用栈，这是最重要的信息，一般也能够提供足够的线索来定位问题。
