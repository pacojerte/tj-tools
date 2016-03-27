# tj\_heap #

A tj\_heap is a macro defined customizable heap structure.  The user
defines the key and value types as well as the comparison function(s),
which are then expanded into a set of structures and functions.  This
approach provides efficient, reusable, heap code with compile time
type checking.  The structure supports adding, popping, peeking,
searching, and deleting from the interior of the heap.

```
// Create a max heap over float keys.
int floatmore(float a, float b) { return a  > b; }
TJ_HEAP_DECL(floatheap, float, char *, floatmore);

int main() {
  floatheap *fheap = floatheap_create(8);
  floatheap_add(fheap, 2.4, "2.4");
  floatheap_add(fheap, 0.3, "0.3");
  floatheap_add(fheap, 0.7, "0.7");
  floatheap_add(fheap, 7.8, "7.8");
  floatheap_add(fheap, 4.0, "4.0");

  float f;
  char *v;

  printf("Float max heap.\n");
  if (floatheap_peek(fheap, &f, &v)) {
    printf("Peek %4f %6s\n", f, v);
  }

  while (floatheap_pop(fheap, &f, &v)) {
    printf("  %4f %6s\n", f, v);
  }
  printf("\n");

  floatheap_finalize(fheap);
}
```


## Dependencies ##

None.


## Usage ##

Add tj\_heap.h to your include path.


## API ##

The following is the primary documentation for tj\_heap.  Given its
macro implementation, there are not doxygen comments for the
functions.  A more detailed example of usage is given in the
test/test-tj\_heap.c file.


### Declaration ###

A tj\_heap is an abstract structure.  A concrete class is declared
using the TJ\_HEAP\_DECL(name, key type, value type, comparison
function) macro.  The name is any valid C identifier, and will be the
typedef of your particular heap structure.  The key type may be any
scalar type and is the data over which elements in the heap are
ordered.  The value type may be any scalar type and is used to
associate data with the heap.  The comparison function is given as the
name of a function of type int func(key type, key type), which returns
1 if the first key should be higher in the heap than the second and 0
otherwise.  This enables customization to both the type of the keys,
as well as defining a min or max heap.

```
// Define an integer min heap.
int intless(int a, int b) { return a  < b; }
TJ_HEAP_DECL(intheap, int, char *, intless);

// Define a float max heap.
int floatmore(float a, float b) { return a  > b; }
TJ_HEAP_DECL(floatheap, float, char *, floatmore);
```


### Management ###

Heap instances may then be created and destroyed via X\_create(initial)
and X\_finalize(heap) functions, where X is the name given in the
tj\_heap declaration.  The parameter to the create function is the
initial number of elements to allocate space for in the heap.  It may
not be 0, but additional space will be allocated as necessary.  The
finalize function does not touch any memory connected to the heap
elements, e.g., objects pointed to by the heap value.  X\_create()
returns 0 if it is unsuccessful.

```
  intheap *heap = intheap_create(4);
...
  intheap_finalize(heap);

  floatheap *fheap = floatheap_create(8);
...
  floatheap_finalize(fheap);
```

### Using ###

Elements are added to the heap using the X\_add(heap, key, value).  The
calling code retains ownership of any memory pointed to by the key or
value.  When the heap's allocated space fills, it is doubled in size,
amortizing the cost of the realloc.  If X\_add() fails, e.g., via an
inability to reallocate memory to grow the heap, it returns 0, 1
otherwise.

The top heap element is popped via X\_pop(heap, key ptr, value ptr), storing
its key and value in the given locations.  The top element can be
examined without removing it from the heap via X\_peek(heap, key ptr,
value ptr).  Both functions return 0 if there are no elements
remaining.  Whether the top element is a minimum or
maximum is determined by the comparison function given in the heap declaration.

```
  float f;
  int k;
  char *v;

  intheap_add(heap, 923, "d");
  intheap_add(heap, 467, "b");

  while (intheap_pop(heap, &k, &v)) {
    printf("  %4d %6s\n", k, v);
  }
  printf("\n");


  floatheap_add(fheap, 7.8, "7.8");
  floatheap_add(fheap, 4.0, "4.0");

  if (floatheap_peek(fheap, &f, &v)) {
    printf("Peek %4f %6s\n", f, v);
  }
```

Elements can also be removed from the interior of the heap.  This
enables operations such as cancelling an event in a
schedule heap.

The first part of doing so is finding the element to remove using the X\_find(heap, comparator, data) function.  The comparator parameter is a function pointer of type int (comparator ptr)(void data ptr, key, value) which returns 1 on a match and 0 otherwise, optionally
using the passed data as the comparison.  X\_find() calls this on each element in the heap linearly until a match is
found or there are no more elements.

For example, if the heap values are strings, the following comparator
searches for a passed string:

```
int intfind(void *d, int k, char *v) { return (strcmp(v, d) == 0); }
```

If a match is found, its index in the heap is returned; the index may
be at 0, so a failure is indicated by a -1 result.  That index may
then be passed to the X\_remove(heap, index, key ptr, value ptr) function
which will remove the element, regardless of its location in the heap.

```
  int index;
  index = intheap_find(heap, &intfind, "a");
  if (index == -1) {
    printf("Could not find 'a'.\n");
  } else {
    printf("Found 'a' at %d.\n", index);
    intheap_remove(heap, index, &k, &v);
    printf("Removed %d:%s.\n", k, v);
  }
```

If the element cannot be removed, i.e., the heap is empty, X\_remove()
returns 0, 1 otherwise.  X\_pop() is implemented as x\_remove() on index
0; the two have identical functionality.


### Threads ###

tj\_heap objects are not thread safe, so that mutexing costs aren't
incurred for applications that don't need it.  However, the operations
don't use any non-stack data outside of the tj\_heap itself; if the
tj\_heap is locked by the calling code, operations on that heap are
thread safe.


### Logging ###

tj\_heap has a simple logging and error reporting mechanism built in.
If NDEBUG is defined, logging is removed though error reporting still
happens.  This is a compile time switch, so there is no logging
performance cost if it is disabled.  By default the heap prints log
messages to stdout and error messages to stderr.

This behavior can be changed by redefining several macros before
compiling tj\_buffer.c, i.e., via -D:

  * TJ\_LOG(M, ...) takes a printf formatting string and matching varargs and consumes a report of a typical event---create, append, destroy, etc.
  * TJ\_ERROR(M, ...) takes a printf formatting string and matching varargs and consumes a report of a fault event, i.e., no memory.
  * TJ\_LOG\_STREAM defines the FILE `*` stream to which log messages are written.
  * TJ\_ERROR\_STREAM defines the FILE `*` stream to which log messages are written.

Using these, owner code could, e.g., send all log messages to stderr
as well, or capture all errors in a file.  Note that the default code
does not flush the stream, so crashes may lose data.  Otherwise, the
behavior can be completely changed, e.g., to report to a logging
structure, by changing the macros.

The defaults are defined as:

```
#define TJ_LOG(M, ...) fprintf(TJ_LOG_STREAM, "%s: " M "\n", __FUNCTION__, ##__VA_ARGS__)

#define TJ_ERROR(M, ...) fprintf(TJ_ERROR_STREAM, "[ERROR] %s:%s:%d: " M "\n", __FUNCTION__, __FILE__, __LINE__, ##__VA_ARGS__)

#define TJ_LOG_STREAM stdout

#define TJ_ERROR_STREAM stderr
```


### Tracking ###

Not implemented yet; the goal is to optionally enable runtime tracking
to establish statistics on typical heap usage, such that initial
memory allocations and possibly algorithms may be optimized.