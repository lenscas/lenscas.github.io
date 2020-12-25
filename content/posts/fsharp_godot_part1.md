---
title: "F# and Godot part 1"
date: 2020-12-24T01:23:40+01:00
draft: false
---

In this part I will touch on why I chose F# instead of the many other languages that I know and could've used with godot. In addition it also goes over on how to set it up.

Part 2 will go over how to do some basic things in Godot with F#, like how to emit signals.

In part 3 we take a bit deeper dive into F# and use some of its features to extend one of Godot's libraries so it becomes easier to use. This will touch on features and syntax that this part won't cover. 

## Why F#?

Simply put, it was the one that interested me the most from the available options and also had the most potential to be enjoyable to write.


## Why not GDScript or Rust.

I already know that GDScript and I would not get along, simply because I dislike dynamic typing and I really like to work with lambdas.

Rust was not an option because you can’t compile to wasm with GDNative. Even if you could, it at the very least looks like the crates (the name rust uses for libraries) to work with godot are still very young, changing and may require “unsave” code to work or are dependent on so much “unsave” code that it wouldn’t surprise me if the save abstractions are not yet perfect.


## C# vs F#

Let's start with why I’m not a huge fan of C#, after that lets dive into a few things I like about it and finally how F# does almost everything better, or at the very least better for me.

First of, C# has implicit null. This means that I can’t see from the type signature if a method may not return a proper value. It may not sound like a huge deal but if you ever used a language with a good Option&lt;T> type you will quickly start to miss it in languages that don’t offer it. There is also a reason that “null” is often referred to as the billion dollar mistake.

C# is also not great at inferring your types. There are reasons for that but after spending a good amount of time with a language that is REALLY good at it it just becomes a chore and sigh worthy every time I need to tell C# that the list I made and directly assigned to an property of type List&lt;int> is indeed a list of int. Come on!

It also heavily relies on exceptions, again something that is not visible in the type signature of a function and is thus not something I'm a fan of. 

As for things that I do like about C#?
Well, Linq and extension methods are nice. With Linq it becomes super easy to work with any kind of collection instead of being stuck with ugly foreach loops. Extension methods are just nice to help with chaining methods, though Rust does it better with its trait system.

C# also has good lambda support, and async/await is a good bit less messy than with F#, ESPECIALLY when godot gets involved.

So, lets now finally take a look at F#. First off, it uses Option&lt;T> instead of implicit nulls and has many functions to make working with options nice [https://fsharp.github.io/fsharp-core-docs/reference/fsharp-core-optionmodule.html](https://fsharp.github.io/fsharp-core-docs/reference/fsharp-core-optionmodule.html).F# is also A LOT better at inferring your types (arguably its too good at times…) and it relies a lot less on exceptions, though I am aware that you can't always escape them.

As for Linq, F# has both access to Linq and it has its own versions of the methods provided by Linq, which actually have the correct/standard name instead of Linq’s “sql inspired” naming scheme. But the goodies don’t end there, as it has an amazing way to chain functions (`|>`), Lazy&lt;T>, pattern matching, tail call optimization and if's are expressions instead of statements, removing the need for C#'s ` C ? A:B` syntax.

Of course, it is not perfect, especially not when combined with Godot. Here are some pain points you will run into

First, the syntax is rather different. How different? Well, the following is a a simple add function and how to call it
```fs
let add a b =
    a + b
    
add 10 20
```
This also means that methods are harder to chain, as you can’t simply do `a.b(param).c()` instead it would be written as `(a.b param).c()` . Luckily F# has another way to chain functions, but it would require you to stop writing methods and instead use functions that take it as a parameter. The way to chain functions in F# is done using `|>` which results in code like

```fs
let x =  
    param1 
    |> function1 
    |> function2 thatGetsAnotherFirstParameter 
    |> function3
```
This code roughly translates to this C#
```cs
var x = function3(
    function2(
        thatGetsAnotherFirstParameter,
        function1(
            Param1
        )
    )
);
```

Another downside is that it is really easy to accidentally return a value to godot that is not void. This will result in run time errors, also signals and F# types don’t mix too well either. Lastly, async/await works completely different in F# compared to C#. So, while with C# you can simply use async/await instead of GDscript's “yield”, you need to depend on “ply” in F# and even make a wrapper for godot’s values you normally can await on. Luckily, task based async still works ok (just need to use Async.AwaitTask) and ply also helps with task based async.

I personally didn't dive too much into async though. For my game a WASM build is a MUST have, and both C#'s and F#'s async system are broken for WASM at time of writing [https://github.com/godotengine/godot/issues/34506](https://github.com/godotengine/godot/issues/34506)


## How to set it up

First thing you need to understand is that godot is hardcoded to only have C# scripts attachable to it. This makes it a bit more annoying to use other languages through mono but there is a [plugin](https://github.com/willnationsdev/godot-fsharp-tools) that helps.

You just need a new folder that is an F# project and has godot loaded. The library should also be loaded by the C# project that godot made. The plugin can do this for you. Simply install it, and go to project -> tools -> setup F# project. Pro tip: ***DO NOT*** select the root folder of your godot project. It will get stuck in an infinite loop if you do this. Just make a subdirectory.

With that done you should now enable “auto generate F# scripts”. Simply go to project -> project settings -> mono -> F# tools and fill in the default namespace and default path and then enable it.

Now, the next time you make a C# script a new F# script will be created and the C# script will automatically extend the type in said F# script. Thus automating pretty much the entire workflow.

One thing: It may be that your editor is stupid and doesn’t want to recognize that the F# library exists when you open one of the C# files. I have not found a solution for this, just make sure that `dotnet build` builds without errors and/or godot is able to run the project. Unless you want to use both C# and F# (Which is also a good strategy) your C# files are staying pretty much empty anyway so having those be broken in your editor doesn't matter much.

## End part 1.
Next part will touch on how to do some basic things in F#. If you can't wait there is also [this overview](http://www.lkokemohr.de/fsharp_godot_part2.html). It doesn't go over everything I'm planning to do but it is still a good start and it is what I originally used.

