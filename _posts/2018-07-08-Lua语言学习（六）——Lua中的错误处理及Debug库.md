---
layout: post
title: Lua语言学习（六）——Lua中的错误处理及Debug库
date: 2018-07-08
categories:
- 游戏开发
tags: [Lua脚本]
status: publish
type: post
published: true
---

### 写在前面

距离上一次更博过去了四个月，这四个月来几乎没有任何新的学习，非要找个理由的话，那就是995的工作实在是太特么累了。。从965的公司出来，半年了都还没完全适应7小时工作制到8小时工作制的转变，还要接受这种每天加班的生活，真是一种煎熬

吐槽归吐槽，找借口归找借口，最近的一件事也是给我这个咸鱼注入了一丝能量，开始恢复学习的日子吧

---

**这一篇说明下上一篇中留下的坑：pcall/xpcall/error 几个函数，顺便学习了一下 Lua 的 debug 库**

---

### 错误处理(Error Handling)

借书中的话说，Lua 是一种扩展语言，经常嵌入到应用程序中，所以在发生错误的时候不能简单地就崩溃或者退出，而是应该结束掉当前的错误块，并返回到程序中

Lua 的错误处理经常用到下面这三个函数

#### 1. error

error 函数会终止最后一个被调用的保护函数，并返回消息作为错误对象

error 函数有两个参数：

* 第一个参数为自己定义的错误内容

* 第二个参数 level 会指出错误的位置，默认为1，即调用 error 函数的位置，level = 2 时，指调用(调用 error 的函数)的函数的位置，使用0时代表不输出错误位置

```lua
function errFunc()
	error("error occurred",1)
end
errFunc()
--指出在第2行

function errFunc()
	error("error occurred",2)
end
errFunc()
--指出在第4行
```

#### 2. pcall

如果需要处理错误，可以使用 pcall 函数

pcall 函数有两个参数：

* 第一个参数为要执行的函数

* 第二个参数为传递给要执行函数的入参

pcall 函数会在保护模式(protected mode)下调用第一个参数，所以 可以捕获到该函数运行过程中的错误。如果没有错误，pcall 函数会返回 true 和执行函数的返回值；否则，会返回 false 和错误信息

```lua
function callFunc()
	return "return value"
end
print(pcall(callFunc))
--true	return value

function callFunc()
	error("error occurred")
	return "return value"
end
print(pcall(callFunc))
--false   stdin:2: error occurred
```

#### 3. xpcall

如果想在处理错误时获得更多的信息或做出更多的处理，那可以使用 xpcall 函数，xpcall 函数除了接收要执行的函数外，还可以接收一个错误处理函数(Error Handler Function)，Lua 会在堆栈被展开之前调用错误处理函数，所以可以在该函数中使用 Debug 库来收集想要的信息

例如用 debug.traceback() 打印出相关的堆栈

```lua
function errFunc()
	print(a.b)
end

function traceBackFunc(msg)
	print(debug.traceback(msg))
end

xpcall(errFunc,traceBackFunc)
```

```
stdin:2: attempt to index a nil value (global 'a')
stack traceback:
        stdin:2: in function 'traceBackFunc'
        stdin:2: in function 'errFunc'
        [C]: in function 'xpcall'
        stdin:1: in main chunk
        [C]: in ?
```

---

### Debug

Lua 的 debug 库没有直接提供一个调试器，但提供了编写调试器所需的所有函数，其中包含了两类函数：

* 内省函数(introspective functions)

* 钩子(hooks)

内省函数用来查看活动函数的栈、当前执行的行、局部变量的值和名称等等

钩子则允许用来跟踪程序的执行

这里只简单根据书中的内容介绍一下几个函数，debug 库复杂的应用暂时没有涉及

#### 1. debug.getinfo

debug.getinfo 函数的第一个入参可以是一个函数，或者是一个堆栈级别的值，使用该函数可以获得一个 table 包含了如下的值：

* **source**
>该函数定义的位置，如果该函数通过 loadstring 定义，source 就是那个字符串。如果该函数定义在一个文件中，source 就是该文件名，并且以一个@符号开头

* **short_src**
>最多60个字符的 source，对于错误信息很有用

* **linedefined**
>该函数被定义的行数

* **what**
>该函数是什么，可能是"Lua"——一个普通的 Lua 函数，可能是"C"——一个 C 函数，也可能是"main"——一个 Lua 代码块的 main 部分

* **name**
>该函数的名称

* **namewhat**
>函数名称的含义，这个字段可能是"global" "local" "method" "field"或者为空字符串""，为空字符串时，意味着 Lua 没有找到叫这个名称的函数

* **nups**
>该函数的 upvalues 的个数

* **func**
>该函数本身

因为 getinfo 函数的性能不是很高，所以函数还提供了第二个可选入参，该参数为字符串，通过不同的字母来仅获取全部数据中的一组：

| param | mean |
|:-----:|:---- |
| `n` | 取 name 和 namewhat |
| `f` | 取 func |
| `S` | 取 source，short_src，what 和 linedefined |
| `l` | 取 currentline |
| `u` | 取 nup |


附上书中的栗子：

```lua
function traceback ()
	local level = 1
	while true do
		local info = debug.getinfo(level, "Sl")
		if not info then break end
		if info.what == "C" then
			print(level, "C function")
		else
			print(string.format("[%s]:%d",
				info.short_src, info.currentline))
		end
		level = level + 1
	end
end
```

#### 2. debug.getlocal/debug.setlocal

使用 debug.getlocal 函数，可以获得任何活动的函数中的局部变量，该函数有两个入参：所查看的函数的堆栈级别、变量的索引，函数会返回两个值：变量的名称和值。如果变量的索引超出了变量的个数，则返回 nil；如果函数的堆栈级别无效，则会引发一个错误

同样，这里附上书中的栗子：

```lua
function foo (a,b)
	local x
	do local c = a - b end
	local a = 1
	while true do
		local name, value = debug.getlocal(1, a)
		if not name then break end
		print(name, value)
		a = a + 1
	end
end
foo(10, 20)
```

执行的结果为：

```
a   10
b   20
x   nil
a   4
```

函数 foo 的入参为最先的变量，其次是函数内部的局部变量，变量 c 和 getlocal 不在同一个作用域，所以并没有输出

除此之外，还可以使用 debug.setlocal 函数修改局部变量，前两个入参的含义和 debug.getlocal 相同，第三个入参是想要设置的新值，该函数返回对应变量的名称，如果索引超出，则返回nil

稍微修改一下上面的栗子：

```lua
function foo (a,b)
	local x
	local index = 1
	while true do
		local name, value = debug.getlocal(1, index)
		if not name then break end
		print("before change:",name,value)
		--修改变量的值为对应索引值
		debug.setlocal(1,index,index)
		name, value = debug.getlocal(1, index)
		print("after change:",name,value)

		index = index + 1
	end
end
foo(10, 20)
```

此时可以看到局部变量会被修改：

```
before change:	a	10
after change:	a	1
before change:	b	20
after change:	b	2
before change:	x	nil
after change:	x	3
before change:	index	4
after change:	index	4
```


#### 3. debug.getupvalue/debug.setupvalue

debug 库还提供了访问 upvalue 的方法，就是 debug.getupvalue 函数，这个函数的第一个参数是一个函数，更确切的说是一个闭包(闭包这个概念一直不是非常清楚，之前做 javascript 的时候也没了解过，先留个坑，下一篇学学闭包这个东西)，第二个参数是 upvalue 的索引

关于这个 upvalue 的翻译，从[官方解释](http://www.lua.org/manual/5.1/manual.html#2.6)来看，就是内部函数所访问的(外部)变量，对内部函数而言，称之为 upvalue 或外部局部变量(external local variable)

从[这篇博客](http://asqbtcupid.github.io/luahotupdate2-upvalue/)里有看到了一句觉得更白话(准确不准确暂时不知)的解释：**函数里用到的定义在该函数之前的local变量，就成为了该函数的upvalue**

这里有一个使用 debug.getupvalue 的栗子：

```lua
function newCounter()
	local n = 0
	return function()
		n = n + 1
		return n
	end
end

c = newCounter()
c()
c()

local i = 1

repeat
	name, val = debug.getupvalue(c, i)
	if name then
		print ("index", i, name, "=", val)
		i = i + 1
	end
until not name
```

这段代码输出的结果是：

```
index	1	n	=	2
```

getupvalue 函数输出了函数 c 的 upvalue n

setupvalue 函数和上面的 setlocal 函数类似，前两个参数和 getupvalue 函数一致，第三个参数是要设置的新值，这里就不再多解释了

#### 4. hook

Lua 的 hook 机制允许我们注册一个在程序运行过程中在特定事件下调用的函数，有四种类型的事件可以触发钩子：

* **call**
> 当 Lua 每调用一个函数时触发

* **return**
> 每当一个函数返回时触发

* **line**
> 当 Lua 开始执行新的一行代码时触发

* **count**
> 当 Lua 执行了指定次数的指令后触发

设置钩子可以使用 debug.sethook 函数，该函数的第一个入参就是钩子函数；第二个入参是一个字符串，用来表示我们想监控哪些事件， 对于 call/return/line 事件，我们使用它们的第一个字母(\`c\`/\`r\`/\`l\`)来代替；如果是监控 count 事件，则只需要传递计数次数作为第三个入参即可

另外，如果想停掉钩子，一样是使用 sethook 函数，不传递任何参数就可以了

一个简单的栗子：

```lua
function test()
	local a = 0
	for i = 0, 100 do
		a = a + 1
		print("a="..a)
	end
end
function hook(why)
	error("hook reached: " .. why)
end
debug.sethook (hook, "", 100)
test()
```

我自己的环境测试时，a 大概打印到13时就触发 hook 函数，所以这个 count 事件所谓的指令执行，就是指各种函数执行、赋值、加减、打印等操作

另一个简单的栗子：

```lua
function f()
	function g() end
	g()
	g()
end
function hook (why)
	print ("hook reached: ", why)
	print ("function =", debug.getinfo(2, "n").name)
end
debug.sethook(hook, "c", 0)
f()
```

这个栗子会打印出执行的函数：

```
hook reached: 	call
function =	f
hook reached: 	call
function =	g
hook reached: 	call
function =	g
```

---

### 最后

Lua 的 debug 库的应用，云风大大有很多篇博客讲到，这里先附上：

* [lua 代码的断点调试](https://blog.codingnow.com/2006/11/lua_breakpoint.html)

* [如何优雅的实现一个 lua 调试器](https://blog.codingnow.com/2016/11/lua_debugger.html)

* [Lua 调试器](https://blog.codingnow.com/2017/03/lua_debugger.html)

目前很多 Lua 的开发工具，都有已经实现的调试器，不过像云风大大说的，让调试器和程序分离，在外部监控程序内部，降低反复重启程序带来的时间消耗。现有的工具就是这个[mare](https://mare.js.org/)，有空要研究下这个怎么用，以及怎么用到项目中去

---

### 参考

[https://www.lua.org/pil/8.4.html](https://www.lua.org/pil/8.4.html)

[https://www.lua.org/pil/8.5.html](https://www.lua.org/pil/8.5.html)

[http://www.cnblogs.com/jadeboy/p/3978661.html](http://www.cnblogs.com/jadeboy/p/3978661.html)

[https://www.cnblogs.com/baiyanhuang/archive/2013/01/01/2841398.html](https://www.cnblogs.com/baiyanhuang/archive/2013/01/01/2841398.html)

[https://blog.csdn.net/snlscript/article/details/17138193](https://blog.csdn.net/snlscript/article/details/17138193)

[https://www.lua.org/pil/23.1.1.html](https://www.lua.org/pil/23.1.1.html)

[https://www.jianshu.com/p/180701ad1f07](https://www.jianshu.com/p/180701ad1f07)

[http://www.lua.org/manual/5.3/manual.html#pdf-error](http://www.lua.org/manual/5.3/manual.html#pdf-error)

[https://www.gammon.com.au/scripts/doc.php?lua=debug.getupvalue](https://www.gammon.com.au/scripts/doc.php?lua=debug.getupvalue)

[https://www.gammon.com.au/scripts/doc.php?lua=debug.sethook](https://www.gammon.com.au/scripts/doc.php?lua=debug.sethook)
