# Networked State Machines as First-Class Citizens
State machines are extensively used by the vast majority of games. It is near impossible to imagine implementing menu flows, or turn-based games without them.

Generally, these state machines come in the form of code. An enum to control the state, and logic to transition from one enum value to another. These are often simplified further by wrapping state transitions in macros, to denote which states can transfer to/from which other states. This is boilerplate that must be repeated again and again for each adhoc state machine in the codebase.

Programmers have repeated this pattern with heavy success time and time again. However, in doing so, we have made a grave mistake. We have settled for the 2nd best move. A better move exists.

To understand the problem, we need to understand the almighty Tick. Want to move to a new menu state? Simply check for input or an event in Tick, and update a global state variable. The state machine is a second-class citzen, subserviant to the almighty Tick.

The problem compounds even more when networking is involved, such as a two player versus card game. Now we have three Tick based state machines, two for each player, and one for the server. Again, these exist entirely in code, in which state advances based on the success or failure of network events. Generally, this is behind reliable, TCP-based game server remote procedure calls (RPCs) that require the programmer to have some semblence of this state machine in their head.

Debugging becomes tedious. Within Tick, we can possibly render to some debug screen, often written in an immediate mode renderer such as imgui, to determine which state we are currently in. As before, this must be implemented for each individual state machine in the codebase, completely adhoc. We are missing a layer of abstraction that significatly speeds up development and iteration time of these state machines.

So what does a first-class state machine look like? Fortunately, there are a few great examples we can learn from. Unreal Engine extensively uses visual scripting for many of its components, but one in particular stands out. Animations are driven extensively by an animation graph, which is a visual state machine. The states light up for quick and easy visual debugging, and creating states/transitions is entirely authored via drag and drop.

Unfortunately, this system is entirely constrained to animations. Their coding visual scripting language called 'Blueprint' is not a state machine. It is largely tick or event driven, and incredibly difficult to follow execution flows. This is no different than the Tick + manual tracking we outlined earlier that is commonplace in coding state machines.

There is however a plugin called Logic-Driver Pro that extends the idea of blueprints further, encapsulating them in a parent state machine. Instead of creating an enum for which menu state we are in, we simply create a menu state machine. Each new screen is just a new state. There is no enum. Its just a graph. At any given time, it is obvious what menu state we are in. When transitioning to each state, it becomes trivial to spawn the desired menu widgets.

Transitions can be unconditional, they can be input driven, or they can be event driven. Within these states, there is nothing preventing an internal state Tick. Tick has now become a second-class citizen to the state machine, making the entire system far easier to reason about.

That said, state machines can be pushed even further. What if you were designing a card game? In this scenario, we would want a state machine that travels across client and server boundaries. Again, if we were writing code, we would be manually tracking the state and advancing it via RPCs. If we built out a replicated, networked state machine, then state transitions can be marked as remote procedure calls. It becomes obvious when we are passing the privileged boundaries, it becomes clear when we are doing too much extra networking, and so on.

From what I see, this has never been implemented in a generic form in a public game engine. I'll likely be one of the first to try it, and should have more to say once done 🫡
