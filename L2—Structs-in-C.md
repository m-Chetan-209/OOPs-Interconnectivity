# OOP Notes: Structs, Where Objects Begin

## 1. What is a struct?

A **struct** groups several variables into one block of memory. Fields sit side by side in the order you declare them. (Field = variable inside the struct.)

```c
struct Point { int x; int y; };
struct Point p = {3, 4};   // p lives on the stack
```

```
Stack
┌─────────────┐
│ p.x = 3     │  ← p is these 8 bytes
│ p.y = 4     │
└─────────────┘
```

`sizeof(struct Point)` is 8. The struct is just those 8 bytes, with no hidden extras.

---

## 2. Struct on the heap, using a pointer

```c
struct Point *q = malloc(sizeof(struct Point));  // malloc creates the space
q->x = 10;     // same as (*q).x = 10
free(q);
```

- `->` means "follow the pointer, then access the field."
- `struct Point` is a type you created, just like `int`. So `struct Point *q` reads as "q is a pointer to a `struct Point`", the same pattern as `int *p`.
- `*q` is the struct object itself.
- `q` (the pointer) is on the stack. The struct it points to is on the heap.
- `.` always needs a **struct** on its left. `p` is a struct, and `(*q)` is a struct, but `q` alone is just an address.

---

## 3. Why this is "OOP"?
## Example-

```c
void move(struct Point *self, int dx) { self->x += dx; }

struct Point p = {3, 4}; // x is 3 and y is 4
move(&p, 5);
```

`self` receives the address of `p`, so `self->x += dx` goes to `p`'s memory and changes `x` from 3 to 8. Afterwards `p` is `{8, 4}`.

- **Confusion:** There is only 1 struct. p and *self mean the same struct because *self = &p.

An **object** is data (`p`), and `move` is an operation on that data. The only link is that the object's address is passed as the first argument.

- C++: `p.move(5)` secretly passes `&p` as `this`
- Python: `p.move(5)` secretly passes `p` as `self`

---

## 4. Optimizing padding

**Alignment:** the CPU wants each field to start at an address divisible by its own size. The compiler inserts empty bytes (**padding**) to make that happen, and the total size rounds up to a multiple of the largest field's alignment.

| Type | Starts at multiples of |
|---|---|
| `char` | 1 |
| `int` | 4 |
| `double` | 8 |

### Example: `struct A`

```c
struct A { char a; int b; char c; };
```

| Offset | What | Why |
|---|---|---|
| 0 | `a` (1 byte) | char can go anywhere |
| 1, 2, 3 | padding | next free spot is 1, but `int` needs a multiple of 4, so skip to 4 |
| 4-7 | `b` (4 bytes) | 4 is divisible by 4 ✓ |
| 8 | `c` (1 byte) | any address is fine |
| 9, 10, 11 | padding | round 9 up to a multiple of 4 |

Total: **12 bytes**, of which only 6 hold real data.

### Reordered: `struct B`

```c
struct B { int b; char a; char c; };   // b, a, c, 2 padding = 8 bytes
```

Same data, but reordering saves 4 bytes per object (33%). Across a million objects, that's 4 MB saved.

### Example: `struct T`

```c
struct T { char a; char b; int c; };
```

| Offset | What |
|---|---|
| 0 | `a` |
| 1 | `b` |
| 2-3 | padding (`int` needs a multiple of 4, so skip to 4) |
| 4-7 | `c` |

Total: **8 bytes**. There is no padding at the end because 8 is already a multiple of 4.

### Example: `struct S` (with a double)

```c
struct S { char a; double b; char c; };   // 24 bytes (bad order)
struct S { double b; char a; char c; };   // 16 bytes (good order)
```

Good order: `b` at 0, `a` at 8, `c` at 9, then 6 bytes of end padding. Total 16, with only 10 bytes of real data.
**DECLARE LARGE FIELDS FIRST, SMALL FIELDS AFTER**

### Takeaways

- Declare large fields first, small fields after.
- The real goal: **group small fields together so they fill each other's padding.**
- Why end padding exists: in an array, the next struct starts right after the previous one, so the size must keep every field aligned.
- Reordering reduces padding but can't always eliminate it.

---
