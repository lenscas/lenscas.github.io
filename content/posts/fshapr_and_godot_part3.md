---
title: "F# and Godot part 3"
date: 2020-12-25T01:37:41+01:00
draft: true
---

Part 1 has gone over the why I chose F#, and part 2 showed how to do some basic things. This part will actually dive deeper into F# by writing an abstraction for a part of Godot that is a pain to use no matter what language you use.


## The library

The library I have chosen to make easier is Godots HttpClient. Godot offers 2 ways to make httpRequests.

* [HttpRequest](https://docs.godotengine.org/en/stable/tutorials/networking/http_request_class.html) which uses a Node

* [HttpClient](https://docs.godotengine.org/en/stable/tutorials/networking/http_client_class.html#doc-http-client-class) which doesn't need access to the Scene tree but is more annoying to use as a result.

.NET also offers its own http Client. I would recommend to use that if you can but as it requires async code, which as time of writing doesn't work in WASM contexts, using it may simply not be an option.

Similarly, while Godot's `HttpRequest` is easier to use, the fact that it always needs access to the scene tree may make it a poor option regardless.

## SO, MUCH, POLLING!

If you looked at the tutorial for the HttpClient you may see that A LOT of polling is used and the code is not exactly nice to look at either.

Well, luckily both F# and C# have ways to make it nicer, though F# can go one step further and its also nicer to write the abstraction itself.

The plan: we are going to make a `Poll<T>` class that allows us to easily poll, make functions that helps us compose the Poll's so it is easier to work with and last but not least we are going to make our own `Poll` block, allowing us to work with the Poll class almost as if it was just async code.

## Step 1
The first step is to make the Poll class. It needs a way tell you wether something got returned and a way to poll the value. I also like it to be able to peek, so we can check the state of a Poll without having to poll it.

### The return type
The return type isn't that special. It just has 2 states, either we have a value or not. You could use `Option<T>` for this but I decided to make my own type as it seems more clear what the 2 variants mean. The type I came up with is:
```fs
type PollResult<'a> =
    | NotYet
    | Got of 'a
```

Ok, if you have never seen F# this type will probably make no sense to you. A very short explanation is that this is a `discriminated union`. It denotes a type that can have multiple variants, or states. Kind of like C#'s enums but MUCH more powerful

For now, lets break it down:
First: 
```fs
type PollResult<'a> =
```
the `'a` is just how generics types are written in F#. In C# `<T>` is used

Next, the body of the type:
```fs
| NotYet
| Got of 'a
```
the `|` at the start of the line says that this is a new possible variant. So a type like
```fs
type Example = 
    | A
    | B
    | C
```
is a discriminated union that can either be a `A`, `B` or `C`. So far, it is exactly like an enum.

Next line is `| Got of 'a`. the `| Got` means that this is a new variant called `Got`, no surprises there, The `of 'a` however is where the magic starts. It means that this variant also needs to be given a value. In this case of type `'a`.

We can thus instance this type as either
```fs
PollResult.NotYet
```
or
```fs
PollResult.Got 1
```

The next part will explain how to use this type.

### The class itself

What we are going to make is a class which we need to pass a function.
The function does not get any parameters and will return a `PollResult<T>`.
Every time we run Poll.poll() we run the function and return the Result. We also check if the function returns a `PollResult.Got`, in that case we store the value and from then on return that instead.

The class:

```fs
type Poll<'a>(func: unit -> PollResult<'a>) =
    let mutable funcResult: PollResult<'a> = NotYet

    member this.Peek() = funcResult

    member this.Poll() =
        match funcResult with
        | Got (x) -> Got(x)
        | NotYet ->
            let res = func ()
            match res with
            | Got (x) ->
                funcResult <- Got(x)
                res
            | x -> x
```
Ok, this is even more new stuff. Time to break it down again.

```fs
type Poll<'a>(func: unit -> PollResult<'a>) =
```
This creates a new class which gets a single parameter whenever it gets constructed. In F# you don't really have a constructor function like in C#. Instead the entire class is basically a constructor.

The type of functions is written down as `parameterType -> outcomeType`. Keep in mind that in F# functions can be curried. As a result, a function with 2 parameters is written down as `firstParameterType -> secondParameterType -> outcomeType`

`unit` is similar to C#'s void, it basically means that there is no meaningful value as the only value that is of that type is `()` (An empty tuple). Another difference with `void` is that `unit` can be used for generics, while C#'s `void` can not

So, ` unit -> PollResult<'a>` simply means that its a function that takes `()` as its argument and returns a `PollResult<'a>`

```fs
let mutable funcResult: PollResult<'a> = NotYet
```
makes a new private field called `funcResult`. The `mutable` says that we can mutate the value as in F# you can only mutate values if you specifically said that they can.
This will be where we store the result of `func` so we can return that once we have a value.

```fs
member this.Peek() = funcResult
```
This is a public method that takes `()` and return the current `funcResult`.

And now the fun part:
```fs
member this.Poll() =
    match funcResult with
    | Got (x) -> Got(x)
    | NotYet ->
        let res = func ()
        match res with
        | Got (x) ->
            funcResult <- Got(x)
            res
        | x -> x
```
`member this.Poll() =` same as `this.peek`, just a public method declaration.
```fs
match funcResult with
| Got (x) -> Got(x)
| NotYet ->
    let res = func ()
    match res with
    | Got (x) ->
        funcResult <- Got(x)
        res
    | x -> x
```
Ok, now we got to the `match`. Match is similar to a `switch` in C# there are however 2 major differences.

The first is that match will complain if not every variant has a path. The second is that it can also do something known as "