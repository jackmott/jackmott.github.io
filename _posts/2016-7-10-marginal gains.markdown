---
layout: post
title:  "Marginal Gains"
date:   2016-07-01 14:17:27 -0500
categories: programming tools editor ide visual studio
---
In a former life I was heavily involved in bike racing, and became obsessed with the concept of "marginal gains". It is the idea that there exist a multitude of choices you can make, 
each of which, in isolation, has little to no effect on your result, but in totality can be the difference between success and failure.  It is a philosophy which requires a delicate balance.
Too much time spent worrying about minutiae can distract from the business of actually training. But ignoring marginal gains completely means you will eventually lose to someone
who did not.

So it is with being a software developer. Taking time to master your tools will save you time, and expand your abilities. But, at some point, you just need to shut up and code.

## Become one with the command line

If you spend most of your time in Windows software development it is possible to get by and never really master various command line systems and tools.  Taking the time to force yourself to
learn these things can be extremely valuable and open up entirely new worlds.  Being familiar with how to get things building from source in Linux for example, can allow you to
leverage and contribute to open source projects that might otherwise be unavailable to you.  If your projects are deployed to cloud infrastructures such as Azure or AWS, being 
able to manage all of that from Bash or Powershell let's you get things done much, much faster than working through web GUI interfaces.  You will likely be able to easily automate a lot
of your daily tasks with scripts.  For instance do you type "git add *, git commit -m "foo", git push" 30 times a day? (or use your mouse and click through 3 menus in the GUI equivalent?) 
That is an easy fix for even a beginner at bash or batch file scripting.
Being familiar with the command line also opens up options such as using faster or more flexible code editors, rather than being stuck in Visual Studio, Eclipse, etc because you
don't understand how to build and run things without the IDE to help you. The possibilities here are too many to list.  Take stock of the kinds of things you work on, and take time out 
of your day to learn Bash, or Powershell, or whatever command line skills may be relevant to your job and interests.  

## Master your code editors

Whatever you use to edit code, whether it be an IDE like Visual Studio or a text editor like VIM or Sublime, take some time to truly master it.  Think carefully about what wastes your
time as you use it.  Do you spend lots of time navigating around code with the arrow keys?  Learn what shortcuts are available to speed that up.  Move to next token, move to next / previous
matching brace, these are often features available with a hotkey in a good editor.  If you want a feature that isn't there, look into customizing the editor to add it. Occasionally take a day 
and make it a goal to do all your coding without touching the mouse.  At first you will find hundreds of things you can't accomplish without doing so, but gradually you will learn how to
bring that down to zero, either by learning keyboard shortcuts, or tweaking your environment so you don't need them. Here are some common examples:

### Changing indentation on blocks
![tab-1](/images/tab-1.gif "Tab Slow") ![tab-2](/images/tab-2.gif "Tab Fast")

With Visual Studio you can do this by selecting and using TAB and SHIFT-TAB. You can even select rectangular regions with ALT-SHIFT-ARROWS.
 
### Token Delete and Multi Line Editing
![token-1](/images/token-1.gif "Token Slow") ![token-2](/images/token-2.gif "Token Faster") ![token-3](/images/token-3.gif "Token Fastest")

In Visual Studio you can delete entire tokens at a time with CNTRL-DEL or CNTRL-BACKSPACE. You can also navigate the cursor a token at a time with CNTRL-ARROW. Some may find it useful to also
have a camelcase/pascalcase aware feature.  The third gif here shows Multi Line editing in action, where you can use the ALT-SHIFT selection feature and make identical changes to 
many lines at once. Most code editors will have a feature like this, it can be very useful to learn it.

There are dozens of little tracks like this available in any decent code editor. Any time you find yourself having to bang a lot of keys or use the mouse to get things done, investigate whether
your editor has a shortcut built in for the task, or whether you can easily add one. These will take practice to use quickly and without thinking about it, but when you build up a nice set 
of shortcuts, such that you are rarely touching the mouse or repeating keystrokes, you will get things done faster, and more pleasantly.
 
A few more freebies in Visual Studio:

* F9 set/unset a breakpoint on the current line
* F12 go to definition of the token the cursor is currently on
* F5 to run  SHIFT-F5 to stop the current default project
* CNTRL-TAB to pop to the previously focused window
* Hold CNTRL and press TAB to cycle through all previously open windows
* CNTRL-K CNTRL-O in C++ files to hop between .h/.hpp and .cpp/.c files
* CNTRL-ALT-L to pop to the solution explorer, ARROWS and ENTER to navigate/open


## Expand either the depth, or breadth, of your skills

Take some time to Git Gud.  Maybe you are committed to being deeply expert in one aspect of programming. If so, take time to deepen your understanding.  Think about what language features or 
programming concepts have confused you in the past.  Set time aside to master those things. 

Alternatively, especially if you are younger, take time to learn and practice entirely new philosophies.  Has your career to date been entirely in managed runtimes? Start a personal project in C,
D, or Rust, and learn what programming without garbage collection, and with access to the bare metal is like.  Have you done nothing but Object Oriented Programming your whole life? 
Try some side projects in a functional language, and try it again in ANSI C.  Learn for yourself what the pros and cons of these paradigms are for you.  Do not believe the assertions you
hear every day about different programming paradigms, almost none of them are backed up with rigorous evidence.  Take time to investigate 
the universes you are not familiar with. Consider contributing to an open source project, where you can learn from a new set of people than you deal with at your day job.  You will likely
pick up useful tips and techniques from that that you can use elsewhere. Even if you end up sticking with what you already know, you will likely learn some things to make you better at that too.




