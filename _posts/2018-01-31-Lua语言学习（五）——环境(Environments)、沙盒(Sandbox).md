---
layout: post
title: Lua语言学习（五）——环境(Environments)、沙盒(Sandbox)
date: 2018-01-31
categories:
- 技术
tags: [Lua]
status: publish
type: post
published: true
---

**这一篇结合了好多篇东西。。说明下关于 Lua 中的环境和沙盒相关的内容**

---

### 环境(Environments)

Lua 中的全局变量都是存在表中的，这个表可以通过 \_G 来获取

```lua
for k,v in pairs(_G) do
  print(k,v)
end
```

打印一下表中的值(太多了，省略一些)

```
string	table: 00000000050174C0
xpcall	function: 0000000005019EE0
package	table: 0000000021CF07B0
tostring	function: 000000000501A060
print	function: 000000000501A120
.
.
.
dofile	function: 0000000005017210
_VERSION	Lua 5.1
setfenv	function: 0000000005019F70
error	function: 0000000005017060
loadfile	function: 00000000050172D0
```

在 Lua 5.1 中，可以通过 getfenv 和 setfenv 函数来操作这个环境表

getfenv 函数有一个参数，可以是一个函数，代表取这个函数中的环境，也可以是数字，代表堆栈级别，0代表取全局环境，1代表取当前函数的环境，2代表取调用当前函数的函数的环境，3代表更上一级，依此类推

setfenv 函数的第一个参数和 getfenv 一样，第二个参数代表了新的环境表

```lua
print(getfenv(1) == _G) -- true

a = 1

local function f(t)
  local print = print -- 改变环境表后就没有 print 函数了，所以这里提前定义一个内部的变量

  setfenv(1, t) -- 修改当前函数内的环境表为 t

  print(getmetatable) -- false，因为当前的环境中没有 getmetatable 函数

  a = 2 -- 定义当前环境中的全局变量，此变量不会影响到外部定义的 a
  b = 3 -- 同上
end

local t = {}
f(t)

print(a, b) -- 1 nil
print(t.a, t.b) -- 2 3
```

在 Lua 5.2 中，操作环境则使用 \_ENV 这个变量，直接附上 wiki 上的例子简单说明下吧：

```lua
print(_ENV == _G) -- true

a = 1

local function f(t)
  local print = print

  local _ENV = t

  print(getmetatable) -- nil

  a = 2
  b = 3
end

local t = {}
f(t)

print(a, b) -- 1 nil
print(t.a, t.b) -- 2 3
```

再附上一个自己写的小栗子，能更好的看出各个环境中的变量：

```lua
local x = "xxx"

function test1()
	a = 1
	print("in test1:",a,b,x)
end

function test2(f,env)
	setfenv(f,env)
	b = 2
	f()
	print("in env:  ",env.a,env.b,env.x)
	print("in test2:",a,b,x)
end

local env = {print=print}

print("             ","a","b","x")
test2(test1,env)
```

栗子运行结果：

```lua
         	a	b	x
in test1:	1	nil	xxx
in env:  	1	nil	nil
in test2:	nil	2	xxx
```

解释一下：

* 在 test2 中将 test1 函数的内部环境置为 env 表

* 变量 a 存在于 test1 函数内 及 test1 函数的环境内

* 变量 b 只存在于 test2 函数内

* 变量 xxx 可以在 test1 和 test2 函数内访问

---

### 沙盒(Sandbox)

讲了环境，就不得不说在 Lua 中使用环境能做些什么，比较直观的就是沙盒，使用沙盒，能在不影响全局环境的前提下，执行不可信的代码，并得知执行结果

附上 wiki 上的一个简单的沙盒例子吧

```lua
local env = {}

local function run(untrusted_code)
  if untrusted_code:byte(1) == 27 then return nil, "binary bytecode prohibited" end
  local untrusted_function, message = loadstring(untrusted_code)
  if not untrusted_function then return nil, message end
  setfenv(untrusted_function, env)
  return pcall(untrusted_function)
end

assert(not run [[print(debug.getinfo(1))]])
assert(run [[x=1]])
assert(run [[while 1 do end]])
```

这段代码里有几个之前没接触过的点：

* assert()

* pcall()

* loadstring()

* assert(not run xxx) 中的 not

稍微研究了下，就分别解释一下吧

#### assert

断言表达式(条件)是否为 nil 或 false

```lua
v = assert(v, message)
```

如果表达式 v 是 nil 或 false，则返回错误 message，message 参数是可选的，默认为 "assertion failed!"，如果断言没有错误，则返回表达式 v

```lua
function f()
	print(123)
end

local func,err = assert(f,"error!")

print(func,func == f,err) --function: 00000000586C9120	true	error!
```

#### pcall

和 pcall 相关的还有 xpcall 和 error，这些东西放到下一篇再一起说

#### loadstring

编译一段字符串，并返回编译好的代码块作为函数，但不执行它

```lua
f = loadstring(str, debugname)
```

第二个参数是用于表示调试时的错误信息，当字符串不能编译成正确的函数时，则会返回 nil

```lua
local f,err = loadstring([[looocal i = 1]],"wrrrrrong")
print(err)
--[string "wrrrrrong"]:1: '=' expected near 'i'
f()
--stdin:1: attempt to call global 'f' (a nil value)
```

提到 loadstring 就也要说到 string.dump ，这个库函数的作用是把一个现有的函数（此函数不能包含 upvalue）转成一个二进制字符串，之后就可以再使用 loadstring 获取该函数的一个副本

```lua
function f () print "hello, world" end
s = string.dump (f)
assert (loadstring (s)) () -- hello, world
```

#### not?

上面代码中 assert 中的 not，其实就是对 run 方法的结果取反，没有 not 时，结果是这样的

```lua
assert(run [[print(debug.getinfo(1))]])
--[string "print(debug.getinfo(1))"]:1: attempt to index global 'debug' (a nil value)
```

这个打印其实是出错的，断言的结果就是出错了，前面再加一个 not ，断言就变成了 true

---

### 参考

[http://lua-users.org/wiki/SandBoxes](http://lua-users.org/wiki/SandBoxes)

[http://lua-users.org/wiki/EnvironmentsTutorial](http://lua-users.org/wiki/EnvironmentsTutorial)

[https://www.gammon.com.au/scripts/doc.php?lua=assert](https://www.gammon.com.au/scripts/doc.php?lua=assert)

[https://www.gammon.com.au/scripts/doc.php?lua=loadstring](https://www.gammon.com.au/scripts/doc.php?lua=loadstring)

[http://timothyqiu.com/archives/lua-note-sandboxes](http://timothyqiu.com/archives/lua-note-sandboxes)
