---
title: Predis的坑
categories: PHP
tags:
  - redis
  - Predis
  - PHP
  - NoSQL
date: 2017-11-12 16:53:54
---

过去一周我们公司的Redis一直收到报错信息。 于是乎我就上了我们线上服务器查了下错误日志。清一色都是显示Predis链接超时。
<!-- more -->
首先，我们现在用的是Predis，是一个纯PHP写的PHP连接Redis的一个包。不是phpredis。
于是我查了一下代码，发现在连接Redis的时候明明做了异常处理，为什么还会直接抛出呢？贴上代码：

```
...

try {

    $cache = new Client($config, $option);

   } catch (Exception $exception) {
        
        ....
        
   }
        
...
```

明明这里捕获了异常，为什么还抛出？  
于是我本地测试了一下，追踪了一下代码，发现Predis在new对象的时候根本没有去做连接Redis的操作！！
即使你在代码里面显示的调用链接方法，也不会去链接。
```
    $cache->connect();
```

那到底什么时候去链接呢？我大概看了一下Predis的源码，发现是在操作Redis命令的时候。比如：`get`,`set`等。
这是一个比较坑的地方。也就说如果跟我上面那样去做异常处理，连接不上Redis的时候还是会抛异常。也就是接口500。
所以异常处理应该是在Predis的`__call()`方法的时候去做。为什么是`__call()`方法呢？因为Predis的Redis命令操作都是调的`__call()`方法。

最后，提醒一下另一个坑。  Predis连接redis是采用长连接的，也就说如果没有显示的关闭这个连接，那么这个连接就会一直存在，直到进程结束。当然这对于高并发网站来说是提高性能的一个点。需要注意的是Redis的最大连接数问题。如果连接数超过了Redis的最大连接数，那么这边就会出问题。这个需要注意一下。

还有一个坑点在于Predis的注释。代码是这样的：
```
    /**
     * Closes the underlying connection and disconnects from the server.
     *
     * This is the same as `Client::disconnect()` as it does not actually send
     * the `QUIT` command to Redis, but simply closes the connection.
     */
    public function quit()
    {
        $this->disconnect();
    }
```

欺负我英文不好？？这就说即使我们调用了`quit()`去关闭这个链接，但是并没有实际关闭？带着这个疑惑我是做了一个测试。
测试过程我就不写了。我是用`redis`的`info`命令去查看连接数的。测试结果发现如果我们真的调用了关闭方法，那么这个连接是会真实退出的。并不是像注释说的那样。。。



好了  今天是周末 没出去玩，写了几篇博客，也真的是好久没动手写博客了。周末愉快~~

