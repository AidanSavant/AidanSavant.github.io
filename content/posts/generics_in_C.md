+++
title = "Generics in C" 
date  = "2025-03-05"
tags  = ["C", "Headache"]
+++

Generics functions and generic containers in C.
<!--more-->

# Introduction
There will be a point in your software development career where you'll 
need to write code that can be applied to a set of types rather than a singular type.

This is called "generic programming", more rigorously known as "parametric polymorphism",
and it's a common feature of most programming languages.

Typically, writing generics in those other programming languages looks something like this:
```rust
fn swap<T>(T x, T y) {
    T tmp = x;
    x = y;
    y = tmp;
}
```

Basically, we're describing some "abstract" type that will be replaced with some "concrete"
type upon calling ``swap``. You can think of this as "type substitution" or "type rewriting"
because that is commonly how it's implemented. 

This notion of "type substitution" is typically implemented as the 
compiler stamping out different versions of `swap` with the given "concrete" type. 
This process is called "monomorphization".

Let's look at monomorphization in action with our ``swap`` example:
```rust
int a = 5;
int b = 10;
swap(a, b)

string c = "foo"
string d = "bar"
swap(c, d)
```

so with our integer variables the compiler is actually calling an underlying 
function that looks like this whenever call our generic ``swap`` function:
```c
fn swap_int(int x, int y) {
    int tmp = x;
    x = y;
    y = tmp;
}
```

and likewise for our string variables. 

But how does the compiler know which underlying function to call?
Well, the compiler is smart and can usually *infer* the concrete type, 
otherwise we have to explicitly annotate it which typically looks like:
```rust
swap<int>(a, b)
swap<string>(c, d)
```

# Rewrite that in C for me...
That's cool and all but... what is the syntax for that in C? 

It doesn't exist 😃!!!

# Types of approaches
Anyways... C offers two (technically three) mechanisms for us to indirectly implement generics.

1.) Type erasure (``void pointers`` & ``enum`` tags)

2.) Monomorphization (macros)

(You can also use unions & enum tags 
but I'm just going to talk about first two)

both of which have their own pros and cons, to keep it simple and align with the introduction
we'll stick to monomorphization.

# Generic functions
Let's rewrite our little pseudo-code example in C with macros:
```c
#define swap(T, x, y) \
    do {              \
        T tmp = x;    \
        x = y;        \
        y = tmp;      \
    } while(0)        \

int main(void) {
    int a = 5, b = 10;
    swap(int, a, b);

    const char *c = "foo", *d = "bar";
    swap(const char *, c, d);
}
```
You can test the code out and print each variable to ensure it properly swapped.

C allows us to use macros to directly swap our code at compile-time with the given
concrete type that is returned to us via ``typeof`` (which was standardized for C2x in 2023)

# Generic containers
So now we know how to create generic functions, but what about generic data structures?
It's not too much different, but it requires a little bit more care and thought due to
some subtle issues that we'll cover.

Let's implement a generic container for a linked list that'll follow the form of:
```rust
struct ListNode<T> {
    T data;
    ListNode<T> fd, bk;
}

struct List<T> {
    ListNode<T> head, tail;
}
```

We'll need to create some sort of macro system for our ``ListNode`` and ``List`` type,
we can do it like we did our original example, but this is a little different since
we need to name these types.

Alas the question arises, how do we name theses types? How should we
 differentiate between the multiple concrete types?

Let's think about it, what is a unique way we can associate the concrete type 
to our type name? The type name!

Let's look at an example:
```c
#define DECL_LIST_NODE(T)             \
    typedef struct ListNode_##T {     \
        T data;                       \
        struct ListNode_##T *fd, *bk; \
    } ListNode_##T;                   \

#define DECL_LIST(T)                  \
    DECL_LIST_NODE(T)                 \
                                      \
    typedef struct List_##T {         \
        ListNode_##T *head, *tail;    \
    } List_##T;                       \
```

Looks good... right?

# Subtle issue #1
The 1st issue is how constricting our type declaration is, 
ergo we have no flexibility in what the name of the type is. 

Okay so how do we fix this?
Interestingly enough, the solution to this issue is also the solution to the 2nd issue!

# Subtle issue #2
The 2nd issue is in how macros actually work. Macros preform lexical token substutition,
so what happens if we want our ``data`` to be ``volatile`` or ``unsigned``? How will the 
macro substitute our type then?

Well, let's try it and see what happens when we do ``DECL_LIST(const int)``:

```c
typedef struct ListNode_const int { 
  const int data; 
  struct ListNode_const int *fd, *bk; 
} ListNode_const int; 

typedef struct List_const int { 
  ListNode_const int *head, *tail; 
} List_const int;
```

Well... that doesn't look right... (this code also breaks with `T*` types)


To fix this, we'll use indirection and have another parameter to label the type:
```c
#include <stdio.h>
#include <stdlib.h>

#define DECL_LIST_NODE(NAME, T)          \
    typedef struct NAME {                \
        T data;                          \
        struct NAME *fd, *bk;            \
    } NAME;

#define DECL_LIST(NAME, T)               \
    DECL_LIST_NODE(NAME##_node, T)       \
                                         \
    typedef struct NAME {                \
        struct NAME##_node *head, *tail; \
    } NAME;                              \

DECL_LIST(list, const int)
```

let's see what the generated code looks like:
```c
typedef struct list_node { 
    const int data; 
    struct list_node *fd, *bk; 
} list_node;

typedef struct List { 
    struct list_node *head, *tail; 
} List;
```

looks correct to me! Now let's move onto implementing some functions for the generic type.

# Generic container functions
Here's a list of generic container functions we'll implement for our list:
1. Inserting at the tail

2. Popping nodes off at the tail

3. Searching for a node

4. Destroying the list

5. Print the list 

Side note: functions for 3,4, and 5 will require more indirection 
for our macro than you might initially think.

The generate all of these functions at compile-time, we'll use another macro called ``DEFINE_LIST``
which will generate the generic functions for the list. 

Side note: With each extension of the code I'll omit the previous extension
just to keep the code simpler

# Generic node creation
```c
#define DEFINE_LIST(NAME, T)                       \
    NAME##_node *new_##NAME##_node(T data) {       \
        NAME##_node *node = malloc(sizeof(*node)); \
        if(!node) {                                \
            return NULL;                           \
        }                                          \
                                                   \
        node->data = data;                         \
        node->fd = node->bk = NULL;                \
                                                   \
        return node;                               \
    }                                              \
```

# Generic Tail Insertion
```c
#define DEFINE_LIST(NAME, T)                          \
    bool NAME##_push(NAME *list, T data) {            \
        if(!list) {                                   \
            return false;                             \
        }                                             \
                                                      \
        NAME##_node *node = new_##NAME##_node(data);  \
                                                      \
        if(!list->head) {                             \
            list->head = list->tail = node;           \
            list->len += 1;                           \
                                                      \
            return true;                              \
        }                                             \
                                                      \
        list->tail->fd = node;                        \
        node->bk = list->tail;                        \
        list->tail = node;                            \
                                                      \
        list->len += 1;                               \
                                                      \
        return true;                                  \
    }                                                 \
```

Fairly straight forward from our previous examples, 
however for the 2nd function we're going to have to 
take some careful consideration for how we approach it.

## Generic tail deletion
So what do we do when we delete a node? Well, we set the pointers accordingly and
free the dataaa.... oh wait... what do we free if the concrete type is a complex type?

Let's say we have a structure:
```c
typedef struct {
    char *malloc_buff;
} Foo;
```

and we define our operations on the type ``Foo``, it wouldn't make sense to call ``free``
on the object, so the user has to provide their own function to destruct the node.

Implementation:
```c
#define DEFINE_LIST(NAME, T, DTOR)            \
    bool NAME##_pop(NAME *list) {             \
        if(!list || !list->tail) {            \
            return false;                     \
        }                                     \
                                              \
        NAME##_node *tmp = list->tail;        \
        if(list->head == list->tail) {        \
            list->head = list->tail = NULL;   \
        }                                     \
        else {                                \
            list->tail = list->tail->bk;      \
            list->tail->fd = NULL;            \
        }                                     \
                                              \
        free(tmp);                            \
        DTOR(tmp->data);                      \
                                              \
        list->len -= 1;                       \
                                              \
        return true;                          \
    }                                         \

typedef struct {
    char *malloc_buff;
} Foo;

void foo_dtor(Foo fooer) {
    if(!fooer.malloc_buff) {
        return;
    }
    
    free(fooer->malloc_buff);
}

DEFINE_LIST(List, Foo, foo_dtor)
```
As you can see we treat ``DTOR`` as a function that is responsible
for the destruction of ``tmp->data``. (the data the node contains)

# Generic node search
Searching for a given node requires us to supply another function
that will be responsible for equality checks of complex types.

Implementation:
```c
#define DEFINE_LIST(NAME, T, IS_EQU)              \
    NAME##_node *NAME##_find(NAME list, T data) { \
        if(!list.head) {                          \
            return NULL;                          \
        }                                         \
                                                  \
        NAME##_node *tmp = list.head;             \
        while(tmp) {                              \
            if(IS_EQU(tmp->data, data)) {         \
                return tmp;                       \
            }                                     \
                                                  \
            tmp = tmp->fd;                        \
        }                                         \
                                                  \
        return NULL;                              \
    }                                             \

bool is_equ(int l, int r) { return l == r; }

DEFINE_LIST(List, int, is_equ)
```
We define a ``is_equ`` function so that we know how to compare the
concrete types just in case it's more complex than just an ``==``, 
such as a ``const char *`` concrete type where we'd need to call
``strcmp``.

# Generic list destruction
```c
#define DEFINE_LIST(NAME, T, DTOR)         \
    bool NAME##_destroy(NAME *list) {      \
        if(!list || !list->head) {         \
            return false;                  \
        }                                  \
                                           \
        NAME##_node *tmp = list->head;     \
        while(tmp) {                       \
            NAME##_node *tmp_fd = tmp->fd; \
                                           \
            DTOR(tmp->data);               \
            free(tmp);                     \
                                           \
            tmp = tmp_fd;                  \
        }                                  \
                                           \
        return true;                       \
    }                                      \
```

We reuse the same dtor to call on each existing
node to ensure no memory leaks happen.

# Generic list printing
```c
#define DEFINE_LIST(NAME, T, PRINTER) \
    void NAME##_print(NAME list) {    \
        if(!list.head) {              \
            return;                   \
        }                             \
                                      \
        NAME##_node *tmp = list.head; \
        while(tmp) {                  \
            PRINTER(tmp->data);       \
            tmp = tmp->fd;            \
        }                             \
    }                                 \
```
We have to accept a printer so we know how to format the data.

# Full implementation
```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <stdbool.h>

#define DECL_LIST_NODE(NAME, T)                      \
    typedef struct NAME {                            \
        T data;                                      \
        struct NAME *fd, *bk;                        \
    } NAME;

#define DECL_LIST(NAME, T)                           \
    DECL_LIST_NODE(NAME##_node, T)                   \
                                                     \
    typedef struct NAME {                            \
        struct NAME##_node *head, *tail;             \
        size_t len;                                  \
    } NAME;

#define DEFINE_LIST(NAME, T, IS_EQU, DTOR, PRINTER)  \
    NAME##_node *new_##NAME##_node(T data) {         \
        NAME##_node *node = malloc(sizeof(*node));   \
        if(!node) {                                  \
            return NULL;                             \
        }                                            \
                                                     \
        node->data = data;                           \
        node->fd = node->bk = NULL;                  \
                                                     \
        return node;                                 \
    }                                                \
                                                     \
    bool NAME##_push(NAME *list, T data) {           \
        if(!list) {                                  \
            return false;                            \
        }                                            \
                                                     \
        NAME##_node *node = new_##NAME##_node(data); \
                                                     \
        if(!list->head) {                            \
            list->head = list->tail = node;          \
            list->len += 1;                          \
                                                     \
            return true;                             \
        }                                            \
                                                     \
        list->tail->fd = node;                       \
        node->bk = list->tail;                       \
        list->tail = node;                           \
                                                     \
        list->len += 1;                              \
                                                     \
        return true;                                 \
    }                                                \
                                                     \
    bool NAME##_pop(NAME *list) {                    \
        if(!list || !list->tail) {                   \
            return false;                            \
        }                                            \
                                                     \
        NAME##_node *tmp = list->tail;               \
        if(list->head == list->tail) {               \
            list->head = list->tail = NULL;          \
        }                                            \
        else {                                       \
            list->tail = list->tail->bk;             \
            list->tail->fd = NULL;                   \
         }                                           \
                                                     \
        free(tmp);                                   \
                                                     \
        list->len -= 1;                              \
                                                     \
        return true;                                 \
    }                                                \
                                                     \
    NAME##_node *NAME##_find(NAME list, T data) {    \
        if(!list.head) {                             \
            return NULL;                             \
        }                                            \
                                                     \
        NAME##_node *tmp = list.head;                \
        while(tmp) {                                 \
            if(IS_EQU(tmp->data, data)) {            \
                return tmp;                          \
            }                                        \
                                                     \
            tmp = tmp->fd;                           \
        }                                            \
                                                     \
        return NULL;                                 \
    }                                                \
                                                     \
    void NAME##_print(NAME list) {                   \
        if(!list.head) {                             \
            return;                                  \
        }                                            \
                                                     \
        NAME##_node *tmp = list.head;                \
        while(tmp) {                                 \
            PRINTER(tmp->data);                      \
            tmp = tmp->fd;                           \
        }                                            \
    }                                                \
                                                     \
    bool NAME##_destroy(NAME *list) {                \
        if(!list || !list->head) {                   \
            return false;                            \
        }                                            \
                                                     \
        NAME##_node *tmp = list->head;               \
        while(tmp) {                                 \
            NAME##_node *tmp_fd = tmp->fd;           \
                                                     \
            DTOR(tmp->data);                         \
            free(tmp);                               \
                                                     \
            tmp = tmp_fd;                            \
        }                                            \
                                                     \
        return true;                                 \
    }                                                \


/* == Example == */
void int_dtor(int _) { (void) _; }
void str_dtor(const char *_) { (void) _; }

bool int_equ(int l, int r) { return l == r; }
bool str_equ(const char *l, const char *r) { return !strcmp(l, r); }

void int_printer(int data) { printf("%d\n", data); }
void str_printer(const char *data) { puts(data); }

DECL_LIST(List, int)
DEFINE_LIST(List, int, int_equ, int_dtor, int_printer)

DECL_LIST(List2, const char *)
DEFINE_LIST(List2, const char *, str_equ, str_dtor, str_printer)

int main(void) {
    List list = {0};

    for(int i = 1; i <= 10; ++i) {
        List_push(&list, i);
    }

    List_pop(&list);
    List_pop(&list);

    List_print(list);

    List_destroy(&list);

    List2 list2 = {0};

    const char *strs[] = {
        "abc", "def", "ghi"
    };

    for(int i = 0; i < 3; ++i) {
        List2_push(&list2, strs[i]);
    }

    List2_print(list2);

    List2_destroy(&list2);
}
```

# Final Notes
In the relationship between ``DECL_LIST`` and ``DEFINE_LIST`` there is a 
possibility where we can mismatch the name of the type and the underlying
concrete type, there are some possible ideas I've thought of (keeping track
of initialized types in some internalized LUT) but have yet to come to
frutition due to laziness and low IQ...

# Appendix
### do {} while(0)
You might've noticed the ``do {} while(0)`` piece of code in the C macros and was probably
puzzled by it, because it's seemingly redundant and useless?

While that is a good thought, it's not entirely true. See, with multi-line macros sometimes
they can get pasted incorrectly whenever we're nested the macro-call.

[Read more here](https://stackoverflow.com/questions/257418/do-while-0-what-is-it-good-for)

### Token Concatenation
You might've seen the strange syntax of ``##``, that is the token concatenation
operator and it does exactly what the name describes.

It'll take the token of the left-hand side and combine it with the right-hand
side, this was particularly useful with concatenating the token we define for
"NAME" instead of literally "NAME".

[Read more here](https://gcc.gnu.org/onlinedocs/cpp/Concatenation.html)

### Type-erasure Generics
I had briefly mentioned type-erasure with the use of ``void *``s and ``enum``
tags to represent different types. 

This is generally what libc functions utilize to achieve genericness,
some notable functions from libc would be ``qsort``, ``bsearch``, ``memcpy``, etc.

Here's a copied example from chatGPT because I don't feel like writing
all of that out:

```c
int main() {
    int x = 10, y = 20;
    float f1 = 1.1f, f2 = 2.2f;

    // Swap two integers
    printf("Before swap: x = %d, y = %d\n", x, y);
    swap(&x, &y, sizeof(int));
    printf("After swap: x = %d, y = %d\n", x, y);

    // Swap two floats
    printf("Before swap: f1 = %.2f, f2 = %.2f\n", f1, f2);
    swap(&f1, &f2, sizeof(float));
    printf("After swap: f1 = %.2f, f2 = %.2f\n", f1, f2);

    return 0;
}
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

// Define an enum to represent the different types
typedef enum {
    INT,
    FLOAT,
    DOUBLE,
    CHAR
} DataType;

// A generic print function using type erasure with void *
void print(void *value, DataType type) {
    switch (type) {
        case INT:
            printf("%d\n", *(int *)value);  // Cast void * to int *
            break;
        case FLOAT:
            printf("%f\n", *(float *)value);  // Cast void * to float *
            break;
        case DOUBLE:
            printf("%lf\n", *(double *)value);  // Cast void * to double *
            break;
        case CHAR:
            printf("%c\n", *(char *)value);  // Cast void * to char *
            break;
        default:
            printf("Unknown type\n");
    }
}

// A generic swap function using type erasure with void *
void swap(void *a, void *b, size_t size) {
    void *temp = malloc(size);  // Allocate temporary space for swapping

    // Swap the data
    memcpy(temp, a, size);       // Copy data from a to temp
    memcpy(a, b, size);          // Copy data from b to a
    memcpy(b, temp, size);       // Copy data from temp to b

    free(temp);  // Don't forget to free the allocated memory
}

int main() {
    // Variables of different types
    int x = 5, y = 10;
    float f1 = 3.14f, f2 = 2.71f;
    double d1 = 2.718, d2 = 3.14159;
    char c1 = 'A', c2 = 'B';

    // Using the generic print function
    print(&x, INT);
    print(&f1, FLOAT);
    print(&d1, DOUBLE);
    print(&c1, CHAR);

    // Using the generic swap function
    printf("Before swap: x = %d, y = %d\n", x, y);
    swap(&x, &y, sizeof(int));
    printf("After swap: x = %d, y = %d\n", x, y);

    printf("Before swap: f1 = %.2f, f2 = %.2f\n", f1, f2);
    swap(&f1, &f2, sizeof(float));
    printf("After swap: f1 = %.2f, f2 = %.2f\n", f1, f2);

    printf("Before swap: d1 = %.5f, d2 = %.5f\n", d1, d2);
    swap(&d1, &d2, sizeof(double));
    printf("After swap: d1 = %.5f, d2 = %.5f\n", d1, d2);

    printf("Before swap: c1 = %c, c2 = %c\n", c1, c2);
    swap(&c1, &c2, sizeof(char));
    printf("After swap: c1 = %c, c2 = %c\n", c1, c2);

    return 0;
}
```


# Revisions
Honestly, I didn't revise this at all and there's probably grammatical errors and
redundancy all over the place. If you notice any, please send me an email that is found
on the homepage of the website.

