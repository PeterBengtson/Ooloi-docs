# 🟡 Vector Path Descriptors (VPDs) in Ooloi

## Introduction

Vector Path Descriptors (VPDs) are a powerful feature in Ooloi that allow for efficient and flexible navigation and manipulation of the musical piece structure. They provide a consistent way to reference and modify nested elements within a piece, streamlining operations across the entire codebase.

For background on why VPDs were chosen as the primary addressing mechanism in Ooloi, see the [VPDs ADR](../ADRs/0008-VPDs.md).

## What are VPDs?

VPDs are vectors that describe a path to a specific element within the nested structure of a Ooloi piece. They can be used to navigate both the musical hierarchy (musicians, instruments, staves, voices, measures) and the visual hierarchy (layouts, page views, system views, staff views, measure views).

## Why Use VPDs?

1. **Consistency**: VPDs provide a uniform way to reference elements across different parts of the codebase.
2. **Efficiency**: They allow for compact representation of paths, which is especially useful in communication between frontend and backend.
3. **Flexibility**: VPDs can easily accommodate both musical and visual hierarchies.
4. **Scalability**: As the project grows, VPDs make it easier to add new operations without changing the fundamental way of addressing elements.

## How VPDs Work

VPDs use a combination of keywords and indices to navigate the piece structure. Here's a basic example:

```clojure
[:musicians 0 :instruments 1 :staves 0 :measures 3 :voices 0]
```

This VPD points to the 4th measure of the first voice of the first staff of the second instrument of the first musician.

For the layout hierarchy, a VPD might look like this:

```clojure
[:layouts 0 :page-views 2 :system-views 1 :staff-views 0 :measure-views 3]
```

This points to the 4th measure view of the first staff view of the second system view of the third page view of the first layout.

## Compact Form

Ooloi also supports a compact form of VPDs for brevity:

```clojure
[:m 0 1 0 0 3]  ; Equivalent to [:musicians 0 :instruments 1 :staves 0 :measures 3 :voices 0]
[:l 0 2 1 0 3]  ; Equivalent to [:layouts 0 :page-views 2 :system-views 1 :staff-views 0 :measure-views 3]
```

## Accessing Measure Contents

VPDs can be extended to access the contents of measures, which may have an ad-hoc structure. The general format for accessing measure contents is:

```clojure
[:m 0 1 0 0 3 :items <item-selector>]
```

VPDs support simple path navigation with numeric indices. For example, `[:m 0 1 0 0 3 :items 2]` gets the 3rd item in the measure.

For nested structures like tuplets, the VPD can be further extended:

```clojure
[:m 0 1 0 0 3 :items 0 :items 1]
```

This selects the second item in the first tuplet (assuming the first item in the measure is a tuplet) of the specified measure.

## Using VPDs in the API

As a general rule, all Ooloi API functions support VPDs as arguments, followed by the piece object or piece ID. The piece is required as an argument for several reasons:

1. It allows the API to work with multiple pieces simultaneously.
2. It ensures that operations are performed on the correct piece, especially in concurrent environments.
3. It supports the transaction system, allowing operations to be performed within the context of the current piece transaction.

The piece argument can be either:
- A Piece instance (typically used in backend operations)
- A string piece ID (typically used by the frontend to reference a specific piece)

Here are examples of using VPDs with API functions:

```clojure
(add-attachment [:m 0 1 0 0 3] piece-or-id :staccato)
(get-items [:m 0 1 0 0 3] piece-or-id)
(get-measure [:m 0 1 0 0 3] piece-or-id)
(set-time-signature [:m 0 1 0 0 3] piece-or-id [4 4])
(get-key-signature [:m 0 1 0 0 3] piece-or-id)
(set-tempo [:m 0 1 0 0 3] piece-or-id 120)
```

In these examples, `piece-or-id` can be either a Piece instance or a string piece ID. The API will handle the resolution of the piece ID to a Piece instance internally when necessary.

## VPDs and Transactions

VPDs are designed to work seamlessly with Ooloi's transaction system. When using VPDs, operations are automatically performed within the context of the current piece transaction.

## VPDs and Specter

Internally, Ooloi uses the Specter library to resolve VPDs and perform operations. This allows for efficient and powerful manipulation of the piece structure. For measure contents, Specter's powerful selection and transformation capabilities are leveraged to handle the ad-hoc structure efficiently.

## Best Practices

1. **💡 Prefer VPD operations over manual STM** - Use convenient VPD-based API operations like `(api/set-measure vpd piece-id 5 measure)` instead of manual STM like `(alter piece-ref assoc-in vpd measure)`. Why make life difficult when the VPD API handles transactions, validation, and error handling for you? See [API Convenience](POLYMORPHIC_API_GUIDE.md#-api-convenience-why-use-vpd-operations) for details.

2. Know that VPDs are an option when referencing elements within a piece. You don't need to use them for everything.

3. Prefer the verbose form of VPDs in code for readability, unless space is a significant constraint.

4. When creating new API functions, you must support VPDs as arguments for consistency with the rest of the system. The macros in models.core are a great help.

5. Always provide the piece or piece ID when using API functions with VPDs.

## Related Guides

- **[Timewalking Guide](TIMEWALKING_GUIDE.md)** - Timewalking operations return VPD tuples for precise location tracking
- **[Piece Manager Guide](PIECE_MANAGER_GUIDE.md)** - VPD operations in piece lifecycle management  
- **[Specter Guide](SPECTER.md)** - Specter provides the underlying implementation for VPD operations

## Conclusion

Vector Path Descriptors are a core feature of Ooloi, providing a powerful and flexible way to work with complex musical structures. By understanding and utilizing VPDs effectively, developers can create efficient and maintainable code that leverages the full power of the Ooloi system.
