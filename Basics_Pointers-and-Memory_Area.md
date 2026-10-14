# Pointers and Memory 

## Declaration vs initialization
Declaration = creating a variable: `int a;`
Initialization = giving it its first value: `int a = 5;` does both at once

RAM is a huge row of numbered boxes, one byte each. A box's number is its **address**. A variable is just a named group of boxes: an `int` takes 4 boxes, a `char` takes 1.

```c
int x = 5;
int *p = &x;   // p stores x's ADDRESS
```

| Address | Value | Name |
|---------|-------|------|
| 1000    | 5     | x    |
| 1008    | 1000  | p ← "points to" x |

- `&x` means "address of x"
- `*p` means "go to the address stored in `p`"
- `*p = 10;` changes `x` to 10

A pointer is just a variable whose value is an address. Java references and Python names are pointers in disguise, which is why this lesson matters for all three languages.

## Stack vs Heap

**Stack:** memory for function calls. It's automatic and very fast. Local variables live here and vanish when the function returns.

**Heap:** memory you request manually (`malloc` in C, `new` in C++). It lives until you free it, and it's slower and can fragment.

```c
int *arr = malloc(3 * sizeof(int)); // 12 bytes on the heap
arr[0] = 7;
free(arr); // forgetting this = memory leak
```

## Common Confusion: `p` vs `*p`

They are two different things:

- `p` holds an **address** (say 1000). It's a sticky note with a house address on it.
- `*p` means "go to that address and use what's stored there." It's the house itself.
- `*p` means "go to the address stored in p, and use the value that lives there." It's called dereferencing.

| Expression | Meaning | Value |
|------------|---------|-------|
| `p` | the address stored in p | 1000 |
| `*p` | the value at that address | 5 (or 20 after the write) |
| `&a` | address of a | 1000 |
| `&p` | address of p itself | 1008 |

`*` and `&` are opposites: `&` takes you from a value to its address, and `*` takes you from an address back to the value. So `*(&a)` is just `a`

So `p` doesn't have the value of `x`, it has the *location* of `x`. `*p` is what you get by following that location, which is why it equals `x`'s value.

```c
int arr[3] = {10, 20, 30};   // say starts at address 2000
int *p = arr;                // p = 2000
*(p + 1)                     // address 2004 → 20 (same as arr[1])
```

`p+1` means the address of the next datatype present. 
if `arr[1]` is at 2004, then `arr[2]` means address 2008 since int is 4 byte datatype. So `p+1` jumps accordingly.
When you write `arr` alone, C reads it as "the address where the array starts", which is 2000. So `p = 2000`. 

```
a = 4         → address 1000 holds 4
int *p = &a   → p holds 1000
*p = *p + 1   → go to 1000, read 4, write 5
```

Final values: `a = 5`, `*p = 5`, `p = 1000` (the address of a).

```c
int *p = &a;   // declare p AND set it to a's address. Declaration and initialisation both.
int *p;        // declare p, holds GARBAGE (random old bits)
int *p = NULL; // declare p, deliberately points to nothing
```

To create a pointer, must always begin with (datatype, like int) `*p`; 
Then you can use `*p` and `p` anywhere now.
