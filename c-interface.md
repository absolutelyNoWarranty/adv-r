# R-C interface

This is a opinionated translation of section 5 ("System and foreign language interfaces") of [Writing R extensions](http://cran.r-project.org/doc/manuals/R-exts.html), focussing on best practices and use with `devtools`, not documenting things that used to be a good dea. This means it does not cover:

* The `.C` interface
* The old api defined in `Rdefines.h`

It focusses mainly on section 5.9, "Handling R objects in C".

The main point of this guide is to help you read and understand R's C source code. It will also help you write C code, but we generally recommend using Rcpp, because of the additional syntax sugar it provides.

A substantial amount of R is implemented using the functions and macros
described here, so the R source code provides a rich source of examples
and "how to do it": do make use of the source code for inspirational
examples.

## Finding the C source code for a function

`.Primitive`

`.Internal`

## Calling C functions from R

Calling C functions from R involves two parts:

* A C function
* An R function that calls the C function using `.Call`

The first argument to `.Call` is the name of the function, followed by its arguments (to a maximum of 65). Let's pretend we want to write a C function to add two numbers together.  The R side of the interface might look like:
  
    add <- function(a, b) {
      stopifnot(is.numeric(a), is.numeric(b))
      .Call("add", a, b)
    }

And the C side might look like:

    #include <R.h>
    #include <Rinternals.h>

    SEXP add(SEXP a, SEXP b) {
      SEXP result;

      PROTECT(result = allocVector(REALSXP, 1));
      REAL[result] = asReal(a) + asReal(b);
      UNPROTECT();

      return(REAL);
    }

This illustrates most of the important C functions and macros you need to know about: creating new R vectors, coercing input arguments to the appropriate type and dealing with garbage collecting.

The rest of this chapter explains in more detail how these (and other important functions) work; providing a guide to the C function you use to manipulate R data structures. These are in defined header file `Rinternals.h`, which you can easily find and show from within R:

    rinternals <- file.path(R.home(), "include", "Rinternals.h")
    file.show(rinternals)

### Inline

How to use the inline package.

## Basic data structures

All the R objects are stored in a common datatype, the `SEXP`. (Technically, this is a pointer to a structure with typedef `SEXPREC`). A `SEXP` is a variant type, with subtypes for all of the common R data structures. The most important types are:

* `REALSXP`: numeric vector
* `INTSXP`: integer vector
* `LGLSXP`: logical vector
* `STRSXP`: character vector
* `VECSXP`: a list
* `CLOSXP`: a function or function closure
* `ENVSXP`: a environment
* `NILSXP`: `NULL`
 
There are also `SEXP`s for less used object types:

* `CPLXSXP`: complex vectors
* `LISTSXP`: a "pair" list, in R used only for function arguments
* `DOTSXP`: a object used to reprsent '...'
* `SYMSXP`: name/symbol

And for internal objects, objects that are usually only created by C functions, not R functions:

* `LANGSXP`: language constructs
* `CHARSXP`: "scalar" string type
* `PROMSXP`: promises, lazily evaluated function arguments
* `EXPRSXP`: expressions

Generally you should avoid returning these types of object to R.

__Beware:__ At the C level, R's lists are `VECSXP`s, pairlists are `LISTSXP`s. 

### Character vectors

R character vectors are stored as `STRSXP`s, a vector type like `VECSXP`
where every element is of type `CHARSXP`. The `CHARSXP` elements of
`STRSXP`s are accessed using `STRING_ELT` and `SET_STRING_ELT`.

`CHARSXP`s are read-only objects and must never be modified. In
particular, the C-style string contained in a `CHARSXP` should be
treated as read-only and for this reason the `CHAR` function used to
access the character data of a `CHARSXP` returns `(const char *)`. Since
`CHARSXP`s are immutable, the same `CHARSXP` can be shared by any
`STRSXP` needing an element representing the same string. R maintains a
global cache of `CHARSXP`s so that there is only ever one `CHARSXP`
representing a given string in memory.

You can obtain a `CHARSXP` by calling `mkChar` and providing a
C string. This function will return a pre-existing
`CHARSXP` if one with a matching string already exists, otherwise it
will create a new one and add it to the cache before returning it to
you.

`CHARSXP`s can be marked as coming from a known encoding (Latin-1 or
UTF-8). This is mainly intended for human-readable output, and most
packages can just treat such `CHARSXP`s as a whole. However, if they
need to be interpreted as characters or output at C level then it would
normally be correct to ensure that they are converted to the encoding of
the current locale: this can be done by accessing the data in the
`CHARSXP` by `translateChar` rather than by `CHAR`. There is a similar function `translateCharUTF8` which converts to UTF-8:
this has the advantage that a faithful translation is almost always
possible.

## Checking types

Unless you are very sure about the type of the arguments, the code
should check the data types. You can either do this in R, or in C code.  In R, this is often most easily accomplished with `stopifnot` and `is.numeric`, `is.integer`, `is.character` etc, or by coercing the inputs to the correct type. In C, you have two options, either using `TYPEOF` or one of the built in helper functions.

`TYPEOF` returns the `SEXPTYPE` of the incoming SEXP:

      SEXP is_numeric(SEXP x) {
        TYPEOF(x) == REALSXP
      }

These all return 0 for FALSE and 1 for TRUE.

Atomic vectors:

* isInteger: an integer vector (that isn't a factor)
* isReal
* isComplex
* isLogical
* isString
* isNumeric: integer, logical, real
* isNumber: numeric + complex numbers
* isVectorAtomic: atomic vectors (logical, interger, numeric, complex, string, raw)

* isArray: vector with dimension attribute
* isMatrix: vector with dimension attribute of length 2

* isEnvironment
* isExpression
* isFactor: 
* isFunction: function: a closure, a primitive, or an internal function
* isList: __pair__list?
* isNewList: list
* isSymbol
* isNull
* isObject
* isVector: atomic vectors + lists and expressions

Note that some of these functions behave differently to the R-level functions with similar names. For example `isVector` is true for any atomic vector type, lists and expression, where `is.vector` is returns `TRUE` only if its input has no attributes apart from names.

## Coercing types

Note that these coercion functions are *not* the same as calling
`as.numeric` (and so on) in R code, as they do not dispatch on the class
of the object. Thus it is normally preferable to do the coercion in the
calling R code.

Coercing to R objects:

* coerceVector  .  This will return an error if the `SEXP` can not be converted to the desired type.

You will always need to run `coerceVector` inside `PROTECT`, as described in the garbage collection section below.

Coercing to C scalars:

* asLogical
* asInteger
* asReal
* asComplex
* asChar
* asCharacterFactor

Coercing to C vectors:

LOGICAL(x)
INTEGER(x)
RAW(x)    
COMPLEX(x)
REAL(x)

There is no convenience function to extract a C character vector from a `STRSXP`.  You can extract a single `CHARSXP` from a `STRSXP` with `STRING_ELT(x,i)`. You can extract the C string from the `CHARSXP` with CHAR(x)`

## Creating new objects

allocVector(SEXPTYPE type, R_xlen_t length)
allocMatrix(SEXPTYPE mode, int nrow, int ncol)
alloc3DArray(SEXPTYPE mode, int nrow, int ncol, int nface)
allocArray(SEXPTYPE mode, SEXP dims)
allocList(int n)

Convenience functions for creating a single element:

* ScalarLogical(int x)
* ScalarInteger(int x)
* ScalarReal(double x)
* ScalarComplex(Rcomplex x); Rcomplex cmp = {0.1, 0.2};
* ScalarRaw(Rbyte x)
* mkString("mystring")
* ScalarString(mkChar("mystring"))

* mkNamed: to create `list(xi=, yi=, zi=)`

        const char *nms[] = {"xi", "yi", "zi", ""};
        mkNamed(VECSXP, nms);


If storage is required for C objects during the calculations this is
best allocating by calling `R_alloc`; see [Memory
allocation](http://cran.r-project.org/doc/manuals/R-exts.html#Memory-allocation).
All of these memory allocation routines do their own error-checking, so
the programmer may assume that they will raise an error if the memory cannot be allocated.

### Garbage collection

If you create an R object in your C code, you must tell R that you are
using the object by using the `PROTECT` macro on a pointer to the
object. This tells R that the object is in use so it is not destroyed
during garbage collection.  This is also protects all objects pointed to in the corresponding `SEXPREC`, for example all elements of a
protected list are automatically protected.

You are responsible for housekeeping the calls to `PROTECT`. There is a corresponding macro `UNPROTECT` that takes as
argument an `int` giving the number of objects to unprotect when they
are no longer needed. The protection mechanism is stack-based, so
`UNPROTECT(`n`)` unprotects the last n objects which were protected. The
calls to `PROTECT` and `UNPROTECT` must balance when the user's code
returns. R will warn about `"stack imbalance in .Call"` (or `.External`)
if they don't match.

Here is a small example of creating an R numeric vector in C code. First
we use the macros in Rinternals.h:

   #include <R.h>
   #include <Rinternals.h>
   
    SEXP ab;
      ....
    PROTECT(ab = allocVector(REALSXP, 2));
    REAL(ab)[0] = 123.45;
    REAL(ab)[1] = 67.89;
    UNPROTECT(1);

You might wonder how the R object could possibly get removed
during those manipulations, as it is just our C code that is running. As
it happens, we can do without the protection in this example, but in
general we do not know (nor want to know) what is hiding behind the R
macros and functions we use, and any of them might cause memory to be
allocated, hence garbage collection and hence our object `ab` to be
removed. It is usually wise to err on the side of caution and assume
that any of the R macros and functions might remove the object.

Protection is not needed for objects which R already knows are in use.
In particular, this applies to function arguments.

Other specialised forms of `PROTECT` and `UNPROTECT` are needed in some circumstances: `UNPROTECT_PTR(`s`)` that unprotects the
object pointed to by the `SEXP` s, `PROTECT_WITH_INDEX` saves an index of the protection location that can be used to replace the protected value using `REPROTECT`.  Consult R externals for more details.

### Getting and setting attributes

Many R objects have attributes: some of the most useful are classes and
the `dim` and `dimnames` that mark objects as matrices or arrays. It can
also be helpful to work with the `names` attribute of vectors.

To illustrate this, let us write code to take the outer product of two
vectors (which `outer` and `%o%` already do). As usual the R code is
simple

   out <- function(x, y) {
    storage.mode(x) <- storage.mode(y) <- "double"
    .Call("out", x, y)
   }

where we expect `x` and `y` to be numeric vectors (possibly integer),
possibly with names. This time we do the coercion in the calling R code.

However, we would like to set the `dimnames` of the result. Although
`allocMatrix` provides a short cut, we will show how to set the `dim`
attribute directly.

   #include <R.h>
   #include <Rinternals.h>
   
   SEXP out(SEXP x, SEXP y)
   {
    R_len_t i, j, nx = length(x), ny = length(y);
    double tmp, *rx = REAL(x), *ry = REAL(y), *rans;
    SEXP ans, dim, dimnames;
   
    PROTECT(ans = allocVector(REALSXP, nx*ny));
    rans = REAL(ans);
    for(i = 0; i < nx; i++) {
    tmp = rx[i];
    for(j = 0; j < ny; j++)
      rans[i + nx*j] = tmp * ry[j];
    }
   
    PROTECT(dim = allocVector(INTSXP, 2));
    INTEGER(dim)[0] = nx; INTEGER(dim)[1] = ny;
    setAttrib(ans, R_DimSymbol, dim);
   
    PROTECT(dimnames = allocVector(VECSXP, 2));
    SET_VECTOR_ELT(dimnames, 0, getAttrib(x, R_NamesSymbol));
    SET_VECTOR_ELT(dimnames, 1, getAttrib(y, R_NamesSymbol));
    setAttrib(ans, R_DimNamesSymbol, dimnames);
   
    UNPROTECT(3);
    return(ans);
   }

This example introduces several new features. The `getAttrib` and
`setAttrib` functions get and set individual attributes. Their second
argument is a `SEXP` defining the name in the symbol table of the
attribute we want; these and many such symbols are defined in the header
file Rinternals.h.

There are shortcuts here too: the functions `namesgets`, `dimgets` and
`dimnamesgets` are the internal versions of the default methods of
`names<-`, `dim<-` and `dimnames<-` (for vectors and arrays).

### Symbols

    R_ClassSymbol;      /* "class" */
    R_DimNamesSymbol;   /* "dimnames" */
    R_DimSymbol;      /* "dim" */
    R_NamesSymbol;      /* "names" */
    R_LevelsSymbol;     /* "levels" */

    R_Bracket2Symbol;   /* "[[" */
    R_BracketSymbol;    /* "[" */
    R_BraceSymbol;      /* "{" */
    R_DollarSymbol;     /* "$" */
    R_DotsSymbol;     /* "..." */
    R_DropSymbol;     /* "drop" */
    R_ModeSymbol;     /* "mode" */
    R_NameSymbol;     /* "name" */
    R_NaRmSymbol;     /* "na.rm" */
    R_PackageSymbol;    /* "package" */
    R_QuoteSymbol;      /* "quote" */
    R_RowNamesSymbol;   /* "row.names" */
    R_SourceSymbol;     /* "source" */

What happens if we want to add an attribute that is not pre-defined? We
need to add a symbol for it *via* a call to `install`. Suppose for
illustration we wanted to add an attribute `"version"` with value `3.0`.
We could use

    SEXP version;
    PROTECT(version = allocVector(REALSXP, 1));
    REAL(version)[0] = 3.0;
    setAttrib(ans, install("version"), version);
    UNPROTECT(1);

Using `install` when it is not needed is harmless and provides a simple
way to retrieve the symbol from the symbol table if it is already
installed. 

### S3 Classes

In R the class is just the attribute named `"class"` so it can be
handled as such, but there is a shortcut `classgets`. Suppose we want to
give the return value in our example the class `"mat"`. We can use

   #include <R.h>
   #include <Rdefines.h>
      ....
    SEXP ans, dim, dimnames, class;
      ....
    PROTECT(class = mkString("mat"));
    classgets(ans, class);
    UNPROTECT(4);
    return(ans);
   }

### Lists

Some care is needed with lists, as R moved early on from using LISP-like
lists (now called “pairlists”) to S-like generic vectors. As a result,
the appropriate test for an object of mode `list` is `isNewList`, and we
need `allocVector(VECSXP,` n).

List elements can be retrieved or set by direct access to the elements
of the generic vector. Suppose we have a list object

   a <- list(f = 1, g = 2, h = 3)

Then we can access `a$g` as `a[[2]]` by

    double g;
      ....
    g = REAL(VECTOR_ELT(a, 1))[0];

This can rapidly become tedious, and the function like the following is very useful:

    SEXP getListElement(SEXP list, const char *str) {
      SEXP elmt = R_NilValue;
      SEXP names = getAttrib(list, R_NamesSymbol);

      for (R_len_t i = 0; i < length(list); i++) {
        if (strcmp(CHAR(STRING_ELT(names, i)), str) == 0) {
          elmt = VECTOR_ELT(list, i);
          break;
        }
      }
      return elmt;
    }

and enables us to say

     double g;
     g = REAL(getListElement(a, "g"))[0];

### Missing and non-finite values

R's `NA` is a subtype of `NaN` so IEC60559 arithmetic will handle them
correctly.  However, it is unwise to depend on such details, and is better to deal with missings explicitly.

* In doubles, use the `ISNA` macro, `ISNAN`, or `R_FINITE` macros to check for missing, NaN or non-finite values.  Use the constants `NA_REAL`, `R_NaN`, `R_PosInf` and `R_NegInf` to set those values

* Integers, compare to and set with `NA_INTEGER`
* Logicals, compare to and set with `NA_LOGICAL`
* String, compare to and set with `NA_STRING`

## Working with pairlists and language objects

There are a series of small macros/functions to help construct pairlists
and language objects (whose internal structures just differ by
`SEXPTYPE`). Function `CONS(u, v)` is the basic building block: is
constructs a pairlist from `u` followed by `v` (which is a pairlist or
`R_NilValue`). `LCONS` is a variant that constructs a language object.

Functions `list1` to `list5` construct a pairlist from one to five
items, and `lang1` to `lang6` do the same for a language object (a
function to call plus zero to five arguments). 

Functions `elt` and `lastElt` find the ith element and the last element of a pairlist, and `nthcdr` returns a pointer to the nth position in the pairlist (whose `CAR` is the nth item).

## Modifying inputs

When assignments are done in R such as

   x <- 1:10
   y <- x

the named object is not necessarily copied, so after those two
assignments `y` and `x` are bound to the same `SEXPREC` (the structure a
`SEXP` points to). This means that any code which alters one of them has
to make a copy before modifying the copy if the usual R semantics are to
apply. Note that whereas `.C` and `.Fortran` do copy their arguments
(unless the dangerous `dup = FALSE` is used), `.Call` and `.External` do
not. So `duplicate` is commonly called on arguments to `.Call` before
modifying them.

However, at least some of this copying is unneeded. In the first
assignment shown, `x <- 1:10`, R first creates an object with value
`1:10` and then assigns it to `x` but if `x` is modified no copy is
necessary as the temporary object with value `1:10` cannot be referred
to again. R distinguishes between named and unnamed objects *via* a
field in a `SEXPREC` that can be accessed *via* the macros `NAMED` and
`SET_NAMED`. This can take values

* `0`:  The object is not bound to any symbol
* `1`:  The object has been bound to exactly one symbol
* `2`:  The object has potentially been bound to two or more symbols, and
 one should act as if another variable is currently bound to this
 value.

Note the past tenses: R does not do full reference counting and there
may currently be fewer bindings.

It is safe to modify the value of any `SEXP` for which `NAMED(foo)` is
zero, and if `NAMED(foo)` is two, the value should be duplicated (*via*
a call to `duplicate`) before any modification. Note that it is the
responsibility of the author of the code making the modification to do
the duplication, even if it is `x` whose value is being modified after
`y <- x`.

The case `NAMED(foo) == 1` allows some optimization, but it can be
ignored (and duplication done whenever `NAMED(foo) > 0`). (This
optimization is not currently usable in user code.) It is intended for
use within replacement functions. Suppose we used

   x <- 1:10
   foo(x) <- 3

which is computed as

   x <- 1:10
   x <- "foo<-"(x, 3)

Then inside `"foo<-"` the object pointing to the current value of `x`
will have `NAMED(foo)` as one, and it would be safe to modify it as the
only symbol bound to it is `x` and that will be rebound immediately.
(Provided the remaining code in `"foo<-"` make no reference to `x`, and
no one is going to attempt a direct call such as `y <- "foo<-"(x)`.)

Currently all arguments to a `.Call` call will have `NAMED` set to 2,
and so users must assume that they need to be duplicated before
alteration.


## Other functions

* LENGTH

## `.External`

An alternative to using `.Call` is to use `.External`.  It is used almost identically, except that the C function will recieve a single arugment containing a `LISTSXP`, a pairlist from which the
arguments can be extracted.  For example, if we used `.External`, the add function would become.

## Loading your code

The easiest way to get up and running with compiled C code is to use devtools.  You'll need to put your code in a package structure, which means:

* R files in a `R/` directory
* C files in `src/` directory
* A `DESCRIPTION` in the main directory
* A `NAMESPACE` file containing `useDynLib(packagename)`

Then running `load_all` will automatically compile and reload the code in your package.
