# Lua Gotchas

'Gotchas' are Lua features which are often confusing at first, if you are used to other languages.

Here is a short list compiled from the Lua mailing list, particularly by Timothy Hunter.

## 1 Calling methods

If you are used to other languages, then `obj.method()` looks like a method call, but really it is just looking up 'method' in the context of `obj` and then calling that function. To make a proper method call, use `obj:method()` which is 'syntactical sugar' for `obj.method(obj)` - that is, the colon passes the object as the first argument as `self`.  Common methods encountered in basic Lua programming are the string methods, like `s:find("hello")` being short for `string.find(s,"hello")`.

## 2 Values which are false

C-like languages regard `0` as equivalent to `false`, but this is not true for Lua. Only an explicit `false` or `nil` are equivalent to `false`.  When in doubt, make the condition explicit, e.g. `if val == nil then ... end` unless the value is actually `boolean`.

## 3 Undeclared variables

### 3.1 .. are global by default

Like all dynamic languages, you don't have to declare variables. However, the scope of what gets implicitly declared is not the same. For instance, an undeclared variable is local to the function in Python, and you need the keyword `global` to tell the interpreter to look outside. In Lua, the situation is exactly opposite; unless declared using `local`, Lua will look up a name in the current environment, which is usually `_G`.

Also, the value of an undeclared variable is `nil`, because there is no special `undefined` value. Since `nil` may be an acceptable value for a variable, typing errors can bite deep.

In larger programs, this can lead to madness, so people often use a technique which traps undefined global access - see [strict model](index.html#T1.6).

### 3.2 Variables must be declared before use.

This might seem an obvious point, but in JavaScript function declarations are 'hoisted' so that they can be declared in any order. This also works in Python:

    def foo():
        def bar ():
            zog()
        def zog():
            print "Zog!"
        bar()

but the equivalent Lua does not because `zog` has not yet been declared:

    function foo()
     local function bar() zog() end
     local function zog() print "Zog!" end
     bar()
    end

## 4 All numbers are floating point

The good news is that any integer up to 2^52 can be represented exactly by a 64-bit double-precision value. But don't expect equality to work always for floating-point values! (This is more a general programming gotcha, since you must become used to finite precision.)

Lua can be compiled only to use integer arithmetic on embedded platforms that lack hardware floating-point support.

## 5 When does automatic conversion to string occur?

`10+"20"` does work in Lua, but `10=="10"` is not true. In an arithmetic expression, Lua will attempt to convert strings to numbers, otherwise you will get a 'attempt to perform arithmetic on a string value'.  Fortunately, string concatenation is a separate operator, so JavaScript-style weirdness does not happen so often: `"10".."20"` is unambiguously "1020".

## 6 Tables are both arrays and dictionaries

Lua tables are the ultimate one-stop-shop of data structures. It can be confusing that a table can behave like both kinds of containers at once, such as `{1,2,3,enable=true}`. So you will see reference to the array-part and the 'hash' part of a table.

### 6.1 Arrays start at one

This gets people because starting at zero is more common, and the habit is hard to break. The problem is that `t[0] = 1` is not considered a problem, but then the length of the array will be off-by-one.  The value is in the table, but is in the 'hash' or 'dictionary' part. The length operator `#` only applies from 1 to the last non-`nil` element.

### 6.2 Table keys

The 'key' in `t.key` is a constant 'key', not the variable named 'key'. This is because `t.key` is the same as `t["key"]`. If you want to index by a variable, say `t[key]`. This 'quoting' also happens in table constructors, e.g. `t={key=1}` sets the value of `t.key`. But, this can only work for valid Lua names that aren't keywords or contain special characters. So you must say `t={["for"]=1,...}' and `t["for"]`.

JavaScript programmers will be comfortable with Lua tables, but remember there is no distinction between 'undefined' and 'null'.

### 6.3 Undefined field access is a feature

In many languages, trying to access a non-existent key in a list or dictionary causes an exception; in Lua the result is simply `nil`. This is unambiguous because `nil` cannot be usefully put into tables.

### 6.4 Arrays with holes are bad news

Try not to put `nil` in arrays. It will work if you know what you're doing, but the length operator `#` will be confused and standard table functions like `table.sort()` will complain bitterly.

Sparse arrays have their uses, if you remember not to depend on `#`:

    > t = {}
    > t[1] = 1
    > t[4] = 4
    > t[10] = 10
    > = #t  --> NB!
    1
    > for k,v in pairs(t) do print(k,v) end
    1       1
    4       4
    10      10

### 6.5 Element order with pairs() is arbitrary

Yes, you can use `pairs()` with an 'array', but don't expect to get the keys in ascending order. That is the first reason for `ipairs()` which makes that guarantee. The second reason for `ipairs()` is that it only goes over the 'array' part of a table.

## 7 When to use parentheses with functions?

Lua is different in that strings require parentheses if you want to call string methods, such as `('%s=%d'):format('hello',42)`.

But there are important cases when you can leave out parentheses, such as `print 'hello'` and `myfunc {1,2,3; enable=true}`. If the single argument of a function is a string or a table constructor, then they can be safely left out.

(The old-style Python `print 'hello'` has another meaning, since `print` is a statement.)

## 8 Expressions with multiple values

Expressions can have multiple values, so you can swap two values with one line like so:

    x,y = y,x

A powerful and unusual feature, but there are several things to be aware of.

### 8.1 Functions can return multiple values

A Lua function call can return multiple values. A number of the string functions work like this,for instance, `string.find` will return the start index and end index of the match, plus any _captures_.

    > s = "hello dolly"
    > print(s:find("(dolly)"))
    7       11      dolly

(Note that the parentheses in the string pattern have special meaning, like in standard regular expressions.)

It is easy to write a function that returns multiple values:

    function sumprod(x,y) return x+y, x*y end

(The comma is never an operator in Lua; it is only a delimiter.)

Furthermore, these multiple return values are treated specially in function calls and table constructors. The expression `{sumprod(10,20)}` is the table `{30,200}` and `print(sumprod(10,20))` has the same effect as `print(30,200)`.

The function `unpack` goes the other way:

    > print(unpack{1,2,3})
    1       2       3

This is a powerful and efficient feature but it can bite you. For instance, consider this function that turns a string into hexadecimal bytes:

    function escape(s)
      return s:gsub('.',function(s) return ('%2X'):format(s:byte(1)) end)
    end

But there is a gotcha:

    > = escape(' ')
    20      1

`string.gsub` returns the result of the substitution plus the number of substitutions made - so this function can mess up code that is expecting exactly one return code.  The best solution is to enclose the expression in parentheses:

    function escape(s)
      return (s:gsub('.',function(s) return ('%2X'):format(s:byte(1)) end))
    end

(Or you could assign the result of `gsub` to a local variable and return that.)

The rule for expanding multiple results only works for the _last_ argument, so be careful with expressions like `{f(),g()}` - if `f()` returned multiple values, then only the _first_ one would be used. (There has been a [proposal](http://lua-users.org/lists/lua-l/2009-08/msg00242.html) for an operator that would 'unpack' multiple values in any position.)

### 8.2 A Superficial similarity with Python

Note that the following statement works in both Lua and Python, but in very different ways:

    x,y = fun()

In Lua, the function `fun` returns two values, in Python it returns one `tuple` value, which it then helpfully unpacks in this context. This is one of the many little things that makes Lua the 'speed queen' of dynamic languages, since there is no extra hidden overhead with this assignment.

A Python programmer may expect this to also implicitly unpack a table:

    x,y = {1,2}

But there is no such magic in Lua, and you must explicitly use `unpack`. It's a gotcha because it is not an error; `x` becomes the table value, and `y` becomes `nil`.

### 8.3 Multiple Assignments in Declarations

In most languages, several variables can be declared and assigned at the same time:

    var x = 1, y = 2

In Lua the equivalent is

    local x,y = 1,2

and you will get an 'unexpected symbol near '='' error if you try the first syntax.

## 9 Values and references

If `x=10` and `y=10` then we know that `x==y`, but the same is not true for tables and other objects:

    local t1,t2 = {},{}
    print(t1 == t2)
    ==> false

`t1` and `t2` are _distinct_ values; tables are never compared 'element by element'. It is straightforward to do this in Lua; for example see how `deepcompare` is defined in [Penlight](http://github.com/stevedonovan/Penlight/blob/master/lua/pl/tablex.lua)

In the same way `t1 = t2` does not make a copy of `t2`. The variable `t1` will become another reference to the value of `t2`. Assigning to `t1[1]` will change `t2[1]`, because they simply point to the same object.

Now, how does string equality work? Unlike Java, `s1==s2` _will_ work on equal strings, character by character. This is because there is only ever one instance of any particular string stored in memory, so comparison is very quick (this is called _interning_), for an extra cost of string creation.  This can work because strings are 'immutable', there is no way in Lua to modify the contents of a string directly.

The same lessons taught to Java programmers apply here: building up a large string by concatenation creates a lot of temporary strings that can stress the garbage collector. The Lua way is to put the strings into a table and then use `table.concat`.

If you are a C programmer, this may seem massively inefficient, but the answer is that Lua strings are not the right data structure for modification.  Representing a string as a table of string fragments (sometimes called a 'rope') is much more efficient.

## 10 Using metatables

Lua has powerful meta-programming abilities through mechanism of [metatables](http://www.lua.org/pil/13.html). Different table values can be given common behaviour by giving them the same metatable, and so you can think of it as a generalisation of the 'class' concept in other dynamic languages.

### 10.1 __index is a fallback

The metatable can have _metamethods_ like `__index`. This is called when table indexing fails, and in this example forces a table index to return a default zero value if out-of-bounds.

    t = {1,2,3}

    setmetatable(t,{
      __index = function(t,k) return 0 end
    })

    print(t[1],t[5])
    ---> 1   0

Cute, and incredibly powerful. In Lua 4.0 the equivalent concept was called a 'fallback' and that word describes the behaviour very well. A common misunderstanding is that `__index` overrides all table accesses, but this is not so.  In a similar way, `__newindex` only kicks in if the key does not exist in the table. (If you do need to do something for each table key access, then _proxy tables_ are the suggested solution. This technique involves deliberately keeping the table empty, and keeping the implementation data separate. Then `__index` will always fire and you can return whatever you like - this can be used to implement true _properties_ with getters and setters.)

### 10.2 __index can be a table

A common way to implement classes looks like this:

    C = {}
    C.__index = C

    function C:m1() return self:m2() end  -- note the implicit self with : shortcut
    function C:m2() return self.value end

To construct a new instance of `C`, you make a new table and set its metatable to `C`.

    function C.new(value)
      local obj = setmetatable({},C)
      obj.value = value
      return obj
    end

The key to this trick is `C.__index = C`; people often think that merely _putting_ functions in the metatable will allow objects to access them, but all unknown field accesses have to go through `__index`, and this assignment lets objects find their functions in the metatable.

### 10.3 __eq cannot be used to compare arbitrary values

Looking at the [manual](http://www.lua.org/manual/5.1/manual.html#2.8) shows you a full list of the available metamethods. For instance, you can override the usual equality operation and make it do something else, for instance do a true element-by-element comparison for lists.

There is a gotcha with `__eq`; unlike the arithmetical metamethods, it insists that both arguments have the same metatable.
