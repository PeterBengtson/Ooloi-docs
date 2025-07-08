# Welcome to Ooloi

Greetings, and welcome to Ooloi, the spiritual successor to Igor Engraver. If you're seeking yet another conventional music notation software, I'm afraid you've taken a wrong turn. Ooloi aims to be something rather different — and there's a story behind why that matters.

## A Quarter-Century in the Making

Twenty-five years ago, I created Igor Engraver, which became rather successful in the music notation world. When that project ended, it left something unfinished — not just the software, but the understanding of what music notation software could truly become. Ooloi represents the completion of that circle, built with decades of accumulated insight about both music and programming.

In the intervening years, I became an AWS Solutions Architect Professional and created systems like Ocean and OpenSecOps. I have always thought in systems — this shift simply allowed me to give that instinct full rein, to focus entirely on designing foundations that can handle complexity and scale over time through elegant abstraction layers.

I've spent the better part of a year on Ooloi distilling everything I've learned into an architecture that doesn't just work, but works elegantly. This isn't my first attempt at solving these problems, but it's the first time I've had the right tools — functional programming, immutable data structures, enterprise-scale systems thinking, and the kind of patience that comes with experience — to solve them properly.

## What is Ooloi?

Ooloi is open-source music notation software, designed from the ground up to handle complex musical scores with both finesse and power. Built in Clojure, it represents a fundamental rethinking of how music software should work.

What makes it different:
  * **Temporal coordination**: A traversal system that understands musical time
  * **Collaborative by design**: Multiple musicians can edit the same score simultaneously without conflicts
  * **Memory efficient**: Handles massive orchestral scores without breaking a sweat
  * **Extensible architecture**: A lean core augmented by plugins in any JVM language
  * **Professional output**: High-quality rendering and printing that works identically across platforms
  * **Cross-platform**: Mac OS X, Windows, Linux

## Why Ooloi Matters

The world of music notation software has been rather stagnant for too long, content with incremental updates and feature bloat. Most existing software suffers from fundamental architectural problems that can't be fixed with patches — they require starting over with better foundations.

Ooloi solves problems that have plagued music software for decades: proper temporal synchronization, efficient collaborative editing, memory-efficient handling of large scores, and clean extensibility. These aren't just nice features — they're qualitatively different capabilities enabled by choosing the right abstractions.

## The Architecture You'll Inherit

What you'll find here is the result of taking time to get the abstractions right. The backend is conceptually complete, with over 15,000 tests proving it works correctly. The temporal coordination system, the pure tree data structures, the STM-based concurrency — these represent solutions to genuinely hard problems.

But here's the thing: good architecture should be invisible to those who use it. The complexity is handled for you, hidden behind interfaces that make difficult things simple. You can focus on the problems you want to solve — whether that's creating plugins, improving the user interface, or adding new musical capabilities.

## How You Can Contribute

If you're here, you probably have an interest in music, programming, or ideally both. Here's how you can be part of this:

1. **Core Development**: Help improve core functionality or add new features. Clojure experience is valuable, but the architecture is designed to be learnable.

2. **Plugin Development**: Create plugins to extend Ooloi's capabilities. The plugin system supports any JVM language — Java, Kotlin, Scala, even proprietary commercial plugins.

3. **Documentation**: Help make complex concepts accessible. The goal is clarity without dumbing down.

4. **User Experience**: Contribute to interface design. The aim is intuitive interaction that serves creative work.

5. **Testing**: Help ensure reliability. With this level of architectural complexity, comprehensive testing isn't optional.

6. **Ideas and Vision**: Share your thoughts on how we can improve. Constructive feedback shapes the future.

## Getting Started

1. Read our [Code of Conduct](CODE_OF_CONDUCT.md) and [Contribution Guidelines](CONTRIBUTING.md). They set the tone for how we work together.

2. Start with the backend in the `backend/` directory. That's where the architectural foundations live.

3. Explore the documentation, especially the ADRs (Architecture Decision Records) that explain why certain choices were made.

4. Use the REPL extensively. Clojure's interactive development environment is powerful for understanding how the system works.

5. Review open issues or create new ones. The architecture is solid, but there's always more to build.

## What You're Joining

This isn't just another open-source project. It's the culmination of decades of understanding what music notation software needs to be, combined with the architectural discipline to build it right. 

You're joining something that's designed to outlast its creator, to enable work that hasn't been imagined yet, to solve problems that matter to musicians and developers alike. The foundations are solid; now we build the future on top of them.

The architecture is complete, but the work is just beginning. There are plugins to write, interfaces to design, capabilities to add. Most importantly, there are problems to solve that only emerge when you put powerful tools in the hands of creative people.

## A Personal Note

At 64, carrying more than five decades of programming experience and a parallel career as a composer, I've tried to encode into this architecture not just technical solutions, but the aesthetic judgments and performance intuitions that come from actually making music. 

The creative energy that might have gone into another opera has found expression in software architecture. It's a different kind of composition — one that enables other people's creative work rather than expressing my own. In many ways, it's more satisfying.

This is what happens when you take the time to get it right, when you resist the urge to rush, when you're willing to solve the hard problems properly. The result is something that can grow and evolve through the contributions of others while maintaining its essential character.

Now, let's make some music. On all levels.

/ Peter Bengtson
