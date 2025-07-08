# Introduction to Specter for Ooloi

## What is Specter?

Specter is a powerful library for navigating and transforming nested data structures in Clojure. It's particularly useful for Ooloi, which deals with complex, hierarchical musical structures.

## Why Specter for Ooloi?

Ooloi manages intricate musical structures with multiple levels of nesting. Traditional Clojure operations can become verbose and error-prone when dealing with such deep nesting. Specter offers several advantages:

1. Declarative syntax: Describe what you want to do, not how to do it.
2. Performance: Specter operations are often faster than equivalent hand-written code.
3. Composability: Easily combine and reuse navigators and transformations.
4. Readability: Complex operations on nested structures become more readable.

## Performance Findings in Ooloi

1. **Consistent Outperformance**: Specter consistently outperformed the handcoded version across all piece sizes tested.

2. **Significant Speed Boost**: On average, the Specter implementation was approximately 2.5 to 3 times faster than the original handcoded version.

3. **Size-Dependent Performance**:
   - For small pieces (size 20), Specter was about 3.4 to 3.8 times faster.
   - For medium (size 100) and large pieces (size 1000), it maintained a performance advantage of about 2.5 to 2.7 times faster.

4. **Scalability**: The performance advantage was maintained even for large pieces, indicating good scalability.

These performance improvements have significant implications for Ooloi:

- **Enhanced Responsiveness**: Faster time-signature operations lead to improved overall responsiveness, especially in large or complex scores.
- **Better Scalability**: The consistent performance advantage across different sizes suggests better handling of extensive compositions.
- **Improved User Experience**: Reduced latency in time-signature-related operations enhances the user experience.
- **Potential for Complex Operations**: The speed boost opens possibilities for more complex real-time manipulations of musical scores.

While these results are promising, it's important to note that Specter's performance advantages may vary depending on the specific operation and data structure. As always, profiling and benchmarking should be done for critical operations to ensure optimal performance.

## Considerations for Using Specter in Ooloi

1. **Selective Application**: Apply Specter to operations where its performance benefits are most impactful, particularly for complex, deeply nested structures.
2. **Readability vs Performance**: Balance the use of Specter between improving code readability and enhancing performance. In some cases, simple operations might be more readable with traditional Clojure functions.
3. **Learning Curve**: Consider the learning curve for team members when introducing Specter more broadly in the project.
4. **Consistency**: Establish guidelines for when and how to use Specter to maintain consistency across the codebase.
5. **Testing**: Ensure thorough testing of Specter implementations, particularly for edge cases and complex transformations.

By judiciously applying Specter to suitable operations in Ooloi, we can potentially achieve significant performance improvements while maintaining or even enhancing code clarity and maintainability.

## Ooloi's Hierarchical Structure

Ooloi uses the following hierarchical structure for its main musical elements:

```
Piece
├── musicians
│   └── instruments
│       └── staves
│           └── voices
│               └── measures
│                   └── items (Pitch, Chord, Rest, Tuplet, Tremolando, etc.)
└── layouts
    └── page-views
        └── system-views
            └── staff-views
                └── measure-views
                    ├── glyphs
                    └── curves
```

Note that while the top-level structure is fixed, the musical elements within measures can be nested ad hoc. Some elements, like Tie and Slur, can form non-acyclical graphs.

## Key Concepts

1. Navigators: Define paths into nested structures (e.g., `ALL`, `FIRST`, `LAST`).
2. Selectors: Extract data from structures (e.g., `select`, `select-one`).
3. Transformers: Modify data in structures (e.g., `transform`, `setval`).

## Basic Usage in Ooloi

Here are some examples of how Specter can be used in Ooloi, reflecting the correct nesting structure and working with defrecords:

```clojure
(require '[com.rpl.specter :refer :all])

;; Get all pitches in a piece
(select [:musicians ALL :instruments ALL :staves ALL :voices ALL :measures ALL :items ALL
         (multi-path (pred #(instance? Pitch %))
                     (pred #(instance? Chord %) :pitches ALL))]
        piece)

;; Change the note of all C4 pitches to D4 in the first instrument
(transform [:musicians ALL :instruments FIRST :staves ALL :voices ALL :measures ALL :items ALL
            (multi-path (pred #(instance? Pitch %))
                        (pred #(instance? Chord %) :pitches ALL))
            #(= (:note %) :C4)]
           #(assoc % :note :D4)
           piece)

;; Add a staccato attachment to all Pitches and Chords in the piece
(transform [:musicians ALL :instruments ALL :staves ALL :voices ALL :measures ALL :items ALL
            (multi-path (pred #(instance? Pitch %))
                        (pred #(instance? Chord %)))]
           #(update % :attachments conj :staccato)
           piece)

;; Change all staccato attachments to tenuto
(transform [:musicians ALL :instruments ALL :staves ALL :voices ALL :measures ALL :items ALL
            (multi-path (pred #(instance? Pitch %))
                        (pred #(instance? Chord %)))
            :attachments ALL #(= % :staccato)]
           (constantly :tenuto)
           piece)

;; Get all glyphs in the first layout's first page
(select [:layouts FIRST :page-views FIRST :system-views ALL 
         :staff-views ALL :measure-views ALL :glyphs ALL] 
        piece)

;; Set the tempo for all measures in the piece
(transform [:musicians ALL :instruments ALL :staves ALL :voices ALL :measures ALL]
           #(assoc % :tempo 120)
           piece)

;; Remove all fermata attachments from the piece
(transform [:musicians ALL :instruments ALL :staves ALL :voices ALL :measures ALL :items ALL
            (multi-path (pred #(instance? Pitch %))
                        (pred #(instance? Chord %)))
            :attachments]
           #(remove (fn [attachment] (= attachment :fermata)) %)
           piece)
```

## Specter for Flexible Structures

For the more flexible lower-level musical elements, Specter's power really shines. Here's an example of working with nested structures:

```clojure
;; Find all staccato attachments in tuplets and change them to tenuto
(transform [:musicians ALL :instruments ALL :staves ALL :voices ALL :measures ALL :items ALL
            (recursive-path [] p
              [(if-path (pred #(instance? Tuplet %)) :items ALL)
               (multi-path (pred #(instance? Pitch %))
                           (pred #(instance? Chord %)))
               :attachments ALL])]
           #(if (= % :staccato) :tenuto %)
           piece)

;; Double the duration of all notes within tuplets
(transform [:musicians ALL :instruments ALL :staves ALL :voices ALL :measures ALL :items ALL
            (recursive-path [] p
              [(if-path (pred #(instance? Tuplet %)) :items ALL)
               (multi-path (pred #(instance? Pitch %))
                           (pred #(instance? Chord %)))])]
           #(update % :duration (fn [[num denom]] [(* 2 num) denom]))
           piece)
```

These examples demonstrate how Specter can navigate through arbitrarily nested structures to find and modify specific elements.

## Next Steps

As you work with Specter in Ooloi:

1. Familiarize yourself with Specter's navigator functions.
2. Practice creating paths that reflect Ooloi's hierarchical structure.
3. Explore ways to handle the flexible nesting of lower-level musical elements.
4. Consider creating custom navigators for common Ooloi operations.

Remember, while Specter is powerful, it's a tool to be used judiciously. Always prioritize code clarity and maintainability.

## Links 

The Specter repo and doc links:
https://github.com/redplanetlabs/specter

Good introduction:
https://www.futurile.net/2021/03/20/specter-nested-data-manipulation-for-clojure/

More specialised:
https://github.com/redplanetlabs/specter/wiki/Using-Specter-Recursively

Additional navigators:
https://github.com/latacora/eidolon
