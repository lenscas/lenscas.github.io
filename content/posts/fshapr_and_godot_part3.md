---
title: "F# and Godot part 3"
date: 2020-12-25T01:37:41+01:00
draft: true
---

# Generics
Generics are written a bit differently in F# compared to C#.

This C#
```cs
public T someFunction<T>(T param) 
{
    //function body
}
```
becomes
```fs
let someFunction<'a> (param:'a) :'a = 
    //function body
```
There is not much to say about it. The rest of the post will continue to use C#'s way to define generics in the text as that is what most people are familiar with, but will of course use F#'s way in F#'s codeblocks.

# Using F# features to make Godots HttpClient nice to use

As you may or may not know. The game I am making heavily relies on HTTP requests. This means that I need a good and easy way to make http requests. There are at least 2 clients available without loading extra libraries.

You have access to the HTTPClient provided by .NET and by the one provided by Godot. Now, I would almost always advice people to use the .NET one if you can. It is just nicer to use. Sadly, this requires async code, which at time of writing doesn't work in wasm. So, that Godot's client. 

Godot has 2 ways to make requests. There is [HttpRequest](https://docs.godotengine.org/en/stable/tutorials/networking/http_request_class.html) and [HttpClient](https://docs.godotengine.org/en/stable/tutorials/networking/http_client_class.html#doc-http-client-class). The big difference is that HttpRequest is a Node and works with signals, while HttpClient is a more low level api and thus does not need access to the scene tree.

If you can, use HttpRequest and not HttpClient. Its many times easier. I decided to go with the HttpClient though as it being stuck to the scene tree would make everything more annoying in the long run. Because of this and the fact that you already know enough to work with the HttpRequest class this part will on the HttpClient.

## SO MUCH POLLING!

If you looked at the tutorial for the HttpClient you may see that A LOT of polling is used and the code is not exactly nice to look at either.

Well, luckily both F# and C# have ways to make it nicer, though F# can go one step further.

In short: we are going to make a `Poll<T>` class that allows us to easily poll, make functions that helps us compose the Poll's so it is easier to work with and last we are going to make our own `Poll` block, allowing us to work with the Poll class almost as if it was just async code.

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
First: `type PollResult<'a> =` it just means we have a generic type called PollResult.

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