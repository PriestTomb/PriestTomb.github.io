---
layout: post
title: Lua语言学习（四）——协程(Coroutine)、元表(Metatable)、元方法(Metamethod)
date: 2018-01-21
categories:
- 游戏开发
tags: [Lua脚本]
status: publish
type: post
published: true
---

**这一篇同样是根据[ Lua-Users wiki ](http://lua-users.org/wiki/TutorialDirectory)梳理的一些细节**

---

### 协程(Coroutine)

Lua 中的协程不是操作系统线程或进程，而是 Lua 中创建的代码块，协程有像线程一样的控制流。同一时间仅有一个协程在运行，一直运行直到它激活另一个协程或挂起（返回调用它的协程）。协程以一种方便自然的方式表达了多个协作线程，但是不能并行执行，从而没有获得多 CPU 的性能优势。然而，由于协程转换比操作系统线程更快，并且通常不需要复杂且有时昂贵的锁机制，所以使用协程通常比使用完整 OS 线程的等效程序快

为了使多个协程能共享执行，它们必须停止执行（在执行了合理的处理后）然后把控制权传给另一个协程，这个行为称为挂起（yielding）。这是通过调用 Lua 的一个函数 coroutine.yield() 来实现的，这个函数和在函数中使用 return 类似，两者的区别在于：挂起后，还可以从挂起的地方重新进入线程，return 后则会销毁域，不能再重新进入

#### 简单使用

为了创建一个协程就必须有一个代表它的函数

```lua
> function foo()
>>   print("foo", 1)
>>   coroutine.yield()
>>   print("foo", 2)
>> end
```

使用 coroutine.create(fn) 创建协程

```lua
> co = coroutine.create(foo)
> = coroutine.status(co)
suspended
```

这时的协程状态是暂停的（suspended），意思是线程是活着的，只是什么都没做。创建后想启动协程需要使用 coroutine.resume() 函数

```lua
> = coroutine.resume(co)
foo 1
```

可以看出函数内执行到第一条打印语句就停止了，这时我们可以再次执行 coroutine.resume() 函数恢复协程

```lua
> = coroutine.resume(co)
foo 2
```

协程恢复到刚才挂起的下一行继续执行到结束，这时如果再执行 coroutine.resume() 函数则会发现，线程已经死了，不能再恢复

```lua
> = coroutine.resume(co)
false   cannot resume dead coroutine
```

#### 更多细节

这里直接附网站上的这个例子吧，看了好几遍才捋清楚。。

```lua
> function odd(x)
>>   print('A: odd', x)
>>   coroutine.yield(x)
>>   print('B: odd', x)
>> end
>
> function even(x)
>>   print('C: even', x)
>>   if x==2 then return x end
>>   print('D: even ', x)
>> end
>
> co = coroutine.create(
>>   function (x)
>>     for i=1,x do
>>       if i==3 then coroutine.yield(-1) end
>>       if i % 2 == 0 then even(i) else odd(i) end
>>     end
>>   end)
>
> count = 1
> while coroutine.status(co) ~= 'dead' do
>>   print('----', count) ; count = count+1
>>   errorfree, value = coroutine.resume(co, 5)
>>   print('E: errorfree, value, status', errorfree, value, coroutine.status(co))
>> end
----    1
A: odd  1
E: errorfree, value, status     true    1       suspended
----    2
B: odd  1
C: even 2
E: errorfree, value, status     true    -1      suspended
----    3
A: odd  3
E: errorfree, value, status     true    3       suspended
----    4
B: odd  3
C: even 4
D: even         4
A: odd  5
E: errorfree, value, status     true    5       suspended
----    5
B: odd  5
E: errorfree, value, status     true    nil     dead
```

网站接下来的内容不直接翻译了，说几个我理解的比较核心的点吧：

* 最外围是一个 while 循环，循环的条件是协程 co 的状态不是 dead

* 协程函数内部也是一个循环，一个循环 x 次的 for 循环，x 的值是在使用 resume() 方法时传入的

* 使用 resume() 方法重启协程时，第一个参数后的其他参数会被协程接收，这个例子里只体现了一次，就是第一次启动协程时，传入的参数 5 被赋值给协程函数的 x，所以这个例子中协程内部会循环 5 次；之后的 resume() 会使协程返回到上一次 yield() 的地方继续执行

* 使用 yield() 方法挂起协程时，会返回到重启当次协程的 resume() 方法的地方，所以 yield(x) 和 yield(-1) 时，这个 x 和 -1 都被 while 循环中的 value 接收，errorfree 则接收了当次 resume() 操作的结果，在本例中所有的重启都是成功的，所以 errorfree 都是 true

* 在内部和外部的循环中来回跳转的关键就是在 yield() 和 resume() 之间跳转

#### 插播一个栗子

wiki 上的例子缺少了一些"赋值"的细节，这里再来一个[w3cschool教程](https://www.w3cschool.cn/lua/lua-coroutine.html)上的例子

```lua
function foo (a)
    print("foo 函数输出", a)
    return coroutine.yield(2 * a)
end

co = coroutine.create(function (a , b)
    print("第一次协同程序执行输出", a, b)
    local r = foo(a + 1)

    print("第二次协同程序执行输出", r)
    local r, s = coroutine.yield(a + b, a - b)

    print("第三次协同程序执行输出", r, s)
    return b, "结束协同程序"
end)

print("main", coroutine.resume(co, 1, 10))
print("---分割线----")
print("main", coroutine.resume(co, "r"))
print("---分割线---")
print("main", coroutine.resume(co, "x", "y"))
print("---分割线---")
print("main", coroutine.resume(co, "x", "y"))
print("---分割线---")
```

执行结果：

```
第一次协同程序执行输出	1	10
foo 函数输出	2
main	true	4
---分割线----
第二次协同程序执行输出	r
main	true	11	-9
---分割线---
第三次协同程序执行输出	x	y
main	true	10	结束协同程序
---分割线---
main	false	cannot resume dead coroutine
---分割线---
```

在这个例子中可以更明显的看出 resume() 和 yield() 时怎么传参赋值：

* 第一次 resume(co, 1, 10) 时，把 1 和 10 传递给了协程函数，赋值给 a 和 b

* 在 foo() 函数中 yield(2 * a) 时，把 4 传回了上一次启动的地方，即 resume(co, 1, 10)，所以第一次输出 main 时，是 main    true    4

* 第二次 resume(co, "r") 时，把 "r" 传回了上一次挂起的地方，即 yield(2 * a)，foo() 函数的返回值也就变成了 "r"，所以 local r = "r"

* 同理，yield(a + b, a - b) 时，把 11 和 -9 传回了 resume(co, "r") 的地方，第二次输出 main 时，是 main    true    11  -9

* 同理，第三次 resume(co, "x", "y") 时，把 "x" 和 "y" 传回，赋值给了 local r, s

---

### 元表和元方法

Lua 中有类似其他语言中"运算符重载"一样的概念，元表是包含一些元方法的常规表，在 Lua 中执行某些操作时会触发事件，这些元方法就和这些事件有关

举个例子就像两个表相加的操作，会触发 \_\_add 方法

#### 元表(Metatable)

使用 setmetatable() 方法设定一个表作为元表

```lua
local x = {value = 5}

local mt = {
  __add = function (lhs, rhs)
    return { value = lhs.value + rhs.value }
  end
}

setmetatable(x, mt)

local y = x + x

print(y.value) --10

local z = y + y --error
```

设置 mt 为 x 的元表，元表中拥有 \_\_add 元方法，所以 x 才可以使用 + 操作符，y 没有元方法，不能使用 + 操作符

如果其中一个操作数是数字，同样也可以触发元表中的方法，并且，左边的操作数就是函数中的第一个参数，右边的操作数就是第二个参数，所以具有元表的表也不一定是元方法的第一个参数：

```lua
local x = {value = 5}

local mt = {
	__add = function(l,r)
		return {value = l + r.value}
	end
}

setmetatable(x,mt)

local y = 5 + x

print(y.value) --10
```

#### 元方法(Metamethod)

这里直接搬运 wiki 上的元方法的示例

###### 0. \_\_index

使用 \_\_index 键可以接备用表(fallback table)或者函数，如果接函数，则第一个参数是查找失败的表，第二个参数是查找的键。备用表同样可以触发它的 \_\_index 键，所以可以创建一个很长的备用表链

```lua
local func_example = setmetatable({}, {__index = function (t, k)
  return "key doesn't exist"
end})

local fallback_tbl = setmetatable({
  foo = "bar",
  [123] = 456,
}, {__index=func_example})

local fallback_example = setmetatable({}, {__index=fallback_tbl})

print(func_example[1]) --> key doesn't exist
print(fallback_example.foo) --> bar
print(fallback_example[123]) --> 456
print(fallback_example[456]) --> key doesn't exist
```

###### 1. \_\_newindex

当向表中新分配一个不存在的键时，会触发 \_\_newindex 元方法，如果该键已存在，则不会触发

```lua
local t = {}

local m = setmetatable({}, {__newindex = function (table, key, value)
  t[key] = value
end})

-- 注：上面使用函数等同于直接指定 __newindex 为表 t
-- local m = setmetatable({}, {__newindex = t})

m[123] = 456
print(m[123]) --> nil
print(t[123]) --> 456
```

###### 2. 比较

```lua
local mt
mt = {
  __add = function (lhs, rhs)
    return setmetatable({value = lhs.value + rhs.value}, mt)
  end,
  __eq = function (lhs, rhs)
    return lhs.value == rhs.value
  end,
  __lt = function (lhs, rhs)
    return lhs.value < rhs.value
  end,
  __le = function (lhs, rhs)
    return lhs.value <= rhs.value
  end,
}
```

###### 3. \_\_metatable

如果想保护一个元表不被程序修改，可以使用 \_\_metatable 键

wiki 上简短的一两句话还没有示例，有点让人摸不着头脑，我一开始以为是说一个表设置了 \_\_metatable 之后，就不能被设置为其他表的元表了，测试后发现不对，看了这篇[Lua中的metatable详解](http://www.jb51.net/article/56690.htm)的内容后才明白。。

```lua
local mt = {__metatable = "error"}

local t = setmetatable({}, mt)

print(getmetatable(t)) --error

setmetatable(t,{}) --stdin:1: cannot change a protected metatable
```

原来是一个已经被设置为元表的表，设置这个键后就不允许通过 getmetatable() 获取，从而不允许被修改，反例：

```lua
local mt = {test=1}

local t = setmetatable({}, mt)

local mt2 = getmetatable(t)

print(mt.test,mt2.test) --1 1

-- 修改元表中的数据
mt2.test = 2

print(mt.test,mt2.test) --2 2
```

###### 4. 元方法手册

详细的元方法列表可以看[这里](http://www.lua.org/manual/5.2/manual.html#2.4)
