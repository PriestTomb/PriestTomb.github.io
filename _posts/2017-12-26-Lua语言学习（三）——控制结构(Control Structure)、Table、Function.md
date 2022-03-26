---
layout: post
title: Lua语言学习（三）——控制结构(Control Structure)、Table、Function
date: 2017-12-26
categories:
- 技术
tags: [Lua]
status: publish
type: post
published: true
---

**这一篇同样是根据[ Lua-Users wiki ](http://lua-users.org/wiki/TutorialDirectory)梳理的一些细节**

---

### 控制结构(Control Structure)

#### break

在嵌套循环中，break 只作用于最内层的一个循环：

```lua
> for i = 1, 2 do
>> while true do
>> break
>> end
>> print(i)
>> end
1
2
```

在循环外使用 break 是一个语法错误：

```lua
> break
stdin:1: <break> at line 1 not inside a loop
```

#### 类 continue

Lua 不像其他的语言拥有 continue 关键字能跳出当前循环，在 Lua 5.2 中，可以使用 goto 关键字实现：

```lua
> for i = 1, 10 do
>> if i > 3 and i < 6 then goto continue end
>> print(i)
>> ::continue:: --使用双冒号包围一个指定位置
>> end
1
2
3
6
7
8
9
10
```

在5.2以前的版本中，想实现类 continue 的效果，只能用其他的一些变通方法：

```lua
--直接使用内部 if 判断
for i = 1, 10 do
  if (i <= 3 or i >= 6) then
    print(i)
  end
end

--使用 repeat + if
for i = 1, 10 do
  repeat
    if (i > 3 and i < 6) then
      break
    end
    print(i)
  until true
end

--使用 while + if
for i = 1, 10 do
  while true do
    if (i > 3 and i < 6) then
      break
    end
    print(i)
    break
  end
end
```

#### 条件

条件不一定非要是 boolean 值，在 Lua 中，除了 nil 和 false 外，任何值都是 true：

```lua
> if 5 then print("true") else print("false") end
true
> if 0 then print("true") else print("false") end
true
> if true then print("true") else print("false") end
true
> if {} then print("true") else print("false") end
true
> if "string" then print("true") else print("false") end
true
> if nil then print("true") else print("false") end
false
> if false then print("true") else print("false") end
false
```

其他语言中，条件也可以是一个表达式，但在 Lua 中是不行的：

```lua
> i = 0
> while (i = i + 1) <= 10 do print(i) end
stdin:1: ')' expected near '='
```

---

### Table

在 Lua 中，除了 nil 和 NAN(Not a Number)，任何值都可以做 key：

```lua
> t = {}
> k = {}
> f = function () end
> t[k] = 123
> t[f] = 456
> = t[k]
123
> = t[f]
456
> t[nil] = 123
stdin:1: table index is nil
stack traceback:
        stdin:1: in main chunk
        [C]: in ?
> t[0/0] = 123
stdin:1: table index is NaN
stack traceback:
        stdin:1: in main chunk
        [C]: in ?
```

当 key 是 string 时，可以有这几种方法初始化：

```lua
-- 初始化时使用方括号设置 key，要有引号
> t = {["test"] = 123}

-- 初始化时不用方括号，不需要引号
> t = {test = 123}

-- 使用点设置 key，不用引号
> t.test = 123

-- 使用点和方括号设置 key，要有引号
> t.["test"] = 123
```

使用方括号设置 key 时，如果不带引号，则代表是一个变量，如果这个变量没有初始化，就会报错：

```lua
-- 没初始化 test，报错
> t[test] = 123
stdin:1: table index is nil
stack traceback:
        stdin:1: in main chunk
        [C]: ?

-- 初始化 test，正常
> test = "test"
> t[test] = 123
```

如果初始化时不指定 key 值，则 key 默认从1开始递增，在使用 iterator 遍历时可以看到，会先遍历默认 key，再遍历指定 key：

```lua
> t = {"a", "b", [123]="foo", "c", name="bar", "d", "e"}
> for k,v in pairs(t) do print(k,v) end
1       a
2       b
3       c
4       d
5       e
123     foo
name    bar
```

table 的长度也可以使用 # 符号获取：

```lua
> t = {"a", "b", "c"}
> = #t
3
```

使用 table.insert() 方法向 table 中插入值：

```lua
> t = {"a", "c"}

-- 两个参数：要操作的 table 要插入的值，默认在末尾插入
> table.insert(t, "d")

-- 三个参数：要操作的 table 在哪个位置插入 要插入的值
> table.insert(t, 2, "b")

> for k,v in pairs(t) do print(k,v) end
1	a
2	b
3	c
4	d
```

使用 table.remove() 方法从 table 中移除值：

```lua
> t = {"a", "b", "c", "d", test = 123}

-- 默认删除默认 key 的最后一个
> table.remove(t)  -- 删除 d 而不是123

-- 删除指定 key
> table.remove(t, 2)  -- 删除 b

> for k,v in pairs(t) do print(k,v) end
1	a
2	c
test	123
```

使用 table.concat() 方法可以拼接 table 中的值，但测试发现两点，key 为字符串的自定义值不被拼接、不连贯的数字 key 也不会被拼接：

```lua
-- 指定一个数字 key 为5，则可拼接
> t = {"a", "b", "c", "d", [5] = "e", ["test"] = 123}
> print(table.concat(t,"-"))
a-b-c-d-e

-- 指定一个数字 key 为8，不可拼接
> t = {"a", "b", "c", "d", [8] = "e", ["test"] = 123}
> print(table.concat(t,"-"))
a-b-c-d

-- 指定索引从第一位拼到第三位
> t = {"a", "b", "c", "d", [5] = "e", ["test"] = 123}
> print(table.concat(t, "-", 1, 3))
a-b-c

-- 指定的索引超过了 table 的 key
> t = {"a", "b", "c", "d", [5] = "e", ["test"] = 123}
> print(table.concat(t, "-", 1, 10))
stdin:1: invalid value (nil) at index 6 in table for 'concat'
stack traceback:
        [C]: in function 'concat'
        stdin:1: in main chunk
        [C]: ?
```

当把一个 table 传给一个 function，或存为一个新的变量时，Lua 不会创建一个新的 table 副本，而是同一个 table 的引用：

```lua
> t = {}
> u = t
> u.foo = "bar"
> = t.foo
bar
> function f(x) x[1] = 2 end
> f(t)
> = u[1]
2
```

---

### Function

#### 多返回值

Lua 中的函数可以返回多个值：

```lua
> f = function ()
>>  return "x", "y", "z" -- 返回三个值
>> end
> a, b, c, d = f() -- 将三个值返回给四个变量，第四个会被设置成 nil
> = a, b, c, d
x y z nil
> a, b = (f()) -- 使用括号包裹，会丢弃多余的返回值
> = a, b
x, nil
> = "w"..f() -- 使用函数作为一个子表达式，同样会丢弃多余的返回值
wx
> print(f(), "w") -- 作为另一个函数的参数时，也会丢弃多余的返回值
x w
> print("w", f()) -- 但当函数调用是最后一个参数时，会返回所有返回值
w x y z
> print("w", (f())) -- 这时使用括号包裹也会起作用
w x
> t = {f()} -- 多个返回值可以直接被存储在 table 中
> = t[1], t[2], t[3]
x y z
```

在最后一个例子中需要注意的是，如果函数返回的值中有 nil，那使用 # 操作符获取 table 中值的数量时就不一定准确了：

```lua
> f = function () return "x","y",nil,"z" end
> t = {f()}
> print(#t)
4
```

#### 函数作为参数

将函数作为参数或用它们返回值是一个有用的功能，这里有个很好的例子——table.sort：

```lua
{% raw %}
> list = {{3}, {5}, {2}, {-1}}
> table.sort(list)
attempt to compare two table values
stack traceback:
        [C]: in function 'sort'
        stdin:1: in main chunk
        [C]: in ?
> table.sort(list, function (a, b) return a[1] < b[1] end)
> for i,v in ipairs(list) do print(v[1]) end
-1
2
3
5
{% endraw %}
```

#### 可变参数

函数可以在参数列表的末尾使用 ... 作为可变参数：

```lua
> f = function (x, ...)
>>  x(...)
>> end
> f(print, "1 2 3")
1 2 3
```

想从 ... 中获取特定的项可以使用 select() 函数，该函数可指定从第几位开始获取入参，而且还可以接收 "#" 作为参数，获取可变参数的长度：

```lua
> f=function(...) print(select("#", ...)) print(select(3, ...)) end
> f(1, 2, 3, 4, 5)
5
3 4 5
```

... 还可以被打包(packed)进一个 table：

```lua
> f=function(...) tbl={...} print(tbl[2]) end
> f("a", "b", "c")
b
```

一个数组项 table 还可以被拆包(unpacked)成参数列表：

```lua
> f=function(...) tbl={...} print(table.unpack(tbl)) end -- 在 Lua 5.1 中是直接使用 unpack() 方法，不需要前面的 table.
> f("a", "b", "c")
a b c
> f("a", nil, "c") -- undefined result, may or may not be what you expect
```

像前面说到的，如果 table 中有 nil 值存在，就会出现一些乱七八糟的问题，所以从 Lua 5.2 开始加入了 table.pack() 方法，也新增了一个 "n" 字段表示 table 中项的数量：

```lua
> f=function(...) tbl=table.pack(...) print(tbl.n, table.unpack(tbl, 1, tbl.n)) end
> f("a", "b", "c")
3 a b c
> f("a", nil, "c")
3 a nil c
```

#### 递归和尾调用

一个简单的求阶乘的递归函数：

```lua
function factorial(x)
  if x == 1 then
    return 1
  end
  return x * factorial(x-1)
end
```

上面这个递归函数在 x 的值非常大的时候，将有很严重的性能问题，在 return 中调用自身函数时，需要等待下一个调用的返回值，用来乘以当前函数中的 x，这就使得需要保存一些信息在调用栈(call stack)中，每次调用都需要保存，所以栈空间将迅速增长

这个问题可以通过尾调用(tail calls)来优化，这里有一个使用尾调用来重写上述求阶乘的函数：

```lua
function factorial_helper(i, acc)
  if i == 0 then
    return acc
  end
  return factorial_helper(i-1, acc*i)
end

function factorial(x)
  return factorial_helper(x, 1)
end
```

在 factorial_helper 函数中可以看到，return 中仅仅是调用自身函数，不需要等待该次调用的返回，所以不会向调用栈中保存额外的信息

简单的一句话来讲就是：尾调用仅仅是一次跳跃，而不是一次实际的函数调用(a tail call is just a jump, not an actual function call)

这里还有一些例子来说明什么是尾调用、什么不是尾调用(就不翻译了)：

```lua
return f(arg) -- tail call
return t.f(a+b, t.x) -- tail call
return 1, f() -- not a tail call, the function's results are not the only thing returned
return f(), 1 -- not a tail call, the function's results are not the only thing returned
return (f()) -- not a tail call, the function's possible multiple return values need to be cut down to 1 after it returns
return f() + 5 -- not a tail call, the function's return value needs to be added to 5 after it returns
return f().x -- not a tail call, the function's return value needs to be used in a table index expression after it returns
```

最后，Lua 官方也把尾调用称之为“正确的尾调用”([Proper Tail Calls](https://www.lua.org/pil/6.3.html))
