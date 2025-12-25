# Welcome to Ooloi

Greetings, and welcome to Ooloi, the spiritual successor to Igor Engraver. If you're seeking yet another conventional music notation software, I'm afraid you've taken a wrong turn. Ooloi aims to be something rather different – and there's a story behind why that matters.

## A Quarter-Century in the Making

Twenty-five years ago, I created Igor Engraver, which became rather successful in the music notation world. When that project ended, it left something unfinished – not just the software, but the understanding of what music notation software could truly become. Ooloi represents the completion of that circle, built with decades of accumulated insight about both music and programming.

In the intervening years, I became an AWS Solutions Architect Professional and created systems like Ocean and OpenSecOps. I have always thought in systems – this shift simply allowed me to give that instinct full rein, to focus entirely on designing foundations that can handle complexity and scale over time through elegant abstraction layers.

Ooloi distills everything I've learned into an architecture that doesn't just work, but works elegantly. This isn't my first attempt at solving these problems, but it's the first time I've had the right tools – functional programming, immutable data structures, enterprise-scale systems thinking, and the kind of patience that comes with experience – to solve them properly.

## What is Ooloi?

Ooloi is open-source music notation software, designed from the ground up to handle complex musical scores with both finesse and power. Built in Clojure, it represents a fundamental rethinking of how music software should work.

What makes it different:
  * **Temporal coordination**: A traversal system that understands musical time
  * **Collaborative by design**: Multiple musicians can edit the same score simultaneously without conflicts
  * **Memory efficient**: Handles massive orchestral scores without breaking a sweat
  * **Extensible architecture**: A lean core augmented by plugins in any JVM language
  * **Professional output**: High-quality rendering and printing that works identically across platforms
  * **Cross-platform**: Mac OS X, Windows, Linux

## What Ooloi Attempts

Music notation software has long struggled with certain problems: temporal synchronization across complex scores, collaborative editing without conflicts, memory efficiency with large orchestral works, and extensibility that doesn't compromise the core.

These aren't failures of vision or effort – they're consequences of architectural choices made when different technologies were available. Ooloi bets everything on one presupposition: that functional programming techniques, now mature enough for production use, can address these pain points in ways that weren't practical before.

This isn't disruption or competition for market share. It's an attempt to create an open platform using different foundations, then see whether that approach proves useful. First-class MusicXML import means you can try Ooloi with your existing scores without migration risk. Sit, straddle, hang? Come and have a look.

## The Architecture You'll Inherit

What you'll find here is the result of taking time to get the abstractions right. The engine is conceptually complete, with comprehensive testing proving it works correctly. The temporal coordination system, the pure tree data structures, the STM-based concurrency – these represent solutions to genuinely hard problems.

But here's the thing: good architecture should be invisible to those who use it. The complexity is handled for you, hidden behind interfaces that make difficult things simple. You can focus on the problems you want to solve – whether that's creating plugins, improving the user interface, or adding new musical capabilities.

## On Collaboration and Realism

The engine's musical and architectural foundations are complete. Work at the intersection of music notation, functional programming, and distributed systems has solved the genuinely hard problems. Comprehensive testing proves correctness.

This foundational work happened alone because that particular intersection – professional music notation, functional programming, and architectural discipline – is rare enough to make collaboration impractical. That's reality, not complaint.

But Ooloi is more than music theory in Clojure. There's server infrastructure, asynchronous processing, streaming, printing, build pipelines across platforms, deployment, performance optimization. These are distributed systems problems, not music problems. If you're experienced with Clojure and interested in these domains, there's work here that doesn't require understanding figured bass or tuplet scaling.

Most importantly, the plugin architecture exists precisely because different contributors have different strengths and work in different languages. That's the primary contribution path, and intentionally so.

## How You Can Contribute

Ooloi's architecture recognizes that different contributors bring different expertise. Here's what's realistic:

**1. Plugin Development**: This is the primary and most practical contribution path. The plugin system supports any JVM language – Java, Kotlin, Scala, Groovy, whatever you're already using. Bring your own tools, libraries, and build system. If you're yearning for your endofunctors in a religiously strict environment of the Opus Dei type, you can have them. Commercial plugins are explicitly permitted under MPL 2.0. You don't need to understand Clojure or music notation internals; you just need to work with the published interfaces.

**2. Infrastructure and Platform Work**: Build pipelines need testing on Windows and Linux. Deployment scenarios need validation. Performance characteristics need measurement across platforms. If you're comfortable with Clojure but uninterested in music theory, this matters.

**3. Core Development**: The musical foundations are complete, but there's ongoing work in server infrastructure, asynchronous processing, streaming, printing, and cross-platform concerns. If you're experienced with Clojure and interested in distributed systems or real-time processing, contributions are realistic here. If you're learning Clojure or unfamiliar with these domains, the plugin path is more practical.

**4. Testing and Quality Assurance**: Help ensure reliability across platforms, configurations, and use cases. Testing matters as much as features.

**5. Documentation**: Make the architecture understandable. Explain concepts clearly without dumbing them down. Documentation is contribution.

**6. User Interface and Experience**: The frontend needs refinement. JavaFX expertise is valuable here.

## Getting Started

1. Read our [Code of Conduct](CODE_OF_CONDUCT.md) and [Contribution Guidelines](CONTRIBUTING.md). They set the tone for how we work together.

2. **For core work**: Start with `shared/` – that's where the engine lives. Both `backend/` and `frontend/` are relatively thin wrappers around it. The musical foundations are complete; ongoing work centers on infrastructure, server concerns, and cross-platform deployment. **For plugin work**: Review the plugin architecture documentation and choose your preferred JVM language.

3. Explore the documentation, especially the ADRs (Architecture Decision Records) that explain architectural decisions and their rationale.

4. If working with Clojure, use the REPL extensively. Interactive development is powerful for understanding system behaviour.

5. Review open issues or create new ones. The architecture is solid, but there's always more to build, test, and refine.

## What You're Building On

This is architecturally sound software built for durability. The core is complete, tested, and documented. The plugin system provides extensibility that lets you solve your problems with your tools.

Whether Ooloi becomes widely used depends on whether it's useful. The architecture doesn't require adoption to validate its approach – the problems are solved correctly regardless. But if it proves useful, the foundations support whatever gets built on them.

The work continues. What gets built next depends on who finds it worth building.

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

*"Nun denn, allein."*

/ Peter Bengtson