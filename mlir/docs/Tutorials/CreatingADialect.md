# Creating a Dialect

[TOC]

Public dialects are typically separated into at least 3 directories:
* mlir/include/mlir/Dialect/Foo   (for public include files)
* mlir/lib/Dialect/Foo            (for sources)
* mlir/lib/Dialect/Foo/IR         (for operations)
* mlir/lib/Dialect/Foo/Transforms (for transforms)
* mlir/test/Dialect/Foo           (for tests)

Along with other public headers, the 'include' directory contains a
TableGen file in the [ODS format](OpDefinitions.md), describing the
operations in the dialect.  This is used to generate operation
declarations (FooOps.h.inc) and definitions (FooOps.cpp.inc) and
operation interface declarations (FooOpsInterfaces.h.inc) and
definitions (FooOpsInterfaces.cpp.inc).

The 'IR' directory typically contains implementations of functions for
the dialect which are not automatically generated by ODS.  These are
typically defined in FooDialect.cpp, which includes FooOps.cpp.inc and
FooOpsInterfaces.h.inc.

The 'Transforms' directory contains rewrite rules for the dialect,
typically described in TableGen file using the [DDR
format](DeclarativeRewrites.md).

Note that dialect names should not generally be suffixed with “Ops”,
although some files pertaining to the operations of a dialect (e.g.
FooOps.cpp) might be.

## CMake best practices

### TableGen Targets

Operations in dialects are typically declared using the ODS format in
tablegen in a file FooOps.td.  This file forms the core of a dialect and
is declared using add_mlir_dialect().

```cmake

add_mlir_dialect(FooOps foo)
add_mlir_doc(FooOps -gen-dialect-doc FooDialect Dialects/)

```

This generates the correct rules to run mlir-tblgen, along with a
'MLIRFooOpsIncGen' target which can be used to declare dependencies.

Dialect transformations are typically declared in a file FooTransforms.td.
Targets for TableGen are described in typical llvm fashion.
```cmake
set(LLVM_TARGET_DEFINITIONS FooTransforms.td)
mlir_tablegen(FooTransforms.h.inc -gen-rewriters)
add_public_tablegen_target(MLIRFooTransformIncGen)
```

The result is another 'IncGen' target, which runs mlir-tblgen.

### Library Targets

Dialects may have multiple libraries.  Each library is typically
declared with add_mlir_dialect_library().  Dialect libraries often
depend on the generation of header files from TableGen (specified
using the DEPENDS keyword).  Dialect libraries may also depend on
other dialect libraries.  Typically this dependence is declared using
target_link_libraries() and the PUBLIC keyword.  For instance:

```cmake

add_mlir_dialect_library(FooOps
	DEPENDS
	MLIRFooOpsIncGen
	MLIRFooTransformsIncGen
	)
target_link_libraries(FooOps
	PUBLIC
	BarOps
	<some-other-library>
	)

```

add_mlir_dialect_library() is a thin wrapper around add_llvm_library()
which collects a list of all the dialect libraries.  This list is
often useful for linking tools (e.g. mlir-opt) which should have
access to all dialects.  This list is also linked into libMLIR.so.
The list can be retrieved from the MLIR_DIALECT_LIBS global property:

```cmake

get_property(dialect_libs GLOBAL PROPERTY MLIR_DIALECT_LIBS)

```

Note that although the Bar dialect also uses TableGen to declare its
operations, it is not necessary to explicitly depend on the
corresponding IncGen targets.  The PUBLIC link dependency is
sufficient.  Also note that we avoid using add_dependencies
explicitly, since the dependencies need to be available to the
underlying add_llvm_library() call, allowing it to correctly create
new targets with the same sources.



# Dialect Conversions

Conversions from “X” to “Y” live in mlir/include/mlir/Conversion/XToY,
mlir/lib/Conversion/XToY and mlir/test/Conversion/XToY, respectively.

Default file names for conversion should omit “Convert” from their
name, e.g. lib/VectorToLLVM/VectorToLLVM.cpp.

Conversion passes should live separately from conversions themselves
for convenience of users that only care about a pass and not about its
implementation with patterns or other infrastructure. For example
include/mlir/VectorToLLVM/VectorToLLVMPass.h.

Common conversion functionality from or to dialect “X” that does not
belong to the dialect definition can be located in
mlir/lib/Conversion/XCommon, for example
mlir/lib/Conversion/GPUCommon.

## CMake best practices

Each conversion typically exists in a separate library, declared with
add_mlir_conversion_library().  Conversion libraries typically depend
on their source and target dialects, but may also depend on other
dialects (e.g. MLIRStandard).  Typically this dependence is specified
using target_link_libraries() and the PUBLIC keyword.  For instance:

```cmake

add_mlir_conversion_library(MLIRBarToFoo
	BarToFoo.cpp

        ADDITIONAL_HEADER_DIRS
        ${MLIR_MAIN_INCLUDE_DIR}/mlir/Conversion/BarToFoo
	)
target_link_libraries(MLIRBarToFoo
	PUBLIC
	BarOps
	FooOps
	)

```

add_mlir_conversion_library() is a thin wrapper around
add_llvm_library() which collects a list of all the conversion
libraries.  This list is often useful for linking tools
(e.g. mlir-opt) which should have access to all dialects.  This list
is also linked in libMLIR.so.  The list can be retrieved from the
MLIR_CONVERSION_LIBS global property:

```cmake

get_property(dialect_libs GLOBAL PROPERTY MLIR_CONVERSION_LIBS)

```
