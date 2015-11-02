This is a simple FFI for Emacs.  It is based on libffi and relies on
the dynamic modules work to be loaded into Emacs.

This is very preliminary.  I'd appreciate your feedback, either via
email or issues on github.

# Types

Currently the library only supports primitive types for arguments and
return types.  Structure types will be supported some day.

Primitive types are described using keywords:

* `:void`. The void type.  This does not make sense as an argument
  type.

* `:int8`, `:uint8`, `:int16`, `:uint16`, `:int32`, `:uint32`,
  `:int64`, `:uint64`: signed or unsigned integers of the indicated size.

* `:float`, `:double`.  Self-explanatory.

* `:uchar`, `:char`, `:ushort`, `:short`, `:uint`, `:int`, `:ulong`,
  `:long`.  Signed or unsigned integers corresponding to the C type of
  the same name.

* `:pointer`.  A C pointer type.  Pointers currently aren't typed, in
  the sense that they aren't differentiated based on what they point
  to.

* `:size_t`, `:ssize_t`, `:bool`.  These correspond to the C type of
  the same name and internally are just aliases for one of the other
  integral types.

# Type Conversions

Currently all type conversions work the same in both directions.

* A function declared with a `:void` return type will always return
  `nil` to Lisp.

* A function returning any integer or character type will return a
  Lisp integer.  Note that this may result in the value being
  truncated; currently there is nothing that can be done about this.

* A C pointer will be returned as a user-pointer (a new Lisp type
  introduced by the dynamic module patch).

# Exported Functions

* `(define-ffi-library SYMBOL NAME)`.  Used to define a function that
  lazily loads a library.  Like:

```
  (define-ffi-library libwhatever "libwhatever")
```

  ffi-module uses libltdl (from libtool), which will automatically
  supply the correct extension if none is specified, so it's generally
  best to leave off the `.so`.

* `(define-ffi-function NAME C-NAME RETURN-TYPE ARG-TYPES LIBRARY)`.

  NAME is the symbol to define.  C-NAME is a string, the name of the
  underlying C function.

  RETURN-TYPE describes the return type of the C function and
  ARG-TYPES describes the argument types.  ARG-TYPES may be a vector
  or a list.

  LIBRARY is the library where the C function should be found.  This
  is just a symbol, most usually defined with `define-ffi-library`.

  `define-ffi-function` defines a new Lisp function.  It takes as many
  arguments as were in ARG-TYPES.  (While there is internal support
  for varargs functions, it is not exposed by `define-ffi-function`.)

* `(ffi-make-closure CIF FUNCTION)`.  Make a C pointer to the Lisp
  function.  This pointer can then be passed to C functions that need
  a pointer-to-function argument, and FUNCTION will be called with
  whatever arguments are passed in.

  CIF is a CIF as returned by `ffi--prep-cif`.  It describes the
  function's type (as needed by C).

  This returns a C function pointer, wrapped in the usual way as a
  user-pointer object.

* `(ffi-get-c-string POINTER)`.  Assume the pointer points to a C
  string, and return a Lisp string with those contents.

# Internal Functions

* `(ffi--dlopen STR)`.  A simple wrapper for `dlopen` (actually
  `lt_dlopen`).  This returns the library handle, a C pointer.

* `(ffi--dlsym STR HANDLE)`.  A simple wrapper for `dlsym` (actually
  `lt_dlsym`).  This finds the C symbol named STR in a library.
  HANDLE is a library handle, as returned by a function defined by
  `define-ffi-library`.

  This returns a C pointer to the indicated symbol, or `nil` if it
  can't be found.  These pointers need not be freed.

* `(ffi--prep-cif RETURN-TYPE ARG-TYPES &optional N-FIXED-ARGS)`.  A
  simple wrapper for libffi's `ffi_prep_cif` and `ffi_prep_cif_var`.
  RETURN-TYPE is the return type; ARG-TYPES is a vector of argument
  types.  If given, N-FIXED-ARGS is an integer holding the number of
  fixed args.  Its presence, even if 0, means that a varargs call is
  being made.  This function returns a C pointer wrapped in a Lisp
  object; the garbage collector will handle any needed finalization.

* `(ffi--call CIF FUNCTION &rest ARGS)`.  Make an FFI call.

  CIF is the return from `ffi--prep-cif`.

  FUNCTION is a C pointer to the function to call, normally from
  `ffi--dlsym`.

  ARGS are the arguments to pass to FUNCTION.

* `(ffi--mem-ref POINTER SIZE)`.  Read SIZE bytes of memory starting
  at POINTER.  Currently, the bytes are returned in a vector, one
  element per byte.  (This might change if there is a handy way to
  make a unibyte string from the module API.)  This can be useful in
  conjunction with the `bindat` package for unpacking C structures.

* `(ffi--mem-set POINTER VECTOR)`.  Copy the contents of VECTOR to the
  memory at POINTER.  Each element of the vector must be an integer;
  the low eight bits are used as the bytes to write to successive
  memory locations.  Note that this might write partially before
  failing if some element of the vector is not an integer.

* `(ffi--pointer+ POINTER NUMBER)`.  Pointer math in Lisp.

# Partial To-Do List

* Add nice support for varargs calls.  The main issue is that the
  types have to be described at each call, and it would be good to
  have an easy way to describe this in Lisp.

* Structure types in arguments and returns.

* Array types.

* Union types.

* Add a `:c-string` type; for arguments this would be `const char *`
  and for results it would automatically wrap as a new C string.  One
  issue is for results you may want to automatically free the returned
  pointer, so this would require some extra info.

* Typed pointers.

* A way to deal with integer truncation.  Or just add bignums to
  Emacs.
