---
description: >-
  由于市面上的并发书籍基本上都是局限于某种并发方式，最多的就是JAVA多线程，书籍都叫并发编程，实际上只是多线程。然而，多任务处理不仅仅有多线程，还有多进程，IO多路复用，协程等等多种实现方式。因此一直都想自己写一本关于并发编程的书，可能做不到包罗万象，但至少可以涵盖所有关于多任务处理方面的知识。
---

# 序



* [前言](qian-yan.md)
* [第一章 多任务处理的发展历史](di-yi-zhang-duo-ren-wu-chu-li-de-fa-zhan-li-shi/)
  * [1、高性能系统要求](di-yi-zhang-duo-ren-wu-chu-li-de-fa-zhan-li-shi/1-gao-xing-neng-xi-tong-yao-qiu.md)
  * [2、多任务的发展历史](di-yi-zhang-duo-ren-wu-chu-li-de-fa-zhan-li-shi/2-duo-ren-wu-de-fa-zhan-li-shi.md)
  * [3、并发编程技术的介绍](di-yi-zhang-duo-ren-wu-chu-li-de-fa-zhan-li-shi/3-bing-fa-bian-cheng-ji-shu-de-jie-shao.md)
* [第二章 多进程](di-er-zhang-duo-jin-cheng/)
  * [进程介绍](di-er-zhang-duo-jin-cheng/jin-cheng-jie-shao.md)
  * [多进程原理](di-er-zhang-duo-jin-cheng/duo-jin-cheng-yuan-li.md)
  * [多进程模型](di-er-zhang-duo-jin-cheng/duo-jin-cheng-mo-xing.md)
  * [多进程案例讲解](di-er-zhang-duo-jin-cheng/duo-jin-cheng-an-li-jiang-jie/)
    * [Nginx多进程](di-er-zhang-duo-jin-cheng/duo-jin-cheng-an-li-jiang-jie/nginx-duo-jin-cheng.md)
    * [Gunicorn](di-er-zhang-duo-jin-cheng/duo-jin-cheng-an-li-jiang-jie/gunicorn.md)
  * [总结](di-er-zhang-duo-jin-cheng/zong-jie.md)
* [第三章 多线程](di-san-zhang-duo-xian-cheng/)
  * [多线程介绍](di-san-zhang-duo-xian-cheng/duo-xian-cheng-jie-shao.md)
  * [线程安全性](di-san-zhang-duo-xian-cheng/xian-cheng-an-quan-xing/)
    * [安全性问题的由来](di-san-zhang-duo-xian-cheng/xian-cheng-an-quan-xing/an-quan-xing-wen-ti-de-you-lai.md)
    * [线程中的锁事](di-san-zhang-duo-xian-cheng/xian-cheng-an-quan-xing/xian-cheng-zhong-de-suo-shi.md)
    * [线程安全实战](di-san-zhang-duo-xian-cheng/xian-cheng-an-quan-xing/xian-cheng-an-quan-shi-zhan.md)
  * [线程池介绍](di-san-zhang-duo-xian-cheng/xian-cheng-chi-jie-shao/)
    * [线程池介绍](di-san-zhang-duo-xian-cheng/xian-cheng-chi-jie-shao/xian-cheng-chi-jie-shao.md)
    * [线程池原理](di-san-zhang-duo-xian-cheng/xian-cheng-chi-jie-shao/xian-cheng-chi-yuan-li.md)
    * [线程池实战](di-san-zhang-duo-xian-cheng/xian-cheng-chi-jie-shao/xian-cheng-chi-shi-zhan.md)
  * [JAVA并发容器](di-san-zhang-duo-xian-cheng/java-bing-fa-rong-qi.md)
  * [多线程实际案例讲解](di-san-zhang-duo-xian-cheng/duo-xian-cheng-shi-ji-an-li-jiang-jie.md)
  * [多线程总结](di-san-zhang-duo-xian-cheng/duo-xian-cheng-zong-jie.md)
* [第四章 IO多路复用](di-si-zhang-io-duo-lu-fu-yong/)
  * [I/O基础](di-si-zhang-io-duo-lu-fu-yong/io-ji-chu.md)
  * [I/O模型介绍](di-si-zhang-io-duo-lu-fu-yong/io-mo-xing-jie-shao.md)
  * [I/O多路复用原理](di-si-zhang-io-duo-lu-fu-yong/io-duo-lu-fu-yong-yuan-li.md)
  * [高性能网络I/O模式](di-si-zhang-io-duo-lu-fu-yong/gao-xing-neng-wang-luo-io-mo-shi.md)
  * [Redis中的高并发](di-si-zhang-io-duo-lu-fu-yong/redis-zhong-de-gao-bing-fa.md)
  * [Netty高性能介绍](di-si-zhang-io-duo-lu-fu-yong/netty-gao-xing-neng-jie-shao.md)
  * [Python asyncio介绍](di-si-zhang-io-duo-lu-fu-yong/python-asyncio-jie-shao.md)
  * [总结](di-si-zhang-io-duo-lu-fu-yong/zong-jie.md)
* [第五章 协程](di-wu-zhang-xie-cheng.md)

&#x20;    &#x20;

