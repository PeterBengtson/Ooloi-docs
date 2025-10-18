# 🟡 Vector Path Descriptors (VPDs) in Ooloi

## Introduction

Vector Path Descriptors (VPDs) are a powerful feature in Ooloi that allow for efficient and flexible navigation and manipulation of the musical piece structure. They provide a consistent way to reference and modify nested elements within a piece, streamlining operations across the entire codebase.

For background on why VPDs were chosen as the primary addressing mechanism in Ooloi, see the [VPDs ADR](../ADRs/0008-VPDs.md).

## What are VPDs?

VPDs are vectors that describe a path to a specific element within the nested structure of a Ooloi piece. They come in two forms:

- **VPD Form (Compact)** - The default notation using prefix keywords: `[:m 0 1 0 3 0]` or `[:l 0 2 1]`
- **Navigator Form** - The expanded notation for Specter operations: `[:musicians 0 :instruments 1 :staves 0 :measures 3 :voices 0]`

VPDs can navigate both the musical hierarchy (musicians, instruments, staves, measures, voices) and the visual hierarchy (layouts, page views, system views, staff views, measure views).

### Key Concept: VPDs vs Navigators

**VPDs** are the compact form you use in code. **Navigators** are what Specter uses internally. The conversion happens automatically:

```clojure
[:m 0 1 0 3 0]  ; VPD (compact) - what you write
       ↓
[:musicians 0 :instruments 1 :staves 0 :measures 3 :voices 0]  ; Navigator - for Specter
```

## Why Use VPDs?

1. **Consistency**: VPDs provide a uniform way to reference elements across different parts of the codebase.
2. **Efficiency**: They allow for compact representation of paths, which is especially useful in communication between frontend and backend.
3. **Flexibility**: VPDs can easily accommodate both musical and visual hierarchies.
4. **Scalability**: As the project grows, VPDs make it easier to add new operations without changing the fundamental way of addressing elements.

## How VPDs Work

### VPD Form (Default - Always Use This)

Use the compact VPD form in all your code:

```clojure
[:m 0 1 0 3 0]  ; Musical: musician 0, instrument 1, staff 0, measure 3, voice 0
[:l 0 2 1 0 3]  ; Layout: layout 0, page-view 2, system-view 1, staff-view 0, measure-view 3
```

The prefix keywords map to hierarchical structures:
- `:m` expands to `[:musicians :instruments :staves :measures :voices]`
- `:l` expands to `[:layouts :page-views :system-views :staff-views :measure-views]`

### Navigator Form (Internal - Rarely Needed)

The navigator form is what gets used internally with Specter:

```clojure
[:musicians 0 :instruments 1 :staves 0 :measures 3 :voices 0]
[:layouts 0 :page-views 2 :system-views 1 :staff-views 0 :measure-views 3]
```

You rarely need to work with this form directly. The conversion happens automatically when you use VPD operations.

### Converting Between Forms

If you need to convert (rare), use these functions:

```clojure
(require '[ooloi.shared.ops.vpd :as vpd])

;; VPD → Navigator
(vpd/navigator [:m 0 1 0 3 0])
; => [:musicians 0 :instruments 1 :staves 0 :measures 3 :voices 0]

;; Navigator → VPD
(vpd/compact [:musicians 0 :instruments 1 :staves 0 :measures 3 :voices 0])
; => [:m 0 1 0 3 0]
```

Both functions are idempotent - calling them multiple times is safe.

## Accessing Voice Contents

VPDs can be extended to access the contents of voices (which contain items). Use the compact form:

```clojure
[:m 0 1 0 3 0 :items 2]  ; 3rd item in voice 0 of measure 3
```

For nested structures like tuplets, extend the path further:

```clojure
[:m 0 1 0 3 0 :items 5 :items 1]  ; 2nd item inside item 5 (if it's a tuplet)
```

## Using VPDs in the API

As a general rule, all Ooloi API functions support VPDs as arguments, followed by the piece object or piece ID. The piece is required as an argument for several reasons:

1. It allows the API to work with multiple pieces simultaneously.
2. It ensures that operations are performed on the correct piece, especially in concurrent environments.
3. It supports the transaction system, allowing operations to be performed within the context of the current piece transaction.

The piece argument can be either:
- A Piece instance (typically used in backend operations)
- A string piece ID (typically used by the frontend to reference a specific piece)

Here are examples of using VPDs with API functions (always use compact form):

```clojure
(add-attachment [:m 0 1 0 3 0] piece-or-id :staccato)
(get-items [:m 0 1 0 3 0] piece-or-id)
(get-voice [:m 0 1 0 3 0] piece-or-id)
(set-name [:m 0 1] piece-or-id "Violin I")
(set-time-signature [] piece-or-id 0 [4 4])  ; Root piece, measure 0
(set-tempo [] piece-or-id 0 120)              ; Root piece, measure 0
```

In these examples, `piece-or-id` can be either a Piece instance or a string piece ID. The API will handle the resolution of the piece ID to a Piece instance internally when necessary.

### Important: Always Use Compact VPD Form

```clojure
✅ CORRECT:  (get-voice [:m 0 1 0 3 0] piece)
❌ WRONG:    (get-voice [:musicians 0 :instruments 1 :staves 0 :measures 3 :voices 0] piece)
```

The compact form is the standard. Navigator form is only for internal Specter operations.

## VPDs and Transactions

VPDs are designed to work seamlessly with Ooloi's transaction system. When using VPDs, operations are automatically performed within the context of the current piece transaction.

## VPDs and Specter

Internally, Ooloi uses the Specter library for path-based operations. Here's how it works:

1. **You write**: `[:m 0 1 0 3 0]` (compact VPD)
2. **System converts**: `[:musicians 0 :instruments 1 :staves 0 :measures 3 :voices 0]` (navigator)
3. **Specter navigates**: Uses the navigator form to access/modify the data

### Path Extension Pattern

Internal operations extend VPDs with attribute names to build Specter paths:

```clojure
;; Setting the name of instrument 1 in musician 0
;; Internally: (conj [:m 0 1] :name) => [:musicians 0 :instruments 1 :name]
(set-name [:m 0 1] piece "Violin I")
```

This pattern (`conj vpd attr-name`) is how VPDs interface with Specter for attribute access.

## Best Practices

1. **💡 Always use compact VPD form** - Use `[:m 0 1 0 3 0]` not `[:musicians 0 :instruments 1 :staves 0 :measures 3 :voices 0]`. The compact form is the standard notation for all user code. Navigator form is only for internal Specter operations.

2. **💡 Prefer VPD operations over manual STM** - Use convenient VPD-based API operations like `(api/set-measure vpd piece-id 5 measure)` instead of manual STM like `(alter piece-ref assoc-in vpd measure)`. Why make life difficult when the VPD API handles transactions, validation, and error handling for you? See [API Convenience](POLYMORPHIC_API_GUIDE.md#-api-convenience-why-use-vpd-operations) for details.

3. Know that VPDs are an option when referencing elements within a piece. You don't need to use them for everything.

4. When creating new API functions, you must support VPDs as arguments for consistency with the rest of the system. The macros in models.core are a great help.

5. Always provide the piece or piece ID when using API functions with VPDs.

6. **💡 Conversion is idempotent** - Both `vpd/navigator` and `vpd/compact` can be called multiple times safely. The system handles this correctly.

## Related Guides

- **[Timewalking Guide](TIMEWALKING_GUIDE.md)** - Timewalking operations return VPD tuples for precise location tracking
- **[Piece Manager Guide](PIECE_MANAGER_GUIDE.md)** - VPD operations in piece lifecycle management  
- **[Specter Guide](SPECTER.md)** - Specter provides the underlying implementation for VPD operations

## Conclusion

Vector Path Descriptors are a core feature of Ooloi, providing a powerful and flexible way to work with complex musical structures. By understanding and utilizing VPDs effectively, developers can create efficient and maintainable code that leverages the full power of the Ooloi system.
