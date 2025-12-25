# ADR: Choice of Clojure as the Primary Programming Language

## Status

Accepted

## Context

The choice of primary programming language for Ooloi is a fundamental decision that will significantly impact the project's development, performance, maintainability, and ecosystem. We need a language that can handle complex musical data structures, provide high performance for real-time notation rendering and manipulation, support cross-platform development, and foster a productive development environment for both core team and potential contributors.

## Decision

We will use Clojure as the primary programming language for developing Ooloi, both for the backend and the frontend (via JavaFX).

## Rationale

1. Functional Programming Paradigm:
   - Clojure's emphasis on immutable data structures and pure functions aligns well with representing and manipulating complex musical structures.
   - Facilitates easier reasoning about code, particularly beneficial for a complex domain like music notation.

2. JVM Compatibility:
   - Runs on the Java Virtual Machine, providing excellent performance and cross-platform compatibility.
   - Allows easy integration with existing Java libraries, crucial for GUI (JavaFX) and audio processing.

3. Concurrency Support:
   - Built-in support for Software Transactional Memory (STM) and other concurrency primitives, essential for handling real-time user interactions and background processing in a notation application.

4. REPL-Driven Development:
   - Interactive development environment promotes rapid prototyping and iterative development, crucial for a complex application like Ooloi.

5. Metaprogramming Capabilities:
   - Powerful macro system allows for extending the language, potentially useful for creating domain-specific abstractions for music notation.

6. Data-Oriented Design:
   - Clojure's focus on data and its manipulation aligns well with representing musical scores as data structures.

7. Ecosystem:
   - Rich ecosystem of libraries for GUI development (e.g., cljfx for JavaFX), data manipulation, and more.

8. Interoperability:
   - Excellent Java interop allows leveraging existing Java libraries and frameworks when needed.

9. Community:
   - Active and supportive community, particularly strong in domains requiring complex data manipulation.

10. Extensibility:
    - Facilitates the creation of a plugin system, allowing extensions in Clojure or any JVM language.

11. Performance:
    - Capable of high performance, crucial for real-time rendering and manipulation of complex scores.

12. Code Clarity:
    - Clojure's syntax and functional nature often lead to concise, readable code, beneficial for long-term maintenance.

13. Dynamic Typing:
    - Allows for flexible data structures, useful when dealing with varied musical notations and structures.

## Consequences

### Positive

- Simplified handling of complex, immutable data structures representing musical scores.
- Enhanced productivity through REPL-driven development and interactive programming.
- Strong concurrency support for responsive UI and efficient background processing.
- Ability to leverage both Clojure and Java ecosystems.
- Potential for expressive domain-specific abstractions through metaprogramming.
- Cross-platform compatibility via the JVM.

### Negative

- Learning curve for developers not familiar with Clojure or functional programming.
- Smaller pool of developers compared to more mainstream languages.
- Potential performance overhead in some scenarios compared to pure Java.
- Some libraries and tools may need to be wrapped or adapted for idiomatic Clojure use.

## Implementation Approach

1. Set up a Clojure development environment with REPL support for both backend and frontend.
2. Utilize Leiningen or tools.deps for project management and dependency handling.
3. Implement core data structures representing musical elements using Clojure's persistent data structures.
4. Leverage Clojure's concurrency primitives (atoms, refs, agents) for managing application state.
5. Use cljfx or a similar library for creating the JavaFX-based GUI in Clojure.
6. Develop a comprehensive test suite using Clojure testing libraries.
7. Implement the plugin system to allow for Clojure-based plugins, with potential for other JVM language support.

## Related Decisions

- [ADR-0001: Frontend-Backend Separation](0001-Frontend-Backend-Separation.md) - Establishes how Clojure is used to implement separate frontend and backend components
- [ADR-0004: STM for Concurrency](0004-STM-for-concurrency.md) - Details how Clojure's Software Transactional Memory enables thread-safe concurrent operations

## Additional Implementation Notes

8. Utilize Clojure's Java interop capabilities where necessary for performance-critical operations or integration with existing Java libraries.
9. Develop coding standards and best practices specific to Clojure for the Ooloi project.
10. Create documentation and examples to help onboard new developers to the Clojure codebase.

## Why Common Lisp is No Longer the Clear Choice

While Common Lisp was the language of choice for the original Igor Engraver project, several factors led to the decision to use Clojure for Ooloi instead:

1. JVM Ecosystem:
   - Clojure's JVM foundation provides access to a vast ecosystem of libraries and tools, particularly beneficial for GUI development (JavaFX) and audio processing.
   - Common Lisp, while powerful, has a more limited ecosystem, especially for modern GUI and multimedia libraries.

2. Cross-Platform Development:
   - Clojure, running on the JVM, offers superior cross-platform compatibility out of the box.
   - Common Lisp implementations vary in their cross-platform support and often require more effort to achieve consistent behavior across different operating systems.

3. Concurrency Support:
   - Clojure's built-in support for Software Transactional Memory (STM) and other modern concurrency primitives is more aligned with the needs of a contemporary, responsive application.
   - While Common Lisp has concurrency libraries, they are not as integral to the language and ecosystem as in Clojure.

4. Modern Language Features:
   - Clojure, being a more recent language, incorporates modern programming paradigms and features while retaining the power of Lisp.
   - Common Lisp, though still powerful, lacks some of the modern conveniences and idioms that Clojure provides.

5. Interoperability:
   - Clojure's seamless Java interop allows easy integration with a wide range of existing Java libraries and frameworks.
   - Common Lisp's interop capabilities, while present, are not as extensive or as smooth, particularly with regards to Java libraries.

6. Community and Resources:
   - Clojure has a growing, active community with a focus on modern application development.
   - The Common Lisp community, while dedicated, is smaller and less focused on contemporary application domains like music software.

7. Tooling and Development Experience:
   - Clojure benefits from modern development tools, including advanced REPL integrations with popular editors and IDEs.
   - Common Lisp's tooling, while powerful, often feels less integrated with contemporary development workflows.

8. Performance Considerations:
   - While both languages can be performant, Clojure's JVM foundation provides more predictable and often superior performance for the types of operations common in music notation software.

9. Hiring and Onboarding:
   - The pool of developers familiar with or interested in learning Clojure is likely larger than that for Common Lisp in the current job market.
   - Clojure's similarity to other popular languages (in terms of tooling and ecosystem) may make onboarding easier for new team members.

While Common Lisp remains a powerful and flexible language, Clojure's modern features, JVM integration, and active ecosystem make it a more suitable choice for developing Ooloi in today's software landscape. The decision to move from Common Lisp to Clojure represents an evolution in our approach, leveraging the strengths of Lisp-style programming in a more contemporary and well-supported environment.

## Notes

- Invest in Clojure training for team members not familiar with the language or functional programming.
- Regularly assess the impact of Clojure on development speed, code quality, and performance.
- Stay updated with Clojure ecosystem developments and new libraries that could benefit Ooloi.
- Consider creating Clojure wrappers or abstractions for frequently used Java libraries to ensure idiomatic usage.
- Develop guidelines for effective use of Clojure's immutable data structures and functional concepts in the context of a large-scale application like Ooloi.
- Explore Clojure's metaprogramming capabilities for creating music notation specific abstractions.
- Plan for potential integration or interoperation with non-Clojure codebases or libraries.
- Consider performance profiling and optimization strategies specific to Clojure and the JVM.
- Engage with the Clojure community for insights, best practices, and potential collaborations.
