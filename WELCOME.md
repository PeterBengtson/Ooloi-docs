# Welcome to Ooloi

Greetings, and welcome to Ooloi, the spiritual successor to Igor Engraver. If you're seeking yet another conventional music notation software, I'm afraid you've taken a wrong turn. Ooloi aims to be something rather different – and there's a story behind why that matters.

## A Quarter-Century in the Making

Thirty years ago, I created Igor Engraver, which became rather successful in the music notation world. When that project ended, it left something unfinished – not just the software, but the understanding of what music notation software could truly become. Ooloi represents the completion of that circle, built with decades of accumulated insight about both music and programming.

In the intervening years, I became an AWS Solutions Architect Professional and created systems like Ocean and OpenSecOps. I have always thought in systems – this shift simply allowed me to give that instinct full rein, to focus entirely on designing foundations that can handle complexity and scale over time through elegant abstraction layers.

Ooloi distills everything I've learned into an architecture that doesn't just work, but works elegantly. This isn't my first attempt at solving these problems, but it's the first time I've had the right tools – functional programming, immutable data structures, enterprise-scale systems thinking, and the kind of patience that comes with experience – to solve them properly.

## What is Ooloi?

Ooloi is open-source music notation software, designed from the ground up to handle complex musical scores with both finesse and power. Built in Clojure, it represents a fundamental rethinking of how music software should work.

What makes it different:
  * **Temporal coordination**: A traversal system that understands musical time
  * **Collaborative by design**: Multiple musicians can edit the same score simultaneously without conflicts
  * **Memory efficient**: Is designed to handle massive orchestral scores without breaking a sweat
  * **Extensible architecture**: A lean core augmented by plugins in any JVM language
  * **Professional output**: High-quality rendering and printing that works identically across platforms
  * **Cross-platform**: Mac OS X, Windows, Linux

## What Ooloi Attempts

Music notation software has long struggled with certain problems: temporal synchronization across complex scores, collaborative editing without conflicts, memory efficiency with large orchestral works, and extensibility that doesn't compromise the core.

These aren't failures of vision or effort – they're consequences of architectural choices made when different technologies were available. Ooloi bets everything on one presupposition: that functional programming techniques, now mature enough for production use, can address these pain points in ways that weren't practical before.

This isn't disruption or competition for market share. It's an attempt to create an open platform using different foundations, then see whether that approach proves useful. First-class MusicXML import means you can try Ooloi with your existing scores without migration risk.

## The Architecture You'll Inherit

What you'll find here is the result of taking time to get the abstractions right. The engine is conceptually complete, with comprehensive testing proving it works correctly. The temporal coordination system, the pure tree data structures, the STM-based concurrency – these represent solutions to genuinely hard problems.

But here's the thing: good architecture should be invisible to those who use it. The complexity is handled for you, hidden behind interfaces that make difficult things simple. You can focus on the problems you want to solve – whether that's creating plugins, improving the user interface, or adding new musical capabilities.

## On Collaboration: Not Yet

Ooloi is in pre-release development. The core architecture and its governing paradigms are complete but not yet public, and **collaboration is not open until after the first open-source release.**

This isn't gatekeeping for its own sake. The cognitive load of coordinating with others while completing the architecture would slow the work rather than advance it.

The architecture must be released as a whole before it can be extended by others. Piecemeal input during construction serves neither the system nor the contributor.

**If you're interested in contributing to Ooloi, the best thing you can do right now is wait.** Follow development if you like. Read the ADRs when they're published. But contribution paths open after release, not before.

Ooloi will become open source in the way TeX is open source: inspectable, forkable, extensible at defined boundaries. The core is not developed collaboratively.

## After Release: How You'll Be Able to Contribute

Once Ooloi reaches its first public release, the architecture will support contribution through several paths:

**1. Plugin Development**: This will be the primary and most practical contribution path. The plugin system supports any JVM language – Java, Kotlin, Scala, Groovy, whatever you're already using. Bring your own tools, libraries, and build system. If you're yearning for your endofunctors in a religiously strict environment of the Opus Dei type, you can have them. Commercial plugins are explicitly permitted under MPL 2.0. You won't need to understand Clojure or music notation internals; you'll just need to work with the published interfaces.

**2. Infrastructure and Platform Work**: Build pipelines will need testing on Windows and Linux. Deployment scenarios will need validation. Performance characteristics will need measurement across platforms. If you're comfortable with Clojure but uninterested in music theory, this will matter.

**3. Testing and Quality Assurance**: Ensuring reliability across platforms, configurations, and use cases. Testing matters as much as features.

**4. Documentation**: Making the architecture understandable. Explaining concepts clearly without dumbing them down. Documentation is contribution.

**5. User Interface and Experience**: The frontend will need refinement. JavaFX expertise will be valuable here.

**6. Core Development**: The musical foundations are complete. Post-release work will concern infrastructure, asynchronous processing, streaming, printing, and cross-platform concerns. This will not include redesigning musical semantics or architectural paradigms.

## What You're Building On

This is architecturally sound software built for durability. The core is complete, tested, and documented. The plugin system provides extensibility that lets you solve your problems with your tools.

Whether Ooloi becomes widely used depends on whether it's useful. The architecture doesn't require adoption to validate its approach – the problems are solved correctly regardless. But if it proves useful, the foundations support whatever gets built on them.

The work continues. What gets built after release depends on who finds it worth building.

## A Personal Note

At 64, carrying more than five decades of programming experience and a parallel career as a composer, I've tried to encode into this architecture not just technical solutions, but the aesthetic judgments and performance intuitions that come from actually making music. 

The creative energy that might have gone into another opera has found expression in software architecture. It's a different kind of composition – one that enables other people's creative work rather than expressing my own. And who knows, maybe Ooloi is what I need to finally write that modern *opera seria* set in China in the 700s. 

This is what happens when you take the time to get it right, when you resist the urge to rush, when you're willing to solve the hard problems properly. The result is something that can grow and evolve through the contributions of others while maintaining its essential character.

## Credo

Ooloi is built without compromise.<br>
It is not designed to please a community, or to flatter opinion.<br>
It stands on architecture, tests, and documentation.<br>

If you see the value, you will find it here, intact and intelligible.<br>
If you don't, nothing is lost.

/ Peter Bengtson