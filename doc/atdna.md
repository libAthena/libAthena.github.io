---
layout: doc
title: ATDNA
---

This document provides an overview and reference for using **atdna**.

## Introduction ##

Athena's primary purpose as a core library is to facilitate access and
modification of individual *data fields* in a structured *data record*; 
streamed or buffered. Whenever
a developer uses Athena on its own, each individual field is read or written
using explicitly-typed methods like `IStreamReader::readUint32()` or 
`IStreamWriter::writeVec3f()`. 

For complex data records with recursive
hierarchies or random-access parsing, this code can become rather tedious.
Furthermore, separate functions must be maintained for reading and writing.
Some code-generation assistance would be useful in these cases.

## Overview ##

`atdna` is a source-to-source 
[clang tool](http://clang.llvm.org/docs/LibTooling.html) for 
transforming a specially-formed template syntax on C++ record-declarations
(i.e. *Structs*, *Classes*, *Unions*). The results are fully-implemented methods
that form the basis of the *ordered read/write flow* of the streamed data. 

Important details like byte-order 
([endianness](https://en.wikipedia.org/wiki/Endianness))
and
[contiguous subrecord tables](https://en.wikipedia.org/wiki/Array_data_structure)
are automatically accounted for by atdna's parser.

## Using ##

```
atdna [-o <out.cpp>] [-I <include-search-dir>...] <in.hpp>...
```

Since `atdna` is built on LLVM/clang, it functions much like the `clang` tool
itself, with `-o <out.cpp>` to specify output and one or more standalone args
to specify input header files `<in.hpp>...`. **If no output file is specified,
`a.cpp` is emitted in the working directory, overwriting the existing file if 
any!!**

## Diagnostics ##

Clang's built-in *diagnostics engine* is used to supply error reports for 
records that aren't well-formed for Athena (e.g. missing template parameters,
namespace conflicts). Additionally, *all* C++ syntax issues are diagnosed just
like `clang` would as a normal compiler.

{% highlight cpp %}
#include <Athena/DNA.hpp>

struct DemoRecord : public Athena::io::DNA<Athena::BigEndian>
{
    DECL_DNA
    Value<atUint32> myInt;
    Value<float> myFloat;

    Value<atUint32> myCount;
    struct DemoSubRecord : public Athena::io::DNA<Athena::BigEndian>
    {
        DECL_DNA
        Value<atUint16> myShort;
        Value<atUint8> myChar1;
        Value<atUint8> myChar2;
    };
    Vector<DemoSubRecord, DNA_COUNT(myCount)> mySubRecordArray;
};
{% endhighlight %}

<pre>
[jacko@ghor Desktop]$ atdna -o demo.cpp demo.hpp 
<strong>demo.hpp:11:5: <span style="color:red;">error:</span> Athena error: Unable to use type 'struct
      DemoRecord::NotDnaRecord' with Athena</strong>
    Vector<NotDnaRecord, DNA_COUNT(count)> invalidTable;
    <span style="color:#00cc00;">^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~</span>
</pre>

<div class="doc-col-split">
    <div class="doc-col">
        <h3>Sample Input</h3>
{% highlight cpp %}
#include <Athena/DNA.hpp>

struct DemoRecord : 
public Athena::io::DNA<Athena::BigEndian>
{
    DECL_DNA
    Value<atUint32> myInt;
    Value<float> myFloat;

    Value<atUint32> myCount;
    struct DemoSubRecord : 
    public Athena::io::DNA<Athena::BigEndian>
    {
        DECL_DNA
        Value<atUint16> myShort;
        Value<atUint8> myChar1;
        Value<atUint8> myChar2;
    };
    Vector<DemoSubRecord, DNA_COUNT(myCount)> mySubRecordArray;
};
{% endhighlight %}
    </div>
    <div class="doc-col">
        <h3>Sample Output</h3>
{% highlight cpp %}
/* Auto generated atdna implementation */
#include <Athena/Global.hpp>
#include <Athena/IStreamReader.hpp>
#include <Athena/IStreamWriter.hpp>

#include "demo.hpp"

void DemoRecord::read(Athena::io::IStreamReader& 
                      __dna_reader)
{
    __dna_reader.setEndian(Athena::BigEndian);
    /* myInt */
    myInt = __dna_reader.readUint32();
    /* myFloat */
    myFloat = __dna_reader.readFloat();
    /* myCount */
    myCount = __dna_reader.readUint32();
    /* mySubRecordArray */
    mySubRecordArray.clear();
    mySubRecordArray.reserve(myCount);
    for (size_t i=0 ; i<(myCount) ; ++i)
    {
        mySubRecordArray.emplace_back();
        mySubRecordArray.back().read(__dna_reader);
    }
}

void DemoRecord::write(Athena::io::IStreamWriter& 
                       __dna_writer) const
{
    __dna_writer.setEndian(Athena::BigEndian);
    /* myInt */
    __dna_writer.writeUint32(myInt);
    /* myFloat */
    __dna_writer.writeFloat(myFloat);
    /* myCount */
    __dna_writer.writeUint32(myCount);
    /* mySubRecordArray */
    for (auto elem : mySubRecordArray)
        elem.write(__dna_writer);
}

void DemoRecord::DemoSubRecord::read(Athena::io::IStreamReader& 
                                     __dna_reader)
{
    __dna_reader.setEndian(Athena::BigEndian);
    /* myShort */
    myShort = __dna_reader.readUint16();
    /* myChar1 */
    myChar1 = __dna_reader.readUByte();
    /* myChar2 */
    myChar2 = __dna_reader.readUByte();
}

void DemoRecord::DemoSubRecord::write(Athena::io::IStreamWriter& 
                                      __dna_writer) const
{
    __dna_writer.setEndian(Athena::BigEndian);
    /* myShort */
    __dna_writer.writeUint16(myShort);
    /* myChar1 */
    __dna_writer.writeUByte(myChar1);
    /* myChar2 */
    __dna_writer.writeUByte(myChar2);
}
{% endhighlight %}
    </div>
</div>




