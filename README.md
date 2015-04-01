Intermediate Type Language
==========================

Intermediate Type Language (ITL) is an attempt to provide a common
type system for serialization schemes in a machine-friendly format.

# Problem
A common practice for applications that transmit or store data is to
define the structure of the data in a neutral representation and then
generate types that provide a native representation of the data in a
particular programming language and code that can serialize and
deserialize those types.

There are three common problems with this approach.  First, the
neutral representation used to describe the data is often too heavy
for automated translation.  For example, IDL has a number of
user-friendly features that are not machine friendly.  Second, the
type system is often coupled to the serialization scheme.
Translating between serialization schemes leads to an N^2 problem of
matching types in different systems.  Third, serialization often
assumes that the source or target is a native object in a programming
language.  That is, the language-neutral type has a corresponding
concrete type in a programming language and the goal is to serialize
to and from a value of the concrete type.  This prevents potential
optimization where the data is left in a serialized form and
selectively deserialized as needed.

# Use Case

The primary use case for ITL is the development of translators that
convert from one serialization scheme to another.  The user provides a
description of the incoming/outgoing data using ITL.  The translator
uses the ITL description of the data to perform the appropriate
translation.  The translator may interpret the ITL at run-time or the
translator may be generated from ITL.

# Design Goals

1. General - ITL should support common types found in existing
   infrastructure such as IDL, FAST, Avro, Google Protocol Buffers,
   Thrift, etc.
2. Machine-friendly - ITL should be easy to generate, easy to parse,
   and easy to use once parsed.
3. Extensible - ITL should provide a means of annotating types with
   their intended use and external encoding-specific details, e.g., delta
   compression.

ITL is descriptive and not prescriptive.  The types that can be
described with ITL may be a subset or superset of the types that can
be described in another language.  If a tool cannot describe a type in
ITL, then ITL should not be used (and the user should be informed).
If a serializer or deserializer is given a type that cannot be
represented in that serialization scheme, then an appropriate failure
mode should be adopted.

# Types

- byte - An uninterpreted octet.
- bool - Represents a Boolean value (true or false).
- int - Represents an integral number.  An int has an encoding, a size
  (optional), and a flag indicating if the int is unsigned (optional,
  default false).  The size indicates the number of bytes required to
  represent a value of this type for a fixed-length encoding.
- float - Represents a floating-point number.  A float has an encoding
  and an optional size that indicates the number of bytes required to
  represent a value of this type for a fixed-length encoding.
- fixed - Represents a fixed-point number.  A fixed has an encoding, a
  total number of digits, a scale that indicates the number of digits
  after the decimal point, and an optional size that indicates the
  number of bytes required to represent a value of this type for a
  fixed-length encoding.
- rune - Represents a glyph in a written language.  A rune has an
  encoding and an optional size that indicates the
  number of bytes required to represent a value of this type for a
  fixed-length encoding.
- string - Represents a sequence of runes.  A string has an encoding,
  an optional size that indicates the number of runes in the string,
  and an optional capacity that indicates the maximum number of runes
  in the string.  If the runes have a fixed-width encoding, then size
  and capacity can be used to pre-allocate buffers.  It is a semantic
  error if size is greater than capacity.  The interpretation of
  capacity when size is present is deferred to the implementation.
- bitset - Represents a finite set of zero-sized objects.  A bitset
  has a size indicating the number of bytes required to represent the
  universe and a set of named values which must be unsigned integers.
  The names and values should be unique.  The universal is the union
  of values which may be computed using bit-wise OR.  A bitset value
  should be a subset of the universe and may be checked using bit-wise
  AND against the universe.
- enum - Represents a set of disjoint values.  An enum has a type that
  describes the representation of the enum and a set of named values.
  Each value must correspond to the type of the enum.  The names and
  values should be unique.
- sequence - Represents a homogenous sequence of values of a given
  type.  A sequence has an optional size that indicates the
  number of elements in the sequence and an optional capacity that
  indicates the maximum number of elements in the sequence.  If the elements
  have a fixed size, then size and capacity can be used to
  pre-allocate buffers.  It is a semantic error if size is greater
  than capacity.  The interpretation of capacity when size is present
  is deferred to the implementation.
- record - A record represents a potentially heterogeneous sequence of
  named values.  A record is defined by a list of fields.  Each field
  has a name, a type, and an optional flag indicating if the field is
  optional.  The name of each field must be unique.
- union - A union represents a value from a finite set of types.  A
  union has a discriminator type that is used to determine the actual
  type, a set of fields, and a default which indicates the default
  type.  A union field has a name, a type, a set of values
  corresponding to the discriminator type, and an optional flag
  indicating if the field is optional.  The name of each field must be
  unique.  The pair-wise intersection of union field discriminator
  values must be disjoint.

# Standard Encodings

The size parameter for ints, floats, fixed, and runes is required when
specifying a fixed-length encoding.

## Int Encodings

- "2c" - 2's complement in machine format (fixed-length)

## Float Encodings

- "754b" - IEEE754 binary (fixed-length)

## Fixed Encodings

- "bcd" - binary-coded decimal, one digit per byte (fixed-length)
- "pbcd" - packed binary-coded decimal, two digits per byte (fixed-length)

## Rune/String Encodings

- "ascii" (fixed-length)
- "utf8"

# Annotations

Annotations provide a way to capture semantics about encoded data that
govern its use.  To illustrate
the first, consider the problem of serializing a set.  A serialized
set looks like a sequence.  However, when deserializing, the
translator should attempt to restore set semantics by using an
appropriate data type.  In this case, the sequence should
be annotated as a set.  Annotations also provide a way to record
details related to a particular encoding.  For example, FAST delta
compression assumes a know starting value and then sends updates to
that value.  In this case, the field containing the value should be
annotated with delta compression so that translators will know (and
can take advantage of) this fact.

Annotations are a set of key/value pairs where each key corresponds a
system.  For the set example, the key may be "semantic" and the value
may be { "preferredDataType" : "set" }.  For the delta compression
example, the key may be "FAST" and the value may be { "compression" :
"delta" }.  Value nesting is allowed to facilitate the creation of
ontologies for different systems.

# Implementation

ITL is written using JSON to achieve machine friendliness.  ITL
presents a self-contained representation of types.  There is no
facility from importing types from external resources.  There is no
direct support for inheritance.

# Grammar

The grammar is presented as a JSON/BNF hybrid.  Non-terminals are
capitalized (Root) and non-terminals are lower-case (boolean).
Terminals refer to JSON values with the same name.  The terminal
"value" represents any JSON value.  The construct ( ... )? represents
an optional group.

```
Root:
  { "types" : [ TypeDef ] }

TypeDef:
  { "name" : string, "kind" : "byte" }
| { "name" : string, "kind" : "bool" }
| { "name" : string, "kind" : "int", "encoding" : string, ("size" : integer)?, ("unsigned" : boolean)?,  }
| { "name" : string, "kind" : "float", "encoding" : string, ("size" : integer)? }
| { "name" : string, "kind" : "fixed", "encoding" : string, "digits" : integer, "scale" : integer, ("size" : integer)? }
| { "name" : string, "kind" : "rune", "encoding" : string, ("size" : integer)? }
| { "name" : string, "kind" : "string", "encoding" : string, ("size" : integer)?, ("capacity" : integer)? }
| { "name" : string, "kind" : "bitset", "size" : integer, "values" : [ NameValuePair ] }
| { "name" : string, "kind" : "enum", "type" : Type, "values" : [ NameValuePair ] }
| { "name" : string, "kind" : "sequence", "type" : Type, ("size" : integer)?, ("capacity" : integer)? }
| { "name" : string, "kind" : "record", "fields" : [ Field ] }
| { "name" : string, "kind" : "union", "discriminator" : Type, "elements" : [ UnionField ], ("default" : integer)? }

Type:
  string
| TypeDef

NameValuePair:
  { "name" : string, "value" : value }

Field:
  { "name" : string, "type" : Type, ("optional" : boolean)? }

UnionField:
  { "name" : string, "type" : Type, "discriminator_values" : [ JSONValue ], "("optional" : boolean)? }
```

Every JSON Object ({ ... }) has an optional note field ("note" : {
... }) for annotating the field, type, etc.
