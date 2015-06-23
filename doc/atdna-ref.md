---
layout: doc
title: ATDNA Reference
---

For general information on **ATDNA**, please read the 
[overview article](atdna.html).

## Defining a DNA Record ##

`atdna` will only emit code for C++ records (structs, classes, unions) that 
subclass (directly or indirectly) `Athena::io::DNA<Athena::Endian>`. The
DNA base-class is defined in AthenaCore and can be included using
`<Athena/DNA.hpp>`. A master byte-order 
([endianness](https://en.wikipedia.org/wiki/Endianness)) 
must be selected at this time (`Athena::BigEndian` or `Athena::LittleEndian`).

In addition to the inheritance, the `DNA::read()` and `DNA::write()` methods
must be declared within the subclass. A convenience macro `DECL_DNA` is provided
to take care of this automatically.

{% highlight cpp %}
#include <Athena/DNA.hpp>
struct MyFirstDNA : public Athena::io::DNA<Athena::BigEndian>
{
    DECL_DNA
    Value<atUint32> myInt;
};
{% endhighlight %}

## Adding DNA Fields ##

`atdna` uses C++11 
[type alias](http://en.cppreference.com/w/cpp/language/type_alias) 
template specializations to define DNA layouts. These templates are as follows:

### Value ###

{% highlight cpp %}
Value<typename type, Athena::Endian endian = Inherit> myValue;
{% endhighlight %}

Simple numeric primitive type in Athena. Any built-in integer or floating-point
type may be provided for the `type` argument.

Athena uses the following, although atdna will work with built-in C types and
`<stdint.h>` types as well:

| Type       | Description                          |
|------------|--------------------------------------|
| `atInt8`   | Signed 8-bit (`char`)                |
| `atUint8`  | Unsigned 8-bit (`unsigned char`)     |
| `atInt16`  | Signed 16-bit (`short`)              |
| `atUint16` | Unsigned 16-bit (`unsigned short`)   |
| `atInt32`  | Signed 32-bit (`long`)               |
| `atUint32` | Unsigned 32-bit (`unsigned long`)    |
| `atInt64`  | Signed 64-bit (`long long`)          |
| `atUint64` | Unsigned 64-bit (`unsigned long long`) |
| `bool`     | Standard boolean type                |
| `float`    | 32-bit IEEE 754 floating-point       |
| `double`   | 64-bit IEEE 754 floating-point       |
| `atVec3f`  | 3-component 32-bit IEEE 754 floating-point |
| `atVec4f`  | 4-component 32-bit IEEE 754 floating-point |

### Vector ###

{% highlight cpp %}
Vector<typename type, size_t countVar, Athena::Endian endian = Inherit> myVector;
{% endhighlight %}

This is a wrapper template around 
[`std::vector`](http://en.cppreference.com/w/cpp/container/vector) which
represents a 
[contiguous table of records](https://en.wikipedia.org/wiki/Array_data_structure). 
All standard vector operations are available through this type.

The `type` argument determines the *per-element* data type used in the vector.
It must be a type compatible with *Value* or another DNA-record subclass.

The `countVar` argument statically-evaluates the contents of the special macro
`DNA_COUNT`. This macro captures a C expression which is evaluated to determine
how many elements the vector will be filled with.

#### Example ####

{% highlight cpp %}
struct VectorDemo : public Athena::io::DNA<Athena::BigEndian>
{
    DECL_DNA
    Value<atUint32> count;
    
    /* I can also be defined out-of-line */
    struct VectorElement : public Athena::io::DNA<Athena::BigEndian>
    {
        DECL_DNA
        Value<float> val1;
        Value<float> val2;
    };
    Vector<VectorElement, DNA_COUNT(count)> vector;
};
{% endhighlight %}

### Buffer ###

{% highlight cpp %}
Buffer<size_t sizeVar> myBuffer;
{% endhighlight %}

General-purpose data-buffer. The Buffer template is implemented as an `atUint8`
variable-length buffer wrapped in 
[`std::unique_ptr`](http://en.cppreference.com/w/cpp/memory/unique_ptr).

Like Vector, the `sizeVar` works via `DNA_COUNT` to determine how many bytes
long the buffer should read and write.

#### Example ####

{% highlight cpp %}
struct BufferDemo : public Athena::io::DNA<Athena::BigEndian>
{
    DECL_DNA
    Value<atUint32> length;
    Buffer<DNA_COUNT(length)> buffer;
};
{% endhighlight %}


### String ###

{% highlight cpp %}
String<size_t sizeVar = -1> myString;
{% endhighlight %}

General-purpose byte-character string. The String template is implemented as
a [`std::string`](http://en.cppreference.com/w/cpp/string/basic_string) with
all available operations.

By default, strings are assumed to be null-terminated (size -1). 
If the string is actually
fixed-length, the `sizeVar` argument can be supplied with an integer literal
counting the string-buffer's characters. 

### WString ###

{% highlight cpp %}
WString<size_t sizeVar = -1, Athena::Endian endian = Inherit> myWString;
{% endhighlight %}

General-purpose wide-character string. The WString template is implemented as
a [`std::wstring`](http://en.cppreference.com/w/cpp/string/basic_string) with
all available operations.

By default, strings are assumed to be null-terminated (size -1). 
If the string is actually
fixed-length, the `sizeVar` argument can be supplied with an integer literal
counting the string-buffer's characters. 

### UTF8 ###

{% highlight cpp %}
UTF8<size_t sizeVar = -1, Athena::Endian endian = Inherit> myUTF8;
{% endhighlight %}

General-purpose multi-byte-character string. The UTF8 template is implemented as
a [`std::string`](http://en.cppreference.com/w/cpp/string/basic_string) with
all available operations.

By default, strings are assumed to be null-terminated (size -1). 
If the string is actually
fixed-length, the `sizeVar` argument can be supplied with an integer literal
counting the string-buffer's bytes. 

## Adding DNA Meta-Fields ##

`atdna` provides more than data fields. DNA can also include meta-directives
for fine-grained control of the stream.

### Seek ###

{% highlight cpp %}
Seek<off_t offset, Athena::SeekOrigin direction> mySeek;
{% endhighlight %}

Not all formats are easily read sequentially from start-to-finish. Sometimes
some random-access is required to handle complex, variable-length formats.

The Seek template is provided to move the stream cursor. It works exactly like
[fseek](https://en.wikipedia.org/wiki/C_file_input/output#fseek), supplied
with a count of bytes and a direction to seek relative to.

The `offset` argument works with the `DNA_COUNT` macro to evaluate a byte-count
expression within the record.

The `direction` argument may be one of the following:

| Athena::SeekOrigin | Description                        | 
|--------------------|------------------------------------|
| `Athena::Begin`    | Offset from start of stream        |
| `Athena::Current`  | Offset from present stream cursor  |
| `Athena::End`      | Offset from end of stream          |

#### Example ####

{% highlight cpp %}
struct SeekDemo : public Athena::io::DNA<Athena::BigEndian>
{
    DECL_DNA
    Value<atUint32> subRecordOffset;
    Seek<DNA_COUNT(subRecordOffset), Athena::Begin> seek1;
    
    /* I start at the absolute subRecordOffset */
    struct SubRecord : public Athena::io::DNA<Athena::BigEndian>
    {
        DECL_DNA
        Value<float> val1;
        Value<float> val2;
    } record;
};
{% endhighlight %}

### Align ###

{% highlight cpp %}
Align<size_t align> myAlign;
{% endhighlight %}

Some data formats are designed to be loaded as a cache-aligned data-blob.
In order to maintain cache-alignment, padding-bytes are commonly inserted to
bring the start of a sub-record to a 4, 8, 16, 32 byte-aligned position in
the stream.

`atdna` provides the Align template to automatically generate the most efficent
cursor-position evaluation; advancing forward to the nearest alignment multiple.

#### Example ####

{% highlight cpp %}
struct AlignDemo : public Athena::io::DNA<Athena::BigEndian>
{
    DECL_DNA
    struct SubRecordOne : public Athena::io::DNA<Athena::BigEndian>
    {
        DECL_DNA
        Value<atUint32> val;
    } one;

    Align<32> align1;

    /* I'm 32-byte aligned!! */
    struct SubRecordTwo : public Athena::io::DNA<Athena::BigEndian>
    {
        DECL_DNA
        Value<atVec3f> val;
    } two;

    Align<32> align2;
    
    /* I'm also 32-byte aligned!! */
    struct SubRecordThree : public Athena::io::DNA<Athena::BigEndian>
    {
        DECL_DNA
        Value<atUint16> val1;
        Value<atUint16> val2;
    } three;    
};
{% endhighlight %}

### Delete ###

{% highlight cpp %}
Delete myDelete;
{% endhighlight %}

Including a single instance of this field in a record prevents `atdna` from
emitting streaming code for the whole record subclass. This is useful for 
complex types where automatic DNA streaming isn't practical.

It's not necessary to use this type directly, the `DECL_EXPLICIT_DNA` macro
automatically places a Delete field in the record. See 
*Writing Explicit Marshalling Functions* below for details.

## Inheriting DNA Record Types ##

`atdna` is fully aware of type-inheritance involving DNA records. Chains of 
`DNA::read()` and `DNA::write()` are automatically inserted for 
indirectly-inherited DNA subclasses.

Base classes are always visited first, cascading to the individual subclass
implementation.

#### Example ####

{% highlight cpp %}
#include <Athena/DNA.hpp>

struct BaseDNA : public Athena::io::DNA<Athena::BigEndian>
{
    DECL_DNA
    Value<atUint32> myInt;
};

struct SubDNA : public BaseDNA
{
    DECL_DNA
    Value<atUint32> mySubInt;
};

struct SubSubDNA : public SubDNA
{
    DECL_DNA
    Value<atUint32> mySubSubInt;
};

void readTest(Athena::io::IStreamReader& reader)
{
    SubSubDNA ssDna;
    
    /* This call will run BaseDNA::read(), then SubDNA::read(),
     * then SubSubDNA::read() */
    ssDna.read(reader);
}
{% endhighlight %}

## Writing Explicit Marshalling Functions ##

ATDNA is a *one size fits many* solution. For complex custom data structures
like recursive index tables, trees and arbitrarily-chunked-formats, more
customized streaming code is required.

The `DECL_EXPLICIT_DNA` macro is an alternative to `DECL_DNA`. When an explicit
record is defined, `atdna` will skip emitting streaming code automatically.
The resulting program will fail to link, unless the programmer provides
replacement `DNA::read()` and/or `DNA::write()` implementations.

By convention, the implementation should start by setting the byte order on the
reader/writer using `IStreamReader::setEndian()` or `IStreamWriter::setEndian()`.

#### Example ####

{% highlight cpp %}
#include <Athena/DNA.hpp>

struct ExplicitDemo : public Athena::io::DNA<Athena::BigEndian>
{
    DECL_EXPLICIT_DNA
    Value<atUint32> myInt;
    Value<atUint16> myShort;
};

void ExplicitDemo::read(Athena::io::IStreamReader& reader)
{
    reader.setEndian(Athena::BigEndian);
    myInt = reader.readUint32();
    myShort = reader.readUint16();
}

void ExplicitDemo::write(Athena::io::IStreamWriter& writer) const
{
    writer.setEndian(Athena::BigEndian);
    writer.writeUint32(myInt);
    writer.writeUint16(myShort);
}
{% endhighlight %}



