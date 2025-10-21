# The Lost MMORPGS

## Too Much Ambition
The folly of solo game developers wishing to create an MMORPG by themselves is so commonplace that it has become enshrined as a meme. Time and time again, novice developers show up to online spaces such as the IndieDev and GameDev subreddits to reveal their grand plans. And time and time again, their plans are shot down by experienced developers that seem to know better.

There is a detail however that boggles the mind. Runescape, one of the most famous MMORPGs of all time, was developed by two people. Sure, after some success, they hired and scaled up. But RuneScape Classic was a full and meaningful MMORPG with monsters, quests, inventory, icons, 3d models, player vs player combat, an expansive world, and so on. Additionally, it was released in 2001, an era when Dial-up low bandwidith internet connections were still common in many homes.

Additionally, it was built using technology that would be considered jank by today's standards. The client was written in Java and embedded in a web browser plugin. Even the server was written in Java. While nobody would choose these technologies today, these had the advantage of being cross-platform, and the browser embedding allowed for faster growth by avoiding downloaded clients.

Hopefully the root of my confusion is clear. If technology is getting better year over year, where are all the MMORPGs? We can now write games in any language and compile them into WASM modules to run over WebGPU in the browser. We can spin up infinity AWS servers to auto scale our database infrastructure. We can command armies of robots to do asynchronous build tasks.

Shouldn't these technological advances mean that it would require much less ambition to create an MMORPG today? Then where the fuck are all the solo developed MMORPGs?

## Who to Blame
Peter Thiel famously points out that the world of bits has progressed rapidly, but the world of atoms has come to a standstill. The bridges, roads, and buildings we use were primarily built 80 years ago. But computers seem to just keep getting beter! The culprit, Thiel believes, is overregulation. This seems to track, as in order to build anything in the 21st century, it is an endless environmental review and zoning process that only those with the hardest of wills can overcome.

If this is the case, then my mind is boggled even further. There is no government to regulate how you build your MMORPG, other than common sense laws around e-commerce and content. In some sense, it appears that Thiel was wrong, and the entire Earth has come to a standstill. We swap out our JavaScript frameworks in favor of some new shiny one, only to ditch it again next year, and the year after, making no sensible progress.

Jonathan Blow seems to see clearer on this, having famously said that we are now paying the cost for trying to build a tower to the Gods. There is a belief that we can endlessly abstract -- we build assembly to abstract the machine, and C to abstract the assembly, all the way up to our beloved JavaScript, only to get tangled in the mess of a dynamic type system. Do we dismantle the shaky foundations and build a better tower? No, we add yet another layer, throwing on TypeScript to patch over the flaws of JavaScript!

This lunacy has finally caught up with us. We can no longer jump into game development, but instead must spend a few years learning a programming language, then another year or so learning a game engine. After many years, we can finally begin. Of course, when something goes wrong, our tower of abstractions proves meaningless. When a bug in the inline assembly matrix multiply breaks, the C++ breaks, and so does the C# engine running upon them. Tracking it down becomes an exercise in scaling down the entire tower.

## What to Blame
Having worked at Roblox on the game engine team, and another company using Unreal Engine extensively, I have seen the worst of the worst. My mondays begin with a team meeting to dig through stack traces. Generally, these are esoteric memory crashes or threading issues that come from a deeply complex mess of C++. Additionally, any engine changes can cause anywhere from a 15 minute to a 3 hour rebuild. At that point, the day is lost.

Many indie developers draw the wrong lesson from this, and seek to use Rust to solve their problems. Unfortunately, Rust only solves the memory safety problem, not the compile speed problem. Slow iteration speeds are enough to kill the momentum and fun on any indie project. The wiser indie devs seek to move as fast as possible, using Unity or a custom engine. Jonathan Blow's new language Jai also appears quite promising for those who do not mind low level programming.

Even those who go the route of Rust find themselves making a pact with the devil, or making a pact with someone else who made a pact with the devil. Take for example the Bevy Entity Component System. The code is riddled with `unsafe` keywords and `RefCell`, which are effectively methods of disabling the borrow checker and all of the safety Rust aims to grant. We are told that 'its okay, Bevy's code was audited, so trust us bro!'. I have 'been trusting bro' for my entire life, and burned frequently because of it.

Rust would be a spectacular language if it could live by its own aspirations. It is quite difficult to trust any 3rd party library when they can turn off the safety at will, and many of them do.

Of course, this also begs the question, can we not just cast these technologies into the sea, and return to Java embedded clients? If it worked 20 years ago, surely it still works today. If we are drowning in the complexity, then we should abandon the complexity.

While it sounds nice, there is the problem that user expectations have also grown quite large. Releasing RuneScape Classic today would not be particularly interesting when we have games like World of Warcraft. MMORPGs of today must be incredibly rich to stay competitive. And this seems to require embracing these technologies and all their complexity.

## Where from Here?
Still, every monday, on my morning crash triage calls, I cant help but wonder if we can be doing better... TBC
