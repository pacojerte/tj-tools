# tj\_buffer #

## Introduction ##

A tj\_buffer is a simple structure providing an expandable, reusable
byte array.  You can create a tj\_buffer, push data into it, push some
more into it, then reset it and push some new data and the memory
allocated before the reset will be reused.  This enables owner code to
do things like reuse a memory buffer to handle a series of requests
without expensive memory allocation calls each time.

The external interface is defined in
tj\_buffer.h, and the internal code in tj\_buffer.c.  An extended sample of its use is in test/test-tj\_buffer.c.

## Dependencies ##

None.

## Usage ##

Add the directory containing tj\_buffer.h to your include path, and
tj\_buffer.c to your list of source files.

## API ##

The following outlines the API and code usage for a tj\_buffer.

### Management ###

Create a tj\_buffer using the tj\_buffer\_create function, and when you
are completely done with the buffer, tear it down using
tj\_buffer\_finalize:

```
  tj_buffer *buff1 = tj_buffer_create(0);
...
  tj_buffer_finalize(buff1);
```

The parameter to tj\_buffer\_create is the initial amount of memory to
allocate.  This can be 0 if you do not know how big the buffer can be.
If you know how big it will be, you can declare that here to save time
later.  Either way, it will grow to accommodate data as it is added.

tj\_buffer\_create() will return 0 if a problem occurs and the buffer
could not be created.

### Access ###

tj\_buffer\_getAllocated(buff) returns the current total extent of the
internal byte buffer held by buff.  tj\_buffer\_getUsed(buff) returns
the currently used extent of that buffer, the amount of data appended
since the buffer was created or the most recent reset.

```
  printf("Read %s; buffer[%d/%d]\n", argv[0],
         tj_buffer_getUsed(buff1), tj_buffer_getAllocated(buff1));
```

tj\_buffer\_setOwnership(buff, own) allows the caller to claim ownership
of the internal buffer (own=1), such that the memory is not freed when
the tj\_buffer is finalized.  It is assumed that the caller will then
take care of deallocating the memory.  By default the tj\_buffer owns
the buffer memory and will free it when finalized.

The actual data held by the tj\_buffer can be accessed via three
functions.  Core access is provided by tj\_buffer\_getBytes(buff), which
simply returns a pointer to the current buffer.
tj\_buffer\_getBytesAtIndex(buff, i) is simply a convenience function to
point at a subrange of that memory.  tj\_buffer\_getAsString(buff) is
simply a convenience function removing a cast on the internal buffer.
The calling code must ensure a null terminator is present, as
appropriate, tj\_buffer\_getAsString() does not add one.

```
  if (strcmp((char *) tj_buffer_getBytes(buff1), "HELLOHELLOHELLO"))
    FAIL("Accumulated string incorrect.");

  if (strcmp(tj_buffer_getAsString(buff1), "HELLOHELLO"))
    FAIL("Accumulated reset string incorrect.");
```

Note that the memory location returned by these functions may change as a result of future data append operations.

### Data ###

As noted above, a tj\_buffer is intended to be reusable.  The user can
add data, add more data, then reset the buffer and overwrite the
original data without allocating new memory.

Buffers are resent using tj\_buffer\_reset(buff).  Several functions are
provided to append data into the buffer:

  * tj\_buffer\_append(buff, bytes, n)  appends n bytes from bytes into the buffer.
  * tj\_buffer\_appendBuffer(buff, buff2) appends the used extent of buff2 into the buffer.
  * tj\_buffer\_appendString(buff, string)  appends the null terminated string to the very end of the buffer.
    * tj\_buffer\_appendAsString(buff, string)  appends the null terminated string to the end of the buffer, and assumes that the previous buffer contents were null terminated, overwriting that terminator if the buffer was not empty.
  * tj\_buffer\_appendFileStream(buff, fh)  reads contents of the fh into the buffer.  Internally it reads in a chunk at a time so that it can be pointed to stdin.  The page size is defined by TJ\_PAGE\_SIZE, a macro which may be changed at compile time; the default is 1024 bytes.
  * tj\_buffer\_appendFile(buff, fn)  is a convenience function that loads the file and uses tj\_buffer\_appendFileStream() to read the file.

```
  tj_buffer_append(buff1, (tj_buffer_byte *) "HELLO", 5);
  tj_buffer_append(buff1, (tj_buffer_byte *) "HELLO", 5);

  if (tj_buffer_getUsed(buff1) != 10)
    FAIL("Incorrect increased used amount.");
```

```
  tj_buffer_reset(buff1);

  tj_buffer_appendAsString(buff1, "HELLO");
  tj_buffer_appendAsString(buff1, "HELLO");

  if (strcmp(tj_buffer_getAsString(buff1), "HELLOHELLO"))
    FAIL("Accumulated reset string incorrect.");
```

```
  tj_buffer_reset(buff1);

  if ((f=fopen("test/mushi", "r")) == 0) {
    FAIL("Could not read test file test/mushi.");
  } else {
    tj_buffer_appendFileStream(buff1, f);
    fclose(f);
  }
```

Each of the append functions returns 1 if the call succeeds, and 0 otherwise.  If the call fails the previous buffer contents are untouched.

In all cases the caller retains ownership of the given data.


### Threads ###

tj\_buffer objects are not thread safe, so that mutexing costs aren't incurred for applications that don't need it.  However, the operations don't use any non-stack data outside of the tj\_buffer itself; if the tj\_buffer is locked by the calling code, operations on that buffer are thread safe.


### Logging ###

tj\_buffer has a simple logging and error reporting mechanism built in.
If NDEBUG is defined, logging is removed though error reporting still
happens.  This is a compile time switch, so there is no logging
performance cost if it is disabled.  By default the buffer prints log
messages to stdout and error messages to stderr.

This behavior can be changed by redefining several macros before compiling tj\_buffer.c, i.e., via -D:

  * TJ\_LOG(M, ...) takes a printf formatting string and matching varargs and consumes a report of a typical event---create, append, destroy, etc.
  * TJ\_ERROR(M, ...) takes a printf formatting string and matching varargs and consumes a report of a fault event, i.e., no memory.
  * TJ\_LOG\_STREAM defines the FILE `*` stream to which log messages are written.
  * TJ\_ERROR\_STREAM defines the FILE `*` stream to which log messages are written.

Using these, owner code could, e.g., send all log messages to stderr as well, or capture all errors in a file.  Note that the default code does not flush the stream, so crashes may lose data.  Otherwise, the behavior can be completely changed, e.g., to report to a logging structure, by changing the macros.

The defaults are defined as:

```
#define TJ_LOG(M, ...) fprintf(TJ_LOG_STREAM, "%s: " M "\n", __FUNCTION__, ##__VA_ARGS__)

#define TJ_ERROR(M, ...) fprintf(TJ_ERROR_STREAM, "[ERROR] %s:%s:%d: " M "\n", __FUNCTION__, __FILE__, __LINE__, ##__VA_ARGS__)

#define TJ_LOG_STREAM stdout

#define TJ_ERROR_STREAM stderr
```


### Tracking ###

Not implemented yet; the goal is to optionally enable runtime tracking
to establish statistics on typical buffer usage, such that initial
memory allocations may be optimized.