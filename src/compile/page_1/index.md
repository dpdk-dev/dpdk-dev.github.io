# \_\_attribute\_\_((constructor))与静态链接

全文转载：[https://zhuanlan.zhihu.com/p/42814326](https://zhuanlan.zhihu.com/p/42814326)

最近几天，尝试集成 SPDK到我们的分布式系统里。为了避免对SPDK进行较大的改动，决定使用静态链接库，然后利用SPDK的API写target。

编译 SPDK ，写一个简单的 target 例子都很顺利。但是当运行的时候，reactor初始化分配mempool一直错误。这个错误非常奇怪。第一时间怀疑hugepage空间不足，但是仔细看了一下，肯定是够的。

与此同时，发现同样的DPDK配置，SPDK自带的app都可以初始化成功。从日志看一模一样，除了最后，自己写的初始化mempool失败，自带的可以成功。百思不得其解。

终于在SPDK的issue里看到了一个同样的错误：

```
lib/event/reactor.c: 685:spdk_reactors_init: ERROR: spdk_event_mempool creation failed
```

根据反馈看，应该是 \_\_attribute\_\_ ((constructor)) 的问题。

经过在初始化路径上加上日志，发现就是这个问题。但是，为什么会这样呢？

这个扩展属性，不就是gcc默认就打开了么？

经过试验发现，原来，问题出在静态库上。

```
$ cat foo.c
#include <stdio.h>

__attribute__((constructor)) void foo()
{
    printf("foo called\n");
}

#include<stdio.h>
__attribute__((constructor)) void before_main() {
   printf("Before main\n");
}
__attribute__((destructor)) void after_main() {
   printf("After main\n");
}
int main(int argc, char **argv) {
   printf("In main\n");
   return 0;
}
 $ gcc -c foo.c
 $ ar rcs foo.a foo.o
 $ gcc test.c foo.a
 $ ./a.out
Before main
In main
After main
```

看一下动态库：

```
$ g++ -c foo.c -o foo.o -fPIC
$ g++ -shared -fPIC -o libfoo.so foo.o
$ g++ test.c -L. -lfoo
$ export LD_LIBRARY_PATH=`pwd`
$ ./a.out                     
foo called
Before main
In main
After main
```

发现问题了，静态链接的foo.c里面的 \_\_attribute\_\_((constructor)) 并没有执行。动态链接后是正确的。

原因是linker在静态链接的时候，默认只链接用到的符号，没有用到的就不会拿过来了。对于上面的这种情况，\_\_attribute\_\_((constructor)) 属性的模块构造器就被strip了。

解决办法是使用-Wl,--whole-archive和--no-whole-archive标签，显式的标识链接所有的符号。

```
 $ g++ test.c -Wl,--whole-archive foo.a -Wl,--no-whole-archive
root@31-216:~/smartx/spdk
 $ ./a.out 
Before main
foo called
In main
After main
```

> 注意，-Wl,--whole-archive 中间不能有空格。

Done。

这个坑容易踩到，在此记录一下。