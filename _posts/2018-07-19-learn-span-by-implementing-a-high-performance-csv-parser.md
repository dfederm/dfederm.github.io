---
layout: post
title: Learn Span<T> by Implementing a high-performance CSV Parser
date: 2018-07-19 21:14
categories: [.NET]
tags: [.NET, memory, performance, span]
comments: true
---
Ever since I [first](https://channel9.msdn.com/Events/Connect/2017/T125){:target="_blank"} [heard](https://msdn.microsoft.com/en-us/magazine/mt814808.aspx?f=255&MSPPError=-2147217396){:target="_blank"} [about](https://github.com/dotnet/corefxlab/blob/master/docs/specs/span.md){:target="_blank"} [`Span<T>`](http://adamsitnik.com/Span/){:target="_blank"}, I've been wanting play around with using it. It's a [`ref struct`](https://docs.microsoft.com/en-us/dotnet/csharp/reference-semantics-with-value-types#ref-struct-type){:target="_blank"}, so the semantics of using this type and the restrictions that go along with it are best understood by actually trying to use it. So I decided to build a simple CSV parser and see how the performance compared to other mainstream CSV parsers.

You can find the full code on [GitHub](https://github.com/dfederm/DelimiterSeparatedTextParser){:target="_blank"} and reference the package from [NuGet](https://www.nuget.org/packages/DelimiterSeparatedTextParser){:target="_blank"}.

As the main goal of this project was to exercise `Span<T>` and, as we'll later see, `Memory<T>`, so for the initial version I decided to ignore some of the details around CSV parsing such as [handling quotes](https://github.com/dfederm/DelimiterSeparatedTextParser/issues/1){:target="_blank"}, [trimming values](https://github.com/dfederm/DelimiterSeparatedTextParser/issues/2){:target="_blank"}, and [handling comments](https://github.com/dfederm/DelimiterSeparatedTextParser/issues/3){:target="_blank"}.

## Initial Implementation

For my initial implementation of a parser, I simply created a class which took a `ReadOnlyMemory<char>` as the content and two `ReadOnlySpan<char>`s as the value and record delimiters for example a comma and newline respectively for a CSV. Then in the constructor I would walk the entire content saving the index and length for each value in each row as a `List<List<long>>`. Finally, There are members on the class to allow getting the number of records, the number of values in a particular record, and a particular value. Already you may have some questions, so let's dig into the details.

Firstly, I needed to use a `ReadOnlyMemory<char>` to store the content, as `ref struct`s like `Span<T>` are stack-only and so can only be used in method parameters and local variables, while `Memory<T>` is a "regular" struct which can be stored on the heap. `ReadOnlyMemory<T>` can be sliced just as `Span<T>` can be without allocating or copying buffers around.

I wanted the parser to allow random-access into the values of the CSV, so I needed to store the index where each value started in the content, as well the length. To save on memory, I decided to store this data in `List<List<long>>` where the outer list represented a list of the records, the inner list represented the list of the values within a record, and the `long` (`int64`) store for each value was constructed by fusing the index and length data, both `int`s (`int32`), using [bit shifting](https://stackoverflow.com/a/33325313/7065632){:target="_blank"}.

After running some [benchmarks](https://github.com/dfederm/DelimiterSeparatedTextParser/tree/f651f1a107fbbaed9c869725fa2e8231da03b198#benchmark-results){:target="_blank"}, I found that already this implementation was better than other libraries, both in terms of speed and allocations.

Benchmark results after iteration 1:

| Method | Data Size | Mean | Scaled | Allocated |
| ------ | --------- | ----:| ------:| ---------:|
| **My Parser** | **Large** | **1,221.2610 us** | **0.59** | **16111.39 KB**
| StringSplit | Large | 93,149.630 us | 1.00 | 85937.55 KB |
| TinyCsvParser | Large | 169,713.472 us | 1.82 | 25072.83 KB |
| FastCsvParser | Large | 63,043.752 us | 0.68 | 47227.2 KB |
| CsvTextFieldParser | Large | 109,165.716 us | 1.17 | 182031.39 KB |
| CsvHelper | Large | 206,209.245 us | 2.21 | 173448.95 KB |
| FileHelpers | Large | 278,066.185 us | 2.99 | 83321.9 KB |
|  |  |  |  |  |
| **My Parser** | **Medium** | **307.973 us** | **0.76** | **157.07 KB** |
| StringSplit | Medium | 404.962 us | 1.00 | 859.36 KB |
| TinyCsvParser | Medium | 2,406.140 us | 5.94 | 322.31 KB |
| FastCsvParser | Medium | 717.625 us | 1.77 | 796.22 KB |
| CsvTextFieldParser | Medium | 1,066.085 us | 2.63 | 1820.45 KB |
| CsvHelper | Medium | 2,050.169 us | 5.06 | 1747.09 KB |
| FileHelpers | Medium | 2,069.213 us | 5.11 | 851.4 KB |
|  |  |  |  |  |
| **My Parser** | **Small** | **3.269 us** | **0.84** | **1.96 KB** |
| StringSplit | Small | 3.890 us | 1.00 | 8.58 KB |
| TinyCsvParser | Small | 1,059.621 us | 272.45 | 74.74 KB |
| FastCsvParser | Small | 22.811 us | 5.87 | 203.89 KB |
| CsvTextFieldParser | Small | 10.769 us | 2.77 | 18.34 KB |
| CsvHelper | Small | 22.715 us | 5.84 | 28.7 KB |
| FileHelpers | Small | 687.064 us | 176.66 | 31.07 KB |

## Splitting Out a No-allocation Reader

Next I decided to extract the reader from the parser so that scenarios which didn't need random-access to the data didn't need the allocations from the index data. Basically the reader would advance the memory as the consumer read it, much like an `IEnumerator<T>` or `TextReader`. Additionally, the parser was simplified as it could use the reader to enumerate the values and simply worry about building up the index and length data for random-access.

After categorizing the benchmarks into Readers vs Parsers, the [results](https://github.com/dfederm/DelimiterSeparatedTextParser/tree/10e5d738e0e897bfcdf1db384231238c8a178c0e#benchmark-results){:target="_blank"} showed that the reader implementation was far and away better than other libraries which exposed reader-like APIs. The closest contender was for a large data set using [FastCsvParser](https://www.nuget.org/packages/FastCsvParser/){:target="_blank"}, which only has a couple hundred downloads on NuGet.org as of this writing and still was took 3.5x longer to read the data. It requires zero allocations as it simply just stores and advances a span as well as a few other scalar fields.

Benchmark results after iteration 2:

| Method | Data Size | Mean | Scaled | Allocated |
| ------ | --------- | ----:| ------:| ---------:|
| **My Reader** | **Large** | **16,349.969 us** | **1.00** | **0 B** |
| FastCsvParser | Large | 57,649.310 us | 3.53 | 48360656 B |
| CsvTextFieldParser | Large | 103,398.356 us | 6.32 | 186400144 B |
| CsvHelper | Large | 194,618.607 us | 11.90 | 177611728 B |
|  |  |  |  |  |
| **My Parser** | **Large** | **53,862.981 us** | **1.00** | **16498940 B** |
| StringSplit | Large | 87,535.705 us | 1.63 | 88000046 B |
| TinyCsvParser | Large | 157,429.493 us | 2.92 | 25674400 B |
| FileHelpers | Large | 259,745.145 us | 4.82 | 85322480 B |
|  |  |  |  |  |
| **My Reader** | **Medium** | **161.729 us** | **1.00** | **0 B** |
| FastCsvParser | Medium | 640.133 us | 3.96 | 815328 B |
| CsvTextFieldParser | Medium | 1,024.653 us | 6.34 | 1864144 B |
| CsvHelper | Medium | 1,953.924 us | 12.08 | 1789016 B |
|  |  |  |  |  |
| **My Parser** | **Medium** | **321.522 us** | **1.00** | **160840 B** |
| StringSplit | Medium | 382.312 us | 1.19 | 879984 B |
| TinyCsvParser | Medium | 2,249.190 us | 7.00 | 330045 B |
| FileHelpers | Medium | 1,901.365 us | 5.91 | 871837 B |
|  |  |  |  |  |
| **My Reader** | **Small** | **1.647 us** | **1.00** | **0 B** |
| FastCsvParser | Small | 22.443 us | 13.62 | 208784 B |
| CsvTextFieldParser | Small | 10.202 us | 6.19 | 18784 B |
| CsvHelper | Small | 21.441 us | 13.02 | 29392 B |
|  |  |  |  |  |
| **My Parser** | **Small** | **3.385 us** | **1.00** | **2008 B** |
| StringSplit | Small | 3.703 us | 1.09 | 8784 B |
| TinyCsvParser | Small | 1,018.881 us | 301.01 | 76532 B |
| FileHelpers | Small | 664.814 us | 196.41 | 31812 B |

## To `ref struct` or not to `ref struct`
I found that using the reader API was fairly difficult, and I was unable to write convenience subclasses (eg. CSV reader, TSV reader) due to the fact that the reader was defined as a `ref struct` so that it could hold other `ref struct`s like `ReadOnlySpan<char>`. Because of this, the reader is unable to be stored as a field in a class, used in async APIs, or be on the heap in any other way. I decided that instead it could just hold a `ReadOnlyMemory<char>` and become a class for easier usage.

Unfortunately, based on the [benchmark results](https://github.com/dfederm/DelimiterSeparatedTextParser/tree/11012e179e267da4b4a3999e5c3402a2849eedac#benchmark-results){:target="_blank"}, this affected performance non-trivially and slowed down the reader by ~50%. The reader was still faster than the other implementations I tested against, but it was interesting to see just how optimized `Span<T>` and `ref struct` in general are.

Because of this, I may consider bringing back the `ref struct` implementation for usages which aren't limited by it.

Benchmark results after iteration 3:

| Method | Data Size | Mean | Scaled | Allocated |
| ------ | --------- | ----:| ------:| ---------:|
| **My Reader** | **Large** | **24,929.372 us** | **1.00** | **0 B** |
| FastCsvParser | Large | 58,659.486 us | 2.35 | 48360656 B |
| CsvTextFieldParser | Large | 105,256.008 us | 4.22 | 186400144 B |
| CsvHelper | Large | 196,541.906 us | 7.89 | 177611728 B |
|  |  |  |  |  |
| **My Parser** | **Large** | **59,266.792 us** | **1.00** | **16499337 B** |
| StringSplit | Large | 89,254.220 us | 1.51 | 88000057 B |
| TinyCsvParser | Large | 160,947.603 us | 2.72 | 25674926 B |
| FileHelpers | Large | 265,223.622 us | 4.48 | 85322864 B |
|  |  |  |  |  |
| **My Reader** | **Medium** | **244.256 us** | **1.00** | **96 B** |
| FastCsvParser | Medium | 686.830 us | 2.81 | 815328 B |
| CsvTextFieldParser | Medium | 1,050.241 us | 4.30 | 1864144 B |
| CsvHelper | Medium | 1,957.324 us | 8.01 | 1789016 B |
|  |  |  |  |  |
| **My Parser** | **Medium** | **375.222 us** | **1.00** | **160936 B** |
| StringSplit | Medium | 394.238 us | 1.05 | 879984 B |
| TinyCsvParser | Medium | 2,311.955 us | 6.16 | 330044 B |
| FileHelpers | Medium | 2,014.732 us | 5.37 | 871837 B |
|  |  |  |  |  |
| **My Reader** | **Small** | **2.494 us** | **1.00** | **96 B** |
| FastCsvParser | Small | 22.896 us | 9.18 | 208784 B |
| CsvTextFieldParser | Small | 10.506 us | 4.21 | 18784 B |
| CsvHelper | Small | 21.624 us | 8.67 | 29392 B |
|  |  |  |  |  |
| **My Parser** | **Small** | **3.985 us** | **1.00** | **2104 B** |
| StringSplit | Small | 3.825 us | 0.96 | 8784 B |
| TinyCsvParser | Small | 1,050.156 us | 263.60 | 76532 B |
| FileHelpers | Small | 680.903 us | 170.91 | 31812 B |

## Next Steps
While analyzing these benchmarks, I was surprised to find that, the simple and obvious `string.Split` implementation was better than most the other libraries I tested against. One explanation could be that those libraries are optimized for usability rather than performance, for example mapping the records to a class. However, at least in some cases this case be explained by "streamability". In my benchmarks, I loaded the entire content into memory as one big string prior to running each benchmark. I suspect that in real-world conditions like parsing the content from the disk or network, some of these libraries would perform much better. In fact, my implementation suffers from the same problem since it requires the entire buffer at once.

So the next step is to continue this exploration using [System.IO.Pipelines](https://blogs.msdn.microsoft.com/dotnet/2018/07/09/system-io-pipelines-high-performance-io-in-net/){:target="_blank"}. Stay tuned for a follow up post about that!