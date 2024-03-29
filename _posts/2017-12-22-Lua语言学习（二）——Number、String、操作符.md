---
layout: post
title: Lua语言学习（二）——Number、String、操作符
date: 2017-12-22
categories:
- 技术
tags: [Lua]
status: publish
type: post
published: true
---

**接下来的几篇是根据[ Lua-Users wiki ](http://lua-users.org/wiki/TutorialDirectory)梳理的一些细节**

---

### 赋值

```lua
> i = 7
> i, x = i + 1, i
> print(i, x)
8   7
```

Lua 会先计算等号右侧的 i + 1 和 i 的值，然后第二行就变成了```i, x = 8, 7```，然后从右向左分配，x = 7，i = 8

关于从右向左分配可以看下面这个例子：

```lua
> a, a = 1, 2
> print(a)
1
```

Lua 会从右向左，先执行 a = 2，再执行 a = 1，所以最终 a 的值是1

---

### Number

Lua 不像其他高级语言有各种 Number 类型，默认只有**双精度浮点型**

把字符串转换成数字可以用 tonumber() 函数：

```lua
> = tonumber("123") + 4
127
> x = tonumber("123.456e5")
> print(x)
12345600
```

另外也可以用算术运算符直接强制转换字符串，如果字符串不能转为 Number 类型，则报错：

```lua
> = 100 + "7"
107
> = "12" * 3
36
> = "1" + "2" - "3"
0
> = "hello" + 1
stdin:1: attempt to perform arithmetic on a string value
```

Lua 中这种自动类型转换称为'Coercion'，翻译过来应该也就是强制吧。。

有一点要注意的是：比较运算符(==、\~=、<、>、<=、>=)并不会强制转换字符串，如果使用了错误的类型，除 ==、\~= 外，其他比较运算符还会抛出错误：

```lua
> = 100 == "100"
false
> = 100 ~= "hello"
true
> = 100 ~= {}
true
> = 100 == tonumber("100")
true
> = 100 <= "100"
stdin:1: attempt to compare number with string
```

---

### String

Lua 中也可以使用转义序列，比如其他高级语言中常见的 \n \t 之类的，但在双方括号`[[]]`定义
的字符串中这样做是不行的：

```lua
> = 'hello\nNew line\tTab'
hello
New line        Tab
> = [[hello\nNew line\tTab]]
hello\nNew line\tTab
```

双方括号同时还支持嵌套，但需要给最外层的方括号加上等号，等号的个数任意，只要求数量相同即可：

```lua
> = [[one [[two]] one]]        -- bad
stdin:1: nesting of [[...]] is deprecated near '['
> = [=[one [[two]] one]=]      -- ok
one [[two]] one
> = [===[one [[two]] one]===]  -- ok too
one [[two]] one
> = [=[one [ [==[ one]=]       -- ok. nothing special about the inner content.
one [ [==[ one
```

使用 .. 符号可以拼接字符串，连接数字类型时，也可以强制自动转换成字符串类型，但在数字后面使用 .. 符号时，需要加一个空格，否则报错：

```lua
> = "test" .. 123
test123
> = type("test" .. 123)
string
> = "test"..1.."test"
stdin:1: malformed number near '1..'
> = "test" .. 1 .. "test"
test1test
```

需要注意的是，使用 .. 符号做大量拼接时会很慢，因为每一次连接操作都会在内存中分配一个新的 string 对象，网站给了三个示例：

```lua
-- slow
local s = ''
for i=1,10000 do s = s .. math.random() .. ',' end
io.stdout:write(s)

-- fast
for i=1,10000 do io.stdout:write(tostring(math.random()), ',') end

-- fast, but uses more memory
local t = {}
for i=1,10000 do t[i] = tostring(math.random()) end
io.stdout:write(table.concat(t,','), ',')
```

---

### 操作符

关系运算符( ==、~=、<、>、<=、>=)用于返回 true 或 false

除了常见的数字比较、字符串比较，如果类型不同或指向不同的对象，那对象也就不相同：

```lua
> = {} == "table"
false
> = {} == {}      --这里创建了两个不同的 table
false
> test = {}
> test2 = test
> = test == test2 --两个对象引用的是同一个 table
true
```

在 Lua 中只有 nil 和 false 代表 false，其他都是 true，包括了0

Lua 提供三种逻辑运算符 and or not

#### not

not 比较容易理解，除 not nil、not false 为 true，其他全是 false

```lua
> = true, false, not true, not false
true    false   false   true
> = not nil
true
> = not not true
true
> = not "foo"
false
```

#### and

Lua 中的 and 操作符相比其他高级语言就有一点"奇怪"了，如果第一个参数是 false 或 nil，那就不会执行第二个参数，直接返回第一个参数：

```lua
> = false and true
false
> = nil and true
nil
> = nil and false
nil
> = nil and "hello", false and "hello"
nil     false
> = false and print("hello") --print方法没有执行
false
```

当第一个参数为 true 时，就会返回第二个参数：

```lua
> = true and false
false
> = true and true
true
> = 1 and "hello", "hello" and "there"
hello   there
> = true and nil
nil
> = true and print("hello") --print方法执行，并返回print("hello")
hello
nil
```

上个例子中执行 print("hello") 返回了 nil 感觉有点奇怪，就再多测试了一下：

```lua
> = nil == print("hello")
hello
true
```

#### or

or 和 and 一样有点"奇怪"，当第一个参数为 true 时，就不会再执行第二个参数：

```lua
> = true or false
true
> = true or nil
true
> = "hello" or "there", 1 or 0
hello   1
> = true or print("hello") --print方法没有执行
true
```

当第一个参数为 false 或 nil 时，会返回第二个参数：

```lua
> = false or true
true
> = nil or true
true
> = nil or "hello"
hello
> = false or print("hello") --print方法执行，并返回print("hello")
hello
nil
```

or 的这种特性可以用于设置默认值，这里有一个例子：

```lua
> function foo(x)
>>  local value = x or "default"
>>  print(value, x)
>> end
>
> foo()
default nil
> foo(1)
1       1
> foo(true)
true    true
> foo("hello")
hello   hello
```
