---
title: 重新定义Hello World
date: 2022-08-20 11:26:14
tags:
---

目录
- 原始的 Hello World 程序
- 配置开发环境 
- 单元测试
- 调试
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
- 了解更多的断言，如单元数组中包含指定元素；
- 在每个单元测试前面进行准备工作，后面做资源回收工作；
- 对被测函数依赖的函数进行 mock；
- 断言某个函数以指定的参数被调用；
- 临时屏蔽某个测试用例；
- 评估被测代码的测试覆盖率；
