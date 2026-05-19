# A New Language for a New Operating System
The general consensus is that Rust will be the language in which we write the operating system of tomorrow.

Rust is a massive improvement over C and C++. It gives us ownership, lifetimes, and a real language-level response to a lot of the problems that made systems programming a disaster for the last several decades.

The issue is that an operating system immediately drags us into the exact places where the language starts making exceptions.

Boot code. Page tables. Device registers. Interrupts. Syscalls. Inline assembly. Foreign ABIs. CPU instructions that are only meaningful if you already understand the machine contract. The closer we get to the metal, the more we start leaning on escape hatches, project discipline, and comments that amount to "trust me."

To solve these, we simply disable all of the features that make Rust the go-to choice in the first place! Low level code? Just mark it unsafe! Need it to run fast? You can simply turn off the borrow checker, or "defer it to runtime" aka just crash instead.

This is not good enough for a new OS. And as the saying goes, "That which you feel the world is withholding from you, you are withholding from the world".

So I have taken it upon myself to solve these problems by creating my own programming language called Omega, which I will use to build an operating system called Cathedral. This is an intentional name. "Cathedral style" programming is often mocked as the opposite of the bazaar. Too planned. Too centralized. Too architectural. Too obsessed with getting the shape right before users shows up.

But the OS is exactly where you would want this sort of care. And there is something poetic about the cathedrals of today being entirely digital, when building things in the real world is effectively outlawed.

## I'm not in Danger. I am the Danger.
The word `unsafe` tells us there is danger, but not what kind. It does not tell us what was assumed, what was proven, what was checked, or what was simply believed. It creates a fenced-off region where the compiler's normal promises become conditional on human discipline.

This was great for the aims of Rust, to provide an alternative to C, in which the entire program would otherwise be "unsafe". Obviously we can do better than this.

For Cathedral, I do not want "safe code over here, unsafe code over there." I want rings of trust.

### Ring 0: Axioms
At the bottom, there are hardware facts. Some of these are not proven by our program. They are axiom-like statements about the machine. This instruction does what the architecture manual says. This register means what the CPU says it means. This memory ordering rule is real. This interrupt behavior exists.

For now, call these axioms. Maybe the name changes later. The important part is that they are explicitly marked as the starting points of trust.

Above that, the kernel uses those roots to build stronger claims. It can say, "given these hardware axioms, this memory mapping operation preserves these invariants." It can say, "this syscall transition cannot grant a capability the caller does not possess." It can say, "this scheduler state remains valid across this interrupt path."

### Ring 1: Host Calls
Host calls are system calls, possibly through a shared library intermediary. General shared libaries are effectively banned, as they are as dangerous as ring 0.

If the host is Cathedral itself, we can trust harder because the OS is written in the same language and carries the same proof structure. The contract is not merely a wish written over an opaque API. The implementation can participate in the same proof story.

If the host is Windows, macOS, or Linux, the story is different. We can still write a beautiful contract around the API call. We can say what the input invariants are. We can say what the output should satisfy. We can describe preconditions and postconditions. But at the end of the day, we are trusting that OS.

A Windows API call with a contract is not the same thing as a Cathedral syscall with a proof trail. They may expose the same friendly surface to application code, but the trust underneath is different and should be reported differently.

In essence, the more that the Omega language owns, the better the environment.

### Ring 2: I/O
I/O is the next ring because it is the first layer normal application code should usually touch.

This is not raw host access. A program writing a file should not need to import the Windows API, the Linux syscall table, or some Cathedral kernel primitive directly. It should import the standard I/O surface, and that surface should carry a smaller, cleaner contract.

Ring 2 is where authority becomes application-shaped.

Read this file. Open this socket. Write to this console. Ask for randomness. Read the clock. Show a window. These are not hardware axioms, and they are not raw host calls, but they are absolutely still authority. They touch the outside world. They make programs non-deterministic. They can leak data. They can observe user state.

So I/O belongs in its own ring.

This gives the package manager a useful distinction. A package can be forbidden from using Ring 0 axioms and Ring 1 host calls, while still being allowed to use Ring 2 I/O through approved standard surfaces. Or it can be forbidden from using I/O entirely. A pure parser, math library, serializer, or deterministic simulation should not need any of it.

The key is that I/O should be explicit without forcing every application to become a kernel-adjacent program. Most code should not touch the dangerous roots. It should request ordinary capabilities from Ring 2, and the import graph should make that request visible.

### Ring 3: Library Trust
A serious build should be able to answer a basic question, "What did this program trust?"

And I do not mean in some stochastic way, with a series of greps, LLM analysis, or a security audit. The build itself should know.

It should know if the program used hardware axioms directly. It should know if it imported a package that used them. It should know if it touched host access. It should know if it used standard I/O. It should know if some dependency smuggled in filesystem access three layers down because a logging package got ambitious.

This is where the language and package manager start to become the same security system.

A package should be able to say:
- No hardware axioms
- No host calls
- No filesystem
- No network
- No clock
- No randomness
- No I/O at all

In fact, extreme blacklisting should be the hard default, and painful measures should be imposed on the user to override these. The package manager should be able to enforce this with 100% certainty.

If an application wants to post itself for download in a restricted store, maybe it cannot declare its own I/O. Maybe it can only use the platform-approved standard surfaces. Maybe some package category bans host calls entirely. Maybe a deterministic library must prove it has no clock, no randomness, and no filesystem.

This alone would cut out almost every single supply-chain attack seen in recent times. The current security ecosystem of reputations and trusted packages is too fragile when compromising repositories simply means compromising a single trusted maintainer.

At each level, trust should be noted. At each import, authority should be visible. If nobody is allowed to hide authority, nobody can slip in "just a little unsafe thing" without the graph changing.

## Proofs, Axioms, Invariants
The language should support real Lean-like proofs. If a package wants to ship machine-checkable proof that its invariants hold, that should be a first-class artifact. If a kernel subsystem claims a transition preserves a capability invariant, that claim should be challengeable by tools.

Of course I do not expect every programmer must become a mathematician, but being able to prove things about cryptographic functions, sorting algorithms, etc.

For the programmer, this will most likely surface in the form of invariant checking. Prove that an array access is valid. Prove that a loop terminates.

For the operating system author, that is to say me, the proof system will exhaustively cover kernel boundaries, host contracts, unsafe-looking operations, indexing, arithmetic, state transitions, capability transfer, memory ownership, and package authority.

## Inline Assembly
Even inline assembly is possible in user code in a trusted manner! This may seem shocking to some, as we are accustomed to inline assembly = unsafe.

But quite simply, each line of assembly can emit a precondition or contract. To use a SIMD operation, you have to prove that you own the data being loaded. You have to prove that it is SIMD-aligned. You have to prove that the size of the data fits in a SIMD register.

If you can prove all of this, then absolutely you should be able to write the assembly.

Naturally though, this means quite some work on my part. This means every single instruction of assembly needs to consume and emit contract information. This likely means slowly adding supported instructions. It also becomes a potential threat vector.

Quite possibly, this could end up being something banned by applications importing libraries until it is deemed sufficiently safe. But the important thing is that it is possible.
