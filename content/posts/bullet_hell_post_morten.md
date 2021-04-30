---
title: "Card hell post mortem"
date: 2021-04-30T13:49:25+02:00
draft: false
---

This jam was the first time I took F# and Godot to a game jam. I never used either of them before in a gamejam, with earlier projects either using quicksilver and one simple text adventure game with just TS. I also didn't have much experience so far with Godot, because although my main project uses it the Godot side is rather simple. Similarly, F# is a language I only use when doing Godot stuff, as this is the only context where I like it more than Rust.

So, I knew from the start that the road would be kind of bumpy and though I was right about that, I sadly was right for the wrong reasons.

## The game

The gamejam was about bullet hells and the revealed theme was `10 seconds`. So, naturally my first idea was to make a turn based card game. This is because I am just a sucker for card games and because I am _not_ good **AT ALL** with fast paced games, especially not if I need to dodge a lot.

So, something turn based was a must for me to enjoy it and cards make it just a bit more interesting if you ask me. This only meant I needed to think of how I would implement the theme as `10 seconds` isn't exactly a theme that lends itself well for turn based games. Luckily, there where plenty of ideas after a quick brainstorm session with other mods from the [gdb server](https://discord.gg/fvjMQNEkQU) there where plenty of ideas.

The two I wanted to implement where:

- Every turn takes 1 second, every 10 turns an event happens
- You have a timer that starts with 10 seconds and ticks down while it is your turn, this timer does not reset at the end of your turn

Sadly, due to time constraints I only managed to work on the second idea though I did manage to get the `Every turn takes 1 second` part of the first idea to work pretty well.

You can play the game as it was released to the jam on itch. There is also a preview of the next build embedded at the end of this page

Please read the description as I didn't have time to make a tutorial:

{{< rawhtml >}}
<div style="text-align: center;">
<iframe src="https://itch.io/embed/1006451?linkback=true&amp;border_width=0&amp;bg_color=8a8a8a&amp;fg_color=ffffff" width="550" height="165" frameborder="0"><a href="https://lenscas.itch.io/card-hell">card hell by lenscas</a></iframe>
</div>
{{< /rawhtml >}}

## The problems

There where 2 main problems I ran into, with one really draining my motivation to work on it and the other being a result of me not knowing Godot well and doing something stupid.

### Out Of Memory issues

For various reasons, it is currently nicer for me to work on games on my laptop rather than my desktop. However, this laptop only has 8GB of RAM, with 1.3 GB or so taken by the GPU. This, doesn't leave me much RAM to work with but generally spoken it is good enough. Sadly, Ionide, the program I use for autocompletion and the like eats A LOT of RAM. To the point that just VSCode open would often lead to crashes because my laptop ran out of memory. Constantly watching the memory usage of your laptop isn't exactly fun as you can imagine and thus I didn't put in as much time as I would've liked. In fact, there have been about 2 days where I almost did nothing because I just didn't feel like dealing with it.

### Assets not loading

For my main game I don't really have to care about getting assets from disk as I get everything from the server. This is not the case for `card_hell` and thus I needed to find out how to properly load assets from the disk and use them for sprites. My first instinct was to do it similarly to how I did things for my main game.

1. Make a new sprite and texture object
2. Load the sprite from disk using Texture.Load
3. Attach the texture to the sprite
4. ???
5. PROFIT!

And... this method worked if I pressed play in the editor. The moment I exported or tried to play in the browser it would fail to find the assets. Even worse, I didn't discover this error until pretty late into the jam. NOT GOOD!

It took me a while to find the solution, at first assuming that Godot doesn't include the assets because my F# code isn't really seen by Godot. This turned out to not be the case and all I needed to do was to just use `load` instead of the mess I used before.

## What went well

If you read my other posts you may remember that Signal and F# specific (kind of) types don't mix well. I also really wanted to use a `discriminated union` for the cards as that felt the cleanest. However this would mean I couldn't send it over signals. Luckily, I managed to avoid the need for custom signals all together, instead making a lot of use of `callbacks` to drive the state forward. I actually like this structure A LOT more than if I would've used signals, as it becomes a lot more clear what the order of events will be and F# being an FP language makes doing this easy enough as well.

This is also the first time I managed to get to do some art for a gamejam. The art isn't much but it does the job well enough. So, everything considered I am more than happy with them.

Most of the other code also went rather smoothly and despite doing some very stupid things the code doesn't seem to have too many bugs. The 2 big ones I am aware of are bullets moving over the cards and enemies shooting about 488 bullets at once. Neither caused any complaints so, I won't complain about them.

## Feedback

The feedback I got was not surprising. The timer was BRUTAL making the game harder than it has any right to be. There simply weren't enough batteries spawning quickly/close by enough to survive. The deck is also not the greatest which isn't helping either.

People also didn't notice that the enemies where using a deck of their own. Not that surprising as besides bullets there isn't anything that interacts with the decks.

The feedback on the idea however was positive, so I decided to work on it a bit more. At least, for the time being and I already made some changes that should help.

- I changed the logic that spawns batteries to always spawn one if there are less than 3.
- I increased the spawn rates of batteries in general
- I made the timer more forgiving. Instead of ending the game, it now acts as if you got hit by a bullet and then resets.
- I added 2 more game modes.
- - Resetting -> Here the timer does reset at the end of the turn, but the number it resets to becomes smaller over time. Use batteries to get it up again.
- - Relaxed -> The timer is replaced by a turn count.

I also made some improvements to the UI (or lack there of) and added a basic score/high score system as well as fixing the 2 bugs mentioned earlier as well as some general code improvements. I also tried to setup a proper CI but that hasn't been successful yet. I'll probably give it another shot sooner rather than later though.

The new version isn't ready for release yet BUT that doesn't stop me from putting up a nice preview of it here. Just, make sure to read the description if you haven't played it yet (Still no tutorial) :)

## description
Every turn you need to chose 2 cards, the first you click on will be cast while the second one will be discarded. Then, the turn plays out and everything follows the action they chose. Note: Bullets only interact with you if they land on the same square as you. You CAN pass through them if you do NOT land on the same tile.

Now, your hand refills back to 4 cards and you, as well as any enemies are free to make the next turn.

When something gets hit by a bullet they get a new card (big red cross) that will kill them when they cast it. If you cast it, it is game over. This means that if you cast it, you are dead. If an enemy casts it, they are dead.

{{< rawhtml >}}
<div style="text-align: center;">
<iframe src="/card_hell/index.html"  width="1024" height="600" frameborder="0"><a href="/card_hell/index.html">card hell by lenscas</a></iframe>
</div>
{{< /rawhtml >}}