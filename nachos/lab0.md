# 实验要求

> 1. 在提供的虚拟机中编译Nachos3.4
> 2. 通过Makefile熟悉项目结构
> 3. 尝试添加一个源文件，并且成功编译

记录第3步，添加文件时出现的问题。

尝试在`thread/`文件夹中添加一个`test.h`与`test.cc`文件。内容如下：

```c
// test.h
#ifndef TEST_H

#define TEST_H

int add(int, int);

int test_h;

#endif
```

```c
// test.cc
#include "test.h"

int add(int a, int b)
{
	return a + b;
}
```

在`main.cc`中修改，包含该文件，并且调用`add`方法。

```c
// main.cc
//...
#include "test.h"
//...
// line 82-84
int main(int argc, char **argv)
{
	printf("int main add(1,2)=%d\n", add(test_h, test_h));
    ...
}
```

修改对应的Makefile文件，分别是`thread/Makefile`，`Makefile.common`文件。

在`thread/Makefile`文件中添加`main.o`的依赖项。

```makefile
main.o: ...# 已省略
	../threads/test.h
```

对应的，在`Makefile.common`文件中`THREAD`相关的部分添加`test.h`或`test.cc`。

```makefile
# Makefile.common
THREAD_H =../threads/copyright.h\
	...
	../threads/test.h\
	..。

THREAD_C =../threads/main.cc\
	...
	../threads/thread.cc\
	...

THREAD_O =main.o list.o scheduler.o synch.o synchlist.o system.o thread.o \
	utility.o threadtest.o interrupt.o stats.o sysdep.o timer.o test.o
```

编译：

```bash
vagrant@precise32:/vagrant/nachos/nachos-3.4/code$ make
```

可是出现了报错：

![error](https://images.gitee.com/uploads/images/2020/1014/230115_311489c5_5268452.png "屏幕截图.png")

出错阶段为`ld`链接阶段，错误信息为`test_h`出现了重复定义`multiple definition`，可是实际上只定义了一次，这是为什么呢？

原来在编译期间，`main.cc`与`test.cc`均被编译成为了`*.o`文件，`test_h`的定义就保存在其中了。因此在链接阶段，就出现了重复定义错误。

解决办法：

在`test.h`中将`test_h`声明为全局`extern`，具体定义在`test.cc`中完成，这样就可以确保`test_h`这个名称的唯一性了。

```c
// test.h
#ifndef TEST_H

#define TEST_H

int add(int, int);

extern int test_h;

#endif
```

```c
#include "test.h"

int test_h = 10;

int add(int a, int b)
{
	return a + b;
}
```

再次编译项目，可以看到编译成功，运行`nachos`文件，也能得到预期效果。

```bash
vagrant@precise32:/vagrant/nachos/nachos-3.4/code$ make
```

![success](https://images.gitee.com/uploads/images/2020/1014/231007_ceb716cd_5268452.png "屏幕截图.png")

![res](https://images.gitee.com/uploads/images/2020/1014/231046_8b2c17e5_5268452.png "屏幕截图.png")

### 总结

C语言中编译单个文件和编译整个项目的许多概念是非常不同的，首先是使用了Makefile来管理和组织文件，这样每次添加或删除源文件，都需要在Makefile中做出相应的修改，才能保证整个项目的正确编译。

其次还有变量名占用，条件编译等注意事项，这次遇到的问题也让我了解了`extern`关键字的用法，对C项目编译链接等过程有了更深的认识。突然想到最开始学习编程和C语言时听到的一句话，感谢每一次编译器给出的错误信息，那都是学习和提高的好机会！

### 参考资料

https://blog.csdn.net/mantis_1984/article/details/53571758