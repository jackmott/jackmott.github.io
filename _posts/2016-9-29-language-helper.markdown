---
layout: post
title:  "Language Helper"
subtitle: "A useful gamedev library?"
date:   2016-9-29 14:17:27 -0500
categories: programming
---

One of my many half finished side projects is a [text adventure engine](http://jackmott.github.io/dungeonbuilder/), where players can play Zork-like games,
or make their own game from within the engine as well.  One of the tricky problems I ran into was being able to handle certain english grammer issues
in the engine, in a way that was not extremely annoying for users.  For instance, you might want to indicate in the engine editor that a room has 2 gold coins and a dagger
in it. When someone plays the game you could do the usual gamedev thing and present the room something like this:

```
You enter a scary dungeon.  
You see:
  2 gold coins
  1 dagger

> Take 1 coin
You take 1 coin.
You see:
  1 gold coin
  1 dagger
```

It is easy to write code to do that, but it breaks the fourth wall, and no longer reads like a story.  What if instead you wanted it to look like so:

```
You enter a scary dungeon.  There are two gold coins and a dagger here.
> Take 1 coin
You take a coin, and now a gold coin and a dagger remain
```

This is harder code to write, especially when you support a editor where users might enter any words they want as items.  You have to identify if the indefinite article
for a given word should be "a" or "an" for starters, which has no simple rules you can follow.  You need to know the plural forms of words, 
which is not as simple as just adding an s. 

## The LanguageHelper library

I've put together a library that helps make some of these things easier, and might be useful for various text-based RPG applications. Maybe even useful for non text based ones
now that text to speech is starting to sound convincing.

What I have so far:

- Query any word for it's plural form
- Query any word for the correct indefinite article
- Query any verb for past tense, progressive tense, and past perfect tense
- Turn an integer into the words that represent the integer
- A json dictionary format so you can easily add your own words, or cull the dictionary for performance/size reasons

Things I would like to add:

- Query a word for synonyms and antonyms
- More complete coverage of verb conjugation 
- Pronunciation data?
- More languages?

The library is currently in .NET 2.0, so it can be consumed by any C# or F# code, including [Unity3D](https://unity3d.com/), but if people think this sounds useful 
I would create a C++ version as well.

## Useful?

Does this sound useful? Does this already exist? What other features would you need? Let me know!

## Sample Use Case

Following is a simple use case example.  Notice there are some subtleties that the library handles well. "steel ingot" is a two word item.  The library
correct pulls out the indefinitely article for the leading word "steel" but applies the plural form only to the trailing word.

``` ocaml

    (* Some imaginary blacksmith entity structure *)
    let item = "steel ingot"
    let qty = 2
    let currentAction = "forge"
    let pastAction  = "sleep"

    (* Player asks what the Smith has for sale *) 
    let response = "I have " + wordBank.QueryNounQty(item,qty)
    printf "%A\n" response
    (* Output: I have two steel ingots *)

    (* Smith only has 1 *)
    let qty = 1
    let response = "I have " + wordBank.QueryNounQty(item,qty)
    printf "%A\n" response
    (* Output: I have a steel ingot *)

    (* Smith has none left *)
    let qty = 0                
    let response = "I have " + wordBank.QueryNounQty(item,qty)
    printf "%A\n" response
    (* Output: I have no steel ingots *)

    (* What are you doing? *)
    let response = "I am " + wordBank.QueryVerbPresent(currentAction)
    printf "%A\n" response
    (* Output: I am forging *)

    (* What did you do earlier? *)
    let response = "I  " + wordBank.QueryVerbPast(pastAction)
    printf "%A\n" response
    (* Output: I slept *)
```

## Some sample queries

![DemoGif](/images/demo.gif "Demo Gif")



