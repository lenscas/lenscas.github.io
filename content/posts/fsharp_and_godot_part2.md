---
title: "F# and Godot part 2"
date: 2020-12-24T01:23:41+01:00
draft: true
---

In part 1 I went over on why I chose F# and how to set it up with godot. In part 2 I will go over how you do some basic godot things in F# like emitting custom signals.
Part 3 will go deeper in F# and use it to make a library from Godot easier to use.

## Setting a value through this._Ready

F# is a functional style programming language at its core. Null is not seen as idiomatic and even worse F# will try to fight you if you make a property that does not start with some kind of value.

Luckily, F# also comes with a solution. And, if you ask me its a lot better than `this._Ready`.

So, here is a bit of code taken from my card game that illustrates how to get a child Node and store it for later use.

```fs
type LoginScreenFs() as this =
    inherit Control()

    let userNameNode =
        lazy (this.GetNode<LineEdit>(new NodePath("UserName")))
```
Ok, its weird so lets break it down:
```fs
type LoginScreenFs() as this =
```
this simply declares the type we want to create. The `as this` parts makes `this` available for the `let` bindings. `inherit Control()` is used to tell F# what to extend. In this case, we extend the Control node.

Now comes the fun part
```fs
let userNameNode =
    lazy (this.GetNode<LineEdit>(new NodePath("UserName")))
```
`let userNameNode` simply makes a private field. In this case with the value `lazy (this.GetNode<LineEdit>(new NodePath("UserName")))`. We don't have to set the type for `userNameNode` as F# is able to infer it for us to a `Lazy<LineEdit>`.

As for how this works, well easy: `lazy()` takes a single expression and runs it the first time a value is needed. After that, it remembers the value so the next time the value is needed it can simply return the stored value instead of having to run the code again.

So, in other words instead of setting the value of `userNameNode` in `this._Ready` we don't yet give it a useable value and the first time we need a value we search for the node and remember it for next time. Except that we do not have to check ourself, everything is already handled by `userNameNode.Value`.

As for why it is better? Simple: We can NOT forget to give a value to our fields in the `constructor` nor in `this._Ready`. The less there is to forget, the less bugs there are that you can make. The less bugs you can make, the better.

There is of course a downside to doing this. Especially if the code in the Lazy is heavy having it get executed in `this._Process` may cause a stutter despite it only running once. If this happens, then simply try to access the value in `this._Ready` is an easy (though not elegant) fix. This means that it will execute the code inside the Lazy and remembers the value for later.

## Async anything

There are many differences between async in C# and F#. Lets first have some code examples and then break it down.

A basic async function in C#
```cs
public async Task<int> AnAsyncFunction(int a, int b)
{
    var newVal = await DoStuffWithAnInt(a);
    return a + b;
}
```

the same function in F# 
```fs
let AnAsyncFunction a b = 
    async {
        let! newVal = DoStuffWithAnInt a
        return a + b
    }

```
As you can see, everything is different. Which is kind of the theme when switching to F# from C#. Lets break it down:

The first thing you notice is that F# does not have the concept of an `async function`, at least not in the same was as C# does. Instead it makes use of an `async block`. This is technically just a `Computation Expressions` but that is for WAY later. All you have to know is that it tells F# that the code in the block will be dealing with the `Async` type.

Next, you see that F# doesn't use `await` and if you payed good attention you may also see the `!` after the `let` in the block look at `let! newVal = DoStuffWithAnInt a`.

This `!` is very important and tells F# that you don't want the literal value of the function (in this case `Async<int>`) but only care about the actual result. In other words, it works the same as `await` in C#. It actually is a bit more generic than this as the `async` block is not a language construct and you can make your own variants of these blocks, not even limited to Async.

Another difference: F# uses its own type called `Async<T>` instead of using `Task<T>`. You can still use `Tasks` in F# though. Just need to convert them using `Async.AwaitTask` like so `let! a = parameter1 |> someTaskFunction |> Async.AwaitTask`. There are also libraries like Ply that give you a `task` block. This allows you to write the same code as if it was an `async` block, but works with `Task` instead.

A final difference you have to keep in mind: In F# you have to start an async process yourself, use `Async.Start` or one of the many other functions to start them. The all behave a bit differently but are documented so it makes little sense to go over all of them.

## The elephant in the room: Awaiting on signals.
As already said earlier, using `ToSignal()`. In C# you can just use `await` and call it a day. In F# this is sadly enough not the case. This is because of the differences in how F# and C# do async amplified by godot doing it just a bit different again than the standard C# way.

Godot does not use a `Task` for `ToSignal` so, using `Async.AwaitTask` does not help us. If you want to use `ToSignal` you will have to use `ply` and its task block. Additionally, for some weird reason the classes Godot uses are not compatible with `ply` despite implementing everything. So you will also need to wrap the output of `ToSignal` into your own class to make it compatible.

This is wrapper code that I would have used if I didn't gave up on Async all together because of the issues with WASM.

```fs
open System.Runtime.CompilerServices
open Godot

type SignalAwaiter2(wrapped: SignalAwaiter) =
    interface ICriticalNotifyCompletion with
        member this.OnCompleted(cont) = wrapped.OnCompleted(cont)
        member this.UnsafeOnCompleted(cont) = wrapped.OnCompleted(cont)

    member this.IsCompleted = wrapped.IsCompleted

    member this.GetResult() = wrapped.GetResult()

    member this.GetAwaiter() = this
```
Using it is simple (though admittedly, not ideal):
```fs
let someFunc = 
    task {
        let! x = SignalAwaiter2(this.ToSignal(this.GetTree(), "idle_frame"))
    }
```
You could write an extension method to automatically wrap this.ToSignal into a SignalAwaiter, but I stopped using this system all together before I got so far.

## Emitting and listening to (custom) Signals.

Not much has changed with this compared to C#. There are 2 things you need to think about though.

The first is that Godot wants to `marshal` the values passed through Signals. This severely limits what you can send through them. Any F# specific type of type like struct records will NOT work. F# specific stuff like Option/Result will as far as I know also NOT work. Use classes that extend `Godot.Object` if you want to send them over Signals.

Also, there is no way to define a sub type in a class in F#. As a result, I have yet to find a way to define custom signals in F#. HOWEVER it has an easy work around: Just define them in the C# file you already need anyway. As the F# code is loaded as a library you even have access to any F# type you want to send over. It does mean you can't use `nameof` in F# to get the name of the signal when emitting it though.

Other than those 2 things, it works exactly the same. If however these restrictions are a no go however, then both F# and C# have `events` which can fill the same role. You just have to remember to manually unsubscribe from these `events` whenever a node gets removed.

# Next part

The next part will dive deeper into F# and some of it features like `discriminated unions`, `match` and `Computation Expressions`. These will be used to make a wrapper around one of Godot's libraries to make it A LOT nicer to work with. 