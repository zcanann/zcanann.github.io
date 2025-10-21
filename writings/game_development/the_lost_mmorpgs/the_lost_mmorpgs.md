# The Lost MMORPGS

## Too Much Ambition
The folly of solo game developers wishing to create an MMORPG by themselves is so commonplace that it has become enshrined as a meme. Time and time again, novice developers show up to online spaces such as the IndieDev and GameDev subreddits to reveal their grand plans. And time and time again, their plans are shot down by experienced developers that seem to know better.

There is a detail however that boggles the mind. Runescape, one of the most famous MMORPGs of all time, was developed by two people. Sure, after some success, they hired and scaled up. But RuneScape Classic was a full and meaningful MMORPG with monsters, quests, inventory, icons, 3d models, player vs player combat, an expansive world, and so on. Additionally, it was released in 2001, an era when Dial-up low bandwidith internet connections were still common in many homes.

Additionally, it was built using technology that would be considered jank by today's standards. The client was written in Java and embedded in a web browser plugin. Even the server was written in Java. While nobody would choose these technologies today, these had the advantage of being cross-platform, and the browser embedding allowed for faster growth by avoiding downloaded clients.

Hopefully the root of my confusion is clear. If technology is getting better year over year, where are all the MMORPGs? We can now write games in any language and compile them into WASM modules to run over WebGPU in the browser. We can spin up infinity AWS servers to auto scale our database infrastructure. We can command armies of robots to do tasks for us at our whim.

Shouldn't these technological advances mean that it would require much less ambition to create an MMORPG today? Then where the fuck are all the solo developed MMORPGs? Why does building software feel harder, not easier, as our tools improve?

## Tower to the Gods
Peter Thiel famously points out that the world of bits has progressed rapidly, but the world of atoms has come to a standstill. The bridges, roads, and buildings we use were primarily built 80 years ago. But computers seem to just keep getting better! The culprit, Thiel believes, is overregulation. This seems to track, as in order to build anything in the 21st century, it is an endless environmental review and zoning process that only those with the hardest of wills can overcome.

If this is the case, then my mind is boggled even further. There is no government to regulate how you build your MMORPG, other than common sense laws around e-commerce and content. In some sense, it appears that Thiel was wrong, and the entire Earth has come to a standstill. In the world of bits, we have nobody to blame but ourselves. We insist on swapping out our JavaScript frameworks in favor of some new shiny one, only to ditch it again next year, and the year after, making no sensible progress.

Jonathan Blow seems to see clearer on this, having famously said that we are now paying the cost for trying to build a tower to the Gods. There is a belief that we can endlessly abstract -- we build assembly to abstract the machine, and C to abstract the assembly, all the way up to our beloved JavaScript, only to get tangled in the mess of a dynamic type system. Do we dismantle the shaky foundations and build a better tower? No, we add yet another layer, throwing on TypeScript to patch over the flaws of JavaScript!

This lunacy has finally caught up with us. We can no longer jump into game development, but instead must spend a few years learning a programming language, then another year or so learning a game engine. After many years, we can finally begin. Of course, when something goes wrong, our tower of abstractions proves meaningless. When a bug in the inline assembly matrix multiply breaks, the C++ breaks, and so does the C# engine running upon them, and the Lua engine running upon that. Tracking it down becomes an exercise in scaling down the entire tower.

If our abstractions are not abstracting, then what good are they?

## Drowning in Complexity
Having worked at Roblox on the game engine team, and another company using Unreal Engine extensively, I have seen the worst of the worst.

For the last five years, my Mondays have been the same. Wake up, prepare for the day, and hop into a team meeting to dig through stack traces. Generally, these are esoteric memory crashes or threading issues that come from a deeply complex mess of C++. Even if the code is 'clean', the issues can be incredibly subtle and disasterous. The root cause is never obvious, but somebody has to fix them. I once witnessed a team member of mine get assigned a threading related crash on app teardown. Two months later, he reported that he identified 6 or 7 potential causes and fixed them, but there was no dent in the crash rate. He then replied, 'Well, the root cause might be from the 8th or 9th issue, which I am still tracking down'.

The human mind is genuinely not capable of grasping the interplay between several threads across several systems. Even design patterns like RAII that are supposed to be infalliable collapse once exposed to even the simplest of real world use cases. With thousands of developers working on the same codebase, architectural errors are bound to creep in. New features get put behind flags, which are queried from an API. These flags can be rolled out by developers at will, who monitor crash rates to see the impact on crash rates.

Additionally, there is the frequent black swan event, where a latent memory bug comes to surface by chance due to memory getting shuffled around by a new build. But these cannot even be called black swans, because these rare events are not rare at all. Additionally, any engine changes can cause anywhere from a 15 minute to a 3 hour rebuild. Best case, your focus is lost. Worst case, the day is lost.

The primary joy of programming is the act of creating. When 20% of your time is spent creating, 30% debugging, 10% building, 20% in meetings, and the rest monitoring dashboards, then consequently only 20% of the total time is enjoyable. Somehow, we have swapped out creating for futile bug hunts and morphing software development into a data science job.

We seriously seriously, do not have to live like this.

In an effort to escape this madness, many indie developers switch to Rust. Unfortunately, Rust only solves the memory safety problem, not the compile speed problem. Additionally, development costs are higher to appease the borrow checker, the price to pay for safe builds. Slow iteration speeds are enough to kill the momentum and fun on any indie project. As the glib saying goes, 'Rust can do anything except ship'. While this is objectively false, as there are many great Rust projects that have shipped, very few of them are games.

A great way to avoid pitfalls is to adopt a philosophy of working backwards. If you want to make a particular game, look at the technologies similar games have used and work backwards. First person shooter? Probably use Unreal Engine. 2d side-scroller? Unity or Godot seem preferable. Rust rarely appears, not just because its new, but because it is genuinely difficult to make a game in the language. Games can tolerate a certain level of buggyness, and even poor architecture.

The key distinction to make is in game vs game engine. Games need speed. Game engines need safety. This of course gets muddled as the two mix. For example, if you made an open source Unreal Engine clone in Rust, then anybody who uses it would be forced to adopt Rust as well. The safety would bleed into the game, reducing speed. Scripting languages help at first, but any game that uses them extensively finds themselves quickly drowning in complexity from lack of strong typing. Unreal Engine's Verse language is the first serious attempt I have seen to remedy this problem.

Additionally, those who go the route of Rust find themselves unknowingly making a pact with the devil. Take for example the Bevy Entity Component System. This is seen as an incredible marvel of Rust programming and modern game system design, and they boast to be a great foundation for future game engines. However, the code is riddled with `unsafe` keywords and `RefCell`, which are effectively methods of disabling the borrow checker and all of the safety Rust aims to grant. In other words, Bevy may as well be written in C. We are told that 'its okay, Bevy's code was audited, so trust us bro!'. We have 'been trusting bro' for our entire lives, and look where it got us.

It is quite difficult to trust any 3rd party library when they can turn off the safety at will, and many of them do. Instead, developers are left to enforce coding standards to ensure no developers step out of line, and have to be incredibly wary of pulling in crates. A stricter language could have solved all of these woes.

Note that when I refer to Rust from here on out, I really just mean a language with compile time guarantees of safety at runtime. Perhaps in the future, a stricter language will replace it, but the purpose remains to write safe code with zero-cost abstractions, meaning that the language compiles to native assembly language without any additional bloat, allowing it to run lightning fast on a CPU.

So naturally, the wiser indie devs seek to optimize for developer iteration speed, preferring Unity, Godot, or a custom engine. Still, even these are rife with hidden costs. Unity is less opinionated than Unreal Engine, leaving developers to create much of their own tooling. Also, since it is a just-in-time and garbage collected language, profiling and optimization become substantial time sinks. One would think that an engine from scratch surely must be feasible, hoping to ride some technological advancements, but unfortunately the advancements in graphics were all in capabilities, not in simplicity. Few are even attempting to address these problems. Notable examples are Jonathan Blow's new language Jai, which boasts compile times under a second for even large games, at the cost of some language features. This may be a great solution as the project unfolds.

This begs the question, can we not just cast these technologies into the sea, and return to Java embedded clients? If it worked 20 years ago, surely it still works today. If we are drowning in the complexity, then we should abandon the complexity altogether.

While it sounds nice, there is the problem that user expectations have also grown quite large. Releasing RuneScape Classic today would not be particularly interesting when we have games like World of Warcraft. In fact, a few solo developers have released MMORPGs of the same quality as Runescape Classic, but there is no demand for them.

However, it is foolish to claim that this is solely a demand-side problem. Remember that most developers start with the ambition of starting an MMORPG. Are we expected to believe that all of these people did a market analysis and decide it was not worth it? Are we expected to believe that only a single digit number of these developers over the last 20 years were able to cut scope enough to release?

The problem seems not that MMORPGs are too complex, but that all software is needlessly complex, and only large enough teams can survive it long enough to bring a product to market.

## Where from Here?
These questions plague me, because every monday morning I must take my weekly crash triage call, where I am reminded how truly bad these problems have become. Why are speed and safety at odds? We have demonstrated that speed is too fragile to get us to the Gods. We have demonstrated that safety is too slow to get us to the Gods. Even worse, the safety is often a lie, and not even safe at all.

In the age of large language models (LLMs), there may be hope for the safety track. A machine running a large language model has nothing but time. If directed to write code, perhaps this can speed up the development of safe code.

As of today, LLMs are quite terrible at architecting large projects and navigating multi-file codebases, so it remains to be seen how much they will improve on these axis. However, they show great promise on contained files with ample context. In fact, it may be possible to develop faster than an agent helping someone write C#, Java, or JavaScript. More issues are pushed into compile time, meaning faster feedback on errors. For the time being, AI will still litter your codebase with unhandled `.unwrap()` and `RefCell`, but a pre-prompt can correct these for now. In the future, it may be worth the investment to render these types of errors impossible.

It is also possible that our universally accepted programming model is flawed. It is very difficult to write proofs about turing machines, because of the combinatorial explosions. I am not advocating for some Lisp-like utopia where programs are boiled down to some math proof. However, we can write proofs about much simpler machines, like state machines. The more things we can prove at a low-level, the easier it will be to build upon them.

We have somehow found ourselves in a place where we gripe about the same problems of 20 years ago. Even in a mature engine like Unreal Engine, writing networked code with prediction is agony. How is it that after this long, we have not figured out reasonable abstractions for networking code to make predictions easy? Even forgetting complex networking, trying to build something as simple as a networked turn based card game is torture. It is a complex series of networked reliable calls that can drown the developer in complexity.

Again, we seriously do not have to live like this.

Dsigning a card game should be as simple as building a drag-and-drop networked state machine. All of the sudden the multi-player aspect is not so scary! The retries can be handled automatically. Special escape hatches for when things go terribly wrong, like player disconnects. Life becomes simple. However, I will not pretend to know how well this paradigm scales up to MMORPGs, which have substantially more complexity. Players log on and off, creatures spawn and die, and so on. The state is messier.

The answer may look like something else entirely, but the point is that we aren't even trying. Having spent so much of our efforts expanding the capabilities of technology, perhaps it is time to focus on the ergonomics of technology. Build times over a minute are unacceptable. Spending weeks on boiler plate is unacceptable. Fighting complex networking code that literally every other game has independently solves is insanity. Spending every monday digging through crash logs is unacceptable. These are real costs that are eating up massive swaths of developer time, and due to the tragedy of the commons, few seek to fix them beyond hacks.

Credit where credit is due, we have made some progress, WASM is a true marvel that finally allows us to ditch the founding sins of web development, but we can be doing so much more. When Windows serves us advertisements and bloatware, it becomes time to seriously consider making it your secondary OS.

I'll personally be directing my efforts at a Rust first suite of game engines, starting with simple Genres like point-and-click, visual novel, deck builder, working my way up to MMORPG. My work will be done when we start to see solo developers churning out MMORPGs that would have otherwise been lost to time.
