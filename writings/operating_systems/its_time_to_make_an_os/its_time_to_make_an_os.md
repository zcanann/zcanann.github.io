# It's Time to Make a New Operating System
"The next Bill Gates will not start an operating system. The next Larry Page won't start a search engine. The next Mark Zuckerberg won't start a social network company. If you are copying these people, you are not learning from them." - Peter Thiel, 2012

There is great wisdom in this advice. LARPing as Steve Jobs will not make you Steve Jobs. In some twist of irony, I think people are increasingly aware of this, and as a result these areas are actually under-explored. Maybe our social networks kinda suck now. Maybe we need Steve Jobs to save Apple yet again. Maybe, just maybe, we need that new Operating System.

## You alread know this is a problem

I'm not going to spend much time convincing you modern OSes suck. You already know this. It is as obvious as the locked down aisles in CVS and Walgreens that are indicative of the societal decay around us.

Similarly, our OSes have undergone a form of decay.

Every year Windows and Mac get worse, and Linux has never emerged as a serious contender. Many Windows apps are now written in Javascript and WebView2. Opening your mail, start menu, settings, etc mean running a full browser renderer. To mask the emergent lag, they overclock your CPU briefly when opening these apps. Lunacy.

The point of highlighting these is to say that, these are beyond fixing. A company that is doing these cannot simply "turn things around". These decisions are so far from reality that nothing short of a rewrite is necessary. No amount of begging and pleading can restore competence where there is none.

When the greatest systems engineers of our generation, like John Carmack, or Jonathan Blow, are saying "maybe it's time to write an OS", maybe it's time to write an OS.

## What is the Wedge?
I think the desires of a new OS are generally well understood. Heavily sandboxed by default. Security around capabilites instead of around users.

The problem is that, these things are not exactly user-driving behavior. Their apps are on Windows and Mac.

In my mind, the ability to run a compatability layer for Windows/MacOs apps is a non-starter. It is imperative that most apps "just run" on this new OS, to give people time to build and target the OS natively. This was exactly the strategy used by apple for the switch from x64 to ARM64, and by Steam Proton to run Windows games. These strategies worked.

## Attoning for our Sins
Everything is not a file, contrary what Linux may think. And loading a user-mode DLL to abstract OS syscalls ended up not providing the backwards compatability that Microsoft and Linux thought it would.

// To be continued thx
