# tj\_template #

tj\_template provides for simple variable expansion within a (text)
buffer.  It proceeds by creating a tj\_template\_variables object,
adding variable substitions to that object, and then applying it to a
source template buffer to create a destination buffer.

The following is a simple example:

```
  tj_buffer *target = tj_buffer_create(0);
  tj_buffer *src = tj_buffer_create(0);

  tj_buffer_appendString(src, "HEL$XLO!");

  tj_template_variables *vars = tj_template_variables_create();
  tj_template_variables_setFromString(vars, "X", "mushi");

  tj_template_variables_apply(vars, target, src);

  if (strcmp(tj_buffer_getAsString(target), "HELmushiLO!")) {
    FAIL("Did not get expected accumulated string.");
  }
```

The external interface is defined in tj\_template.h, and the internal
code in tj\_template.c. An extended sample of its use is in
test/test-tj\_template.c.  In addition to notes below, the API is
reasonably documented in the header file.

## Dependencies ##

tj\_template uses tj\_buffer objects.


## Usage ##

Add the directory containing tj\_template.h and tj\_buffer.h to your include path, and tj\_template.c and tj\_buffer.c to your list of source files.


## API ##

The following outlines the API and code usage for tj\_template\_variables.


### Management ###

tj\_template\_variables objects are created and destroyed using
tj\_template\_variables\_create() and
tj\_template\_variables\_finalize(vars) calls.  Finalizing the object
finalizes all substitutions defined so far.

```
  tj_template_variables *vars = tj_template_variables_create();
...
  tj_template_variables_finalize(vars);
```

### Define Substitutions ###

Three functions are provided for defining substitutions within a
tj\_template\_variables object:
  * tj\_template\_variables\_setFromString(vars, variable, value)  sets the variable to the string value.
  * tj\_template\_variables\_setFromFileStream(vars, variable, file stream)  sets the variable to the contents of the file stream; this may be stdin.
  * tj\_template\_variables\_setFromFile(vars, variable, filename)  sets the variable to the contents of the given file.

In addition, variables may be marked as recursive using
tj\_template\_variables\_setRecurse(vars, variable, recurse).  The
substituted text of any recursive variable is itself scanned for
substitutions.  No check is made for infinite loops.  Variables are by
default non-recursive.

Note that variables match to the shortest variable label.  In other
words, variables should not be prefixes of other variables.  E.g., the
variable "X" will always take precedence over "XY" and "XYZ"
variables.

```
  tj_template_variables_setFromString(vars, "X", "mushi");
  tj_template_variables_setFromFile(vars, "MUSHI", "test/mushi2");
  tj_template_variables_setRecurse(vars, "MUSHI", 1);
```

All of the setFrom functions return 1 on success, 0 otherwise.  Any
previous buffer contents are effectively lost, even on failure.


### Applying Substitutions ###

Finally, substitutions may be applied to a given input template buffer
to produce the expanded output in another buffer using the
tj\_template\_variables\_apply(vars, target, src) function.  A 1 is
returned on success, and 0 otherwise.

The substitution process goes through the given source template scanning for $X, where X is any variable from a substitution defined in the given tj\_template\_variables object, and replaces it with the substitution value.  There is currently no mechanism for escaping variables, though this could be added easily.  Unrecognized variables are simply left as-is.

```
  vars = tj_template_variables_create();

  tj_buffer_reset(target);
  tj_buffer_reset(src);
  tj_buffer_appendString(src, "$MUSHI");

  tj_template_variables_setFromString(vars, "X", "mushi");
  tj_template_variables_setFromFile(vars, "MUSHI", "test/mushi2");
  tj_template_variables_setRecurse(vars, "MUSHI", 1);

  tj_template_variables_apply(vars, target, src);

  if (strcmp(tj_buffer_getAsString(target), "mushi mushi mushi")) {
    FAIL("Did not get expected accumulated string.");
  }

  tj_template_variables_finalize(vars);
```

### Threads ###

tj\_template\_variables operations are not thread safe, so that mutexing
costs aren't incurred for applications that don't need it. However,
they don't use any non-stack data outside of the tj\_template\_variables
object; if the tj\_template\_variables object is locked by the calling
code, operations using that object are thread safe.


### Logging ###

tj\_template has a simple logging and error reporting mechanism built
in.  If NDEBUG is defined, logging is removed though error reporting
still happens.  This is a compile time switch, so there is no logging
performance cost if it is disabled.  By default the buffer prints log
messages to stdout and error messages to stderr.

This behavior can be changed by redefining several macros when
compiling tj\_template.c (e.g., via -D).

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