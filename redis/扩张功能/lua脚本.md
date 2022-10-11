Why:可以在redis服务端做一些简单的业务逻辑

What: 

Lua是一种轻量级的**脚本语言**，用标准**C语言**编写并以源代码形式开发，其设计的目的是为了嵌入应用程序中，从而为应用提供灵活的扩展和定制功能。

Where:

When:

Who:

How:

How much:



### Lua环境协作组件

从redis2.6.0版本开始，通过内置的**lua编译器/解释器**，可以使用EVAL命令对lua脚本进行求职。

脚本的命令是原子的，ReidsServer在执行脚本命令中，不允许插入新的命令。

脚本的命令可以复制，RedisServer在获取脚本后，生成表示返回，Client根据标识就可以随时执行。



### EVAL/EVALSHA命令实现

EVAL命令

```redis
eval script numkeys key[key ...] arg[arg ...]
```

命令说明：

- Script参数：是一段Lua脚本程序，它会被运行在Redis服务器上下文中，这段脚本不必（也不应该）定义为一个Lua函数。

- **numkeys参数:**用于指定键名参数的个数。

- **key[key ...]参数:**从EVAL的第三个参数开始算起，使用了numkeys个键（key），表示在脚本中所用到的那些Redis键(key),这些键名参数可以在Lua中通过全局变量**KEYS**数组，用1位基址的形式访问（KEYS[1],KEYS[2],诸如此类）

- **arg[arg ...]参数：**可以在Lua中通过全局变量**ARGV**数组访问，访问的形式和KEYS变量类似（ARGV[1]、ARGV[2],诸如此类）

  **Lua脚本中执行Redis命令**

  - Redis.call():
    - 返回值就是redis命令执行的返回值
    - 如果出错，则返回错误信息，不继续执行
  - redis.pcall():
    - 返回值就是redis命令执行的返回值
    - 如果出错，则记录错误信息，并继续执行
  - **注意事项**
    - 在脚本中，是用return语句将返回值返回给客户端，如果没有return，则返回nill

```redis
eval "return redis.call('set',KEYS[1],ARGV[1])" 1 n1 zhanyun
```

