## Introduction

### Purpose

It is often said of Lua that it does not include batteries. That is because the goal of Lua is to produce a lean expressive language that will be used on all sorts of machines, (some of which don't even have hierarchical filesystems). The Lua language is the equivalent of an operating system kernel; the creators of Lua do not see it as their responsibility to create a full software ecosystem around the language. That is the role of the community.

A principle of software design is to recognize common patterns and reuse them. If you find yourself writing things like `io.write(string.format('the answer is %d ',42))` more than a number of times then it becomes useful just to define a function `printf`. This is good, not just because repeated code is harder to maintain, but because such code is easier to read, once people understand your libraries.

Penlight captures many such code patterns, so that the intent of your code becomes clearer. For instance, a Lua idiom to copy a table is `{unpack(t)}`, but this will only work for 'small' tables (for a given value of 'small') so it is not very robust. Also, the intent is not clear. So `tablex.deepcopy` is provided, which will also copy nested tables and and associated metatables, so it can be used to clone complex objects.

The default error handling policy follows that of the Lua standard libraries: if a argument is the wrong type, then an error will be thrown, but otherwise we return `nil,message` if there is a problem. There are some exceptions; functions like `input.fields` default to shutting down the program immediately with a useful message. This is more appropriate behaviour for a _script_ than providing a stack trace. (However, this default can be changed.) The lexer functions always throw errors, to simplify coding, and so should be wrapped in `pcall`.

By default, the error stacktrace starts with your code, since you are not usually interested in the internal details of the library. ??

If you are used to Python conventions, please note that all indices consistently start at 1.

The Lua function `table.foreach` has been deprecated in favour of the `for in` statement, but such an operation becomes particularly useful with the higher-order function support in Penlight. Note that `tablex.foreach` reverses the order, so that the function is passed the value and then the key. Although perverse, this matches the intended use better.

The only important external dependence of Penlight is LuaFileSystem (`lfs`), and if you want `dir.copyfile` to work cleanly on Windows, you will need `alien` as well. (The fallback is to call the equivalent shell commands.)

Some of the examples in this guide were created using [ilua](http://lua-users.org/wiki/InteractiveLua), which doesn't require '=' to print out expressions, and will attempt to print out table results as nicely as possible.  This is also available under Lua for Windows, as a library, so the command `lua -lilua -s` will work (the s option switches off 'strict' variable checking, which is annoying and conflicts with the use of `_DEBUG` in some of these libraries.

### To Inject or not to Inject?

It was realized a long time ago that large programs needed a way to keep names distinct by putting them into tables (Lua), namespaces (C++) or modules (Python).  It is obviously impossible to run a company where everyone is called 'Bruce', except in Monty Python skits. These 'namespace clashes' are more of a problem in a simple language like Lua than in C++, because C++ does more complicated lookup over 'injected namespaces'.  However, in a small group of friends, 'Bruce' is usually unique, so in particular situations it's useful to drop the formality and not use last names. It depends entirely on what kind of program you are writing, whether it is a ten line script or a ten thousand line program.

So the Penlight library provides the formal way and the informal way, without imposing any preference. You can do it formally like:

    local utils = require 'pl.utils'
    utils.printf("%s\n","hello, world!")

or informally like:

    require 'pl'
    utils.printf("%s\n","That feels better")

`require 'pl'` makes all the separate Penlight modules available, without needing to require them each individually..   Generally, the formal way is better when writing modules, since then there are no global side-effects and the dependencies of your module are made explicit.

With Penlight after 0.9, please note that `require 'pl.utils'` no longer implies that a global table `pl.utils` exists, since these new modules are no longer created with `module()`.

Penlight will not bring in functions into the global table, or clobber standard tables like 'io'.  require('pl') will bring tables like 'utils','tablex',etc into the global table _if they are used_. This 'load-on-demand' strategy ensures that the whole kitchen sink is not loaded up front,  so this method is as efficient as explicitly loading required modules.

You have an option to bring the `pl.stringx` methods into the standard string table. All strings have a metatable that allows for automatic lookup in `string`, so we can say `s:upper()`. Importing `stringx` allows for its functions to also be called as methods: `s:strip()`,etc:

    require 'pl'
    stringx.import()

or, more explicitly:

    require('pl.stringx').import()

A more delicate operation is importing tables into the local environment. This is convenient when the context makes the meaning of a name very clear:

    > require 'pl'
    > utils.import(math)
    > = sin(1.2)
    0.93203908596723

`utils.import` can also be passed a module name as a string, which is first required and then imported. If used in a module, `import` will bring the symbols into the module context.

Keeping the global scope simple is very necessary with dynamic languages. Using global variables in a big program is always asking for trouble, especially since you do  not have the spell-checking provided by a compiler. The `pl.strict` module enforces a simple rule: globals must be 'declared'.  This means that they must be assigned before use; assigning to `nil` is sufficient.

    > require 'pl.strict'
    > print(x)
    stdin:1: variable 'x' is not declared
    > x = nil
    > print(x)
    nil

The `strict` module provided by Penlight is compatible with the 'load-on-demand' scheme used by `require 'pl`.

`strict` also disallows assignment to global variables, except in the main program. Generally, modules have no business messing with global scope; if you must do it, then use a call to `rawset`. Similarly, if you have to check for the existance of a global, use `rawget`.

If you wish to enforce strictness globally, then just add `require 'pl.strict'` at the end of `pl/init.lua`.

### What are function arguments in Penlight?

Many functions in Penlight themselves take function arguments, like `map` which applies a function to a list, element by element.  You can use existing functions, like `math.max`, anonymous functions (like `function(x,y) return x > y end`), or operations by name (e.g '*' or '..').  The module `pl.operator` exports all the standard Lua operations, like the Python module of the same name. Penlight allows these to be referred to by name, so `operator.gt` can be more concisely expressed as '>'.

Note that the `map` functions pass any extra arguments to the function, so we can have `ls:filter('>',0)`, which is a shortcut for `ls:filter(function(x) return x > 0 end)`.

Finally, `pl.func` supports _placeholder expressions_ in the Boost lambda style, so that an anonymous function to multiply the two arguments can be expressed as `_1*_2`.

To use them directly, note that _all_ function arguments in Penlight go through `utils.function_arg`. `pl.func` registers itself with this function, so that you can directly use placeholder expressions with standard methods:

    > _1 = func._1
    > = List{10,20,30}:map(_1+1)
    {11,21,31}

Another option for short anonymous functions is provided by `utils.string_lambda`; since 0.9 you have to explicitly ask for this feature:

    > L = require 'pl.utils'.string_lambda
    > = List{10,20,30}:map (L'|x| x + 1')
    {11,21,31}

### Pros and Cons of Loopless Programming

The standard loops-and-ifs 'imperative' style of programming is dominant, and often seems to be the 'natural' way of telling a machine what to do. It is in fact very much how the machine does things, but we need to take a step back and find ways of expressing solutions in a higher-level way.  For instance, applying a function to all elements of a list is a common operation:

    local res = {}
    for i = 1,#ls do
        res[i] = fun(ls[i])
    end

This can be efficiently and succintly expressed as `ls:map(fun)`. Not only is there less typing but the intention of the code is clearer. If readers of your code spend too much time trying to guess your intention by analyzing your loops, then you have failed to express yourself clearly. Similarly, `ls:filter('>',0)` will give you all the values in a list greater than zero. (Of course, if you don't feel like using `List`, or have non-list-like tables, then `pl.tablex` offers the same facilities. In fact, the `List` methods are implemented using `tablex' functions.)

A common observation is that loopless programming is less efficient, particularly in the way it uses memory. `ls1:map2('*',ls2):reduce '+'` will give you the dot product of two lists, but an unnecessary temporary list is created.  But efficiency is relative to the actual situation, it may turn out to be _fast enough_, or may not appear in any crucial inner loops, etc.

Writing loops is 'error-prone and tedious', as Stroustrup says. But any half-decent editor can be taught to do much of that typing for you. The question should actually be: is it tedious to _read_ loops?  As with natural language, programmers tend to read chunks at a time. A for-loop causes no surprise, and probably little brain activity. One argument for loopless programming is the loops that you _do_ write stand out more, and signal 'something different happening here'.  It should not be an all-or-nothing thing, since most programs require a mixture of idioms that suit the problem.  Some languages (like APL) do nearly everything with map and reduce operations on arrays, and so solutions can sometimes seem forced. Wisdom is knowing when a particular idiom makes a particular problem easy to _solve_ and the solution easy to _explain_ afterwards.

### Generally useful functions.

The function `printf` discussed earlier is included in `pl.utils` because it makes properly formatted output easier. (There is an equivalent `fprintf` which also takes a file object parameter, just like the C function.)

Utility functions like `is_callable` and `is_type` help with identifying what kind of animal you are dealing with. Obviously, a function is callable, but an object can be callable as well if it has overriden the `__call` metamethod. The Lua `type` function handles the basic types, but can't distinguish between different kinds of objects, which are all tables. So `is_type` handles both cases, like `is_type(s,"string")` and `is_type(ls,List)`.

A common pattern when working with Lua varargs is capturing all the arguments in a table:

    function t(...)
        local args = {...}
        ...
    end

But this will bite you someday when `nil` is one of the arguments, since this will put a 'hole' in your table. In particular, `#ls` will only give you the size upto the `nil` value.  Hence the need for `table.pack` - this is a new Lua 5.2 function which Penlight defines also for Lua 5.1.

    function t(...)
        local args,n = table.pack(...)
        for i = 1,n do
          ...
        end
    end

The 'memoize' pattern occurs when you have a function which is expensive to call, but will always return the same value subsequently. `utils.memoize` is given a function, and returns another function. This calls the function the first time, saves the value for that argument, and thereafter for that argument returns the saved value.  This is a more flexible alternative to building a table of values upfront, since in general you won't know what values are needed.

    sum = utils.memoize(function(n)
        local sum = 0
        for i = 1,n do sum = sum + i end
        return sum
    end)
    ...
    s = sum(1e8) --takes time!
    ...
    s = sum(1e8) --returned saved value!

Penlight is fully compatible with Lua 5.1, 5.2 and LuaJIT 2. To ensure this, `utils` also defines the global Lua 5.2 [load](http://www.lua.org/work/doc/manual.html#pdf-load) function when needed.

 * the input (either a string or a function)
 * the source name used in debug information
 * the mode is a string that can have either or both of 'b' or 't', depending on whether the source is a binary chunk or text code (default is 'bt')
 * the environment for the compiled chunk

Using `load` should reduce the need to call the deprecated function `setfenv`, and make your Lua 5.1 code 5.2-friendly.

### Application Support

`app.parse_args` is a simple command-line argument parser. If called without any arguments, it tries to use the global `arg` array.  It returns the _flags_ (options begining with '-') as a table of name/value pairs, and the _arguments_ as an array.  It knows about long GNU-style flag names, e.g. `--value`, and groups of short flags are understood, so that `-ab` is short for `-a -b`. The flags result would then look like `{value=true,a=true,b=true}`.

Flags may take values. The command-line `--value=open -n10` would result in `{value='open',n='10'}`; generally you can use '=' or ':' to separate the flag from its value, except in the special case where a short flag is followed by an integer.  Or you may specify upfront that some flags have associated values, and then the values will follow the flag.

	> require 'pl'
	> flags,args = utils.parse_args({'-o','fred','-n10','fred.txt'},{o=true})
	> pretty.dump(flags)
	{o='fred',n='10'}

`parse_args` is not intelligent or psychic; it will not convert any flag values or arguments for you, or raise errors. For that, have a look at @{08-additional.md.Command_line_Programs_with_Lapp|Lapp}.

An application which consists of several files usually cannot use `require` to load files in the same directory as the main script.  `app.require_here()` ensures that the Lua module path is modified so that files found locally are found first. In the `examples` directory, `test-symbols.lua` uses this function to ensure that it can find `symbols.lua` even if it is not run from this directory.

`app.appfile` will create a filename that your application can use to store its private data, based on the script name. For example, `app.appfile "test.txt"` from a script called `testapp.lua` produces the following file on my Windows machine:

	C:\Documents and Settings\SJDonova\.testapp\test.txt

and the equivalent on my Linux machine:

	/home/sdonovan/.testapp/test.txt

If `.testapp` does not exist, it will be created.

Penlight makes it convenient to save application data in Lua format. You can use `pretty.dump(t,file)` to write a Lua table in a human-readable form to a file, and `pretty.read(file.read(file))` to generate the table again, using the `pretty` module.


### Simplifying Object-Oriented Programming in Lua

Lua is similar to JavaScript in that the concept of class is not directly supported by the language. In fact, Lua has a very general mechanism for extending the behaviour of tables which makes it straightforward to implement classes. A table's behaviour is controlled by its metatable. If that metatable has a `__index` function or table, this will handle looking up anything which is not found in the original table. A class is just a table with an `__index` key pointing to itself. Creating an object involves making a table and setting its metatable to the class; then when handling `obj.fun`, Lua first looks up `fun` in the table `obj`, and if not found it looks it up in the class. `obj:fun(a)` is just short for `obj.fun(obj,a)`. So with the metatable mechanism and this bit of syntactic sugar, it is straightforward to implement classic object orientation.

    -- animal.lua

    class = require 'pl.class'

    class.Animal()

    function Animal:_init(name)
        self.name = name
    end

    function Animal:__tostring()
      return self.name..': '..self:speak()
    end

    class.Dog(Animal)

    function Dog:speak()
      return 'bark'
    end

    class.Cat(Animal)

    function Cat:_init(name,breed)
        self:super(name)  -- must init base!
        self.breed = breed
    end

    function Cat:speak()
      return 'meow'
    end

    class.Lion(Cat)

    function Lion:speak()
      return 'roar'
    end

    fido = Dog('Fido')
    felix = Cat('Felix','Tabby')
    leo = Lion('Leo','African')

    $ lua -i animal.lua
    > = fido,felix,leo
    Fido: bark      Felix: meow     Leo: roar
    > = leo:is_a(Animal)
    true
    > = leo:is_a(Dog)
    false
    > = leo:is_a(Cat)
    true

All Animal does is define `__tostring`, which Lua will use whenever a string representation is needed of the object. In turn, this relies on `speak`, which is not defined. So it's what C++ people would call an abstract base class; the specific derived classes like Dog define `speak`. (Please note that if derived classes have their own constructors, they must explicitly call the base constructor for their base class; this is conveniently available as the `super` method.)

All such objects will have a `is_a` method, which looks up the inheritance chain to find a match.  Another form is `class_of`, which can be safely called on all objects, so instead of `leo:is_a(Animal)` one can say `Animal:class_of(leo)`.

There are two ways to define a class, either `class.Name()` or `Name = class()`; both work identically, except that the first form will always put the class in the current environment (whether global or module); the second form provides more flexibility about where to store the class. The first form does _name_ the class by setting the `_name` field, which can be useful in identifying the objects of this type later. This session illustrates the usefulness of having named classes, if no `__tostring` method is explicitly defined.

    > class.Fred()
    > a = Fred()
    > = a
    Fred: 00459330
    > Alice = class()
    > b = Alice()
    > = b
    table: 00459AE8
    > Alice._name = 'Alice'
    > = b
    Alice: 00459AE8

So `Alice = class(); Alice._name = 'Alice'` is exactly the same as `class.Alice()`.

This useful notation is borrowed from Hugo Etchegoyen's [classlib](http://lua-users.org/wiki/MultipleInheritanceClasses) which further extends this concept to allow for multiple inheritance.

