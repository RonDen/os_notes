# 2020年秋季操作系统高级课程小组记录

## XV6源码阅读要求

>- 学习小组形式：4-5人一组（设组长1名，教师和助教的联系人， 上传下达，反馈同学意见建议，组织讨论）。
>- 任务：每周召集一次讨论，讨论时间**不低于1-2小时**，讨论内 容包括在源代码阅读时遇到的问题、XV6关键实现技术、涉及 的操作系统概念总结。每次指定一名同学主讲（要求每个同学 组内至少演讲1次），撰写讨论记录，评定表现。
>- 阅读代码，自行查阅相关资料，了解XV6的工作原理和实现机制。
>- 根据上课内容，结合XV6代码，按**小组**给出代码阅读报告（具体要求会给出）。
>- 报告内容：原理课的机制与实现和XV6机制与实现的比较，从性能、实现复杂性、硬件支持、可伸缩性等等方面加以说明。

具体要求见第一次课课程简介44-45页。

## 进度安排

| 时间       | 主题           | 成员           |
| ---------- | -------------- | -------------- |
| 第一、二周 | 进程线程       | 周旭敏，毕廷竹 |
| 第三、四周 | 同步机制       | 罗登，周旭敏   |
| 第五、六周 | 中断与系统调用 | 罗旭坤，罗登   |
| 第七、八周 | 存储管理       | 待定           |
| 第九、十周 | 文件系统       | 待定           |

成员中第一位为每周的汇报者，第二位为记录者。建议每周的汇报者做一个书面形式的文档或提纲（markdown格式或ppt/pdf格式），可以尝试绘制一些简单易懂的图示，将操作系统中的复杂概念或繁杂的源码结构提纲挈领地展现出来，以减少阅读者学习的负担，少踩一些坑。建议汇报者能讲解加演示达到40min左右，其他参与者每周至少阅读一篇相关博客或针对报告中提出的问题进行搜索的文章，在当面讨论的时候积极提问、交流。每周记录者负责记录开会讨论的内容，以及比较精彩或重要的部分（包括拍照），这很**重要**！

奇数周以讲解源码，熟悉概念为主题；偶数周以完成报告为主，在会上讨论和汇总，尽量在两周之内完成对应主题的报告，不拖沓。

## 相关资源

代码、书籍和相关课程等资源：

XV6源码仓库：https://github.com/mit-pdos/xv6-public

MIT6.828操作系统工程，本课程以学习XV6为主，OS的概念以及各种关于XV6的资料、Lab基本都可以在上面找到，内容非常丰富（就是学起来比较头冷，需要毅力）：https://pdos.csail.mit.edu/6.828/2014/schedule.html

介绍XV6设计的书籍：https://pdos.csail.mit.edu/6.828/2020/xv6/book-riscv-rev1.pdf

书籍的中文版（版本不同，内容相似）：https://github.com/ranxian/xv6-chinese/tree/master/content

学完MIT6.828的知乎大佬的经验分享：https://zhuanlan.zhihu.com/p/74028717

在Youtube上找到两个讲xv6的视频

https://www.youtube.com/watch?v=ktkAlbcoz7o
https://www.youtube.com/watch?v=ikcfQw4FPEw&list=PLrYtVwt5XyvenaqRLNEFMD5uxswTpYwmN

关于汇编语言：

B站UP上传的小甲鱼语言教程：https://www.bilibili.com/video/BV1Et411t7ks

一本关于处理器和指令集的实用手册  https://pdos.csail.mit.edu/6.828/2005/readings/pdp11-40.pdf

网页版的指令索引： https://pdos.csail.mit.edu/6.828/2005/pdp11/

关于XV6能做什么：

东京大学学生在自己制作的CPU上移植XV6系统并实现了各种有趣的小程序：https://fuel.edby.coffee/posts/how-we-ported-xv6-os-to-a-home-built-cpu-with-a-home-built-c-compiler/

XV6源代码阅读之进程线程：https://www.cnblogs.com/icoty23/p/10993807.html

一个个人博客地址，里面有他的XV6的学习记录，可以作为参考（对照着问题去看相应的代码）：https://icoty.github.io/
