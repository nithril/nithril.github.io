---
layout: post
title:  "Elasticsearch Source size impact"
date:   2018-02-15 10:06:01
categories: benchmark
comments: true
---

> The _source field contains the original JSON document body that was passed at index time. The _source field itself is not indexed (and thus is not searchable), but it is stored so that it can be returned when executing fetch requests, like get or search.

`_source` is parsed and loaded in memory during indexation and most of the time during the request. 
In order to construct the result list, the coordinator node fetches the documents, they are transferred on the network, loaded in memory, parsed, aggregated, then returned.

This quicky will only focus on the impact of the `_source` size on the performance, not on the memory and the network pressure.


<!--more-->

# Protocol

For the purpose of this quicky I will not benchmark ES but I will microbenchmark the method used by ES to parse the JSON (`org.elasticsearch.search.lookup.SourceLookup#sourceAsMapAndType`)
then I will compare the results to Jackson and then to msgpack.

The input dataset is a JSON file of 10MB composed of 2 fields: `isbn` and `content`, `content` taking 99.99% of the size.

You may ask why such an unbalanced json: it was the performance issue I had to analyze ;).       


The code is fairly simple:

{% highlight java linenos %}

@State(Scope.Thread)
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@Warmup(iterations = 5)
@Measurement(iterations = 5)
public class ParseTest {

    private BytesArray jsonBytesArray;
    private ObjectMapper jsonMapper;
    private ObjectMapper msgPackMapper;
    private byte[] msgPackByte;
    private byte[] jsonBytes;

    public static void main(String[] args) throws RunnerException {

        Options opt = new OptionsBuilder()
                .include(ParseTest.class.getSimpleName())
                .forks(1)
                .build();
        new Runner(opt).run();

    }

    @Setup
    public void prepare() throws IOException {
        jsonMapper = new ObjectMapper();
        msgPackMapper = new ObjectMapper(new MessagePackFactory());

        jsonBytes = Resources.toByteArray(getClass().getResource("/foo.txt"));
        jsonBytesArray = new BytesArray(jsonBytes);
        msgPackByte = msgPackMapper.writeValueAsBytes(jsonMapper.readValue(jsonBytes, Map.class));
    }


    @Benchmark
    public Tuple<XContentType, Map<String, Object>> elasticBench() {
        return SourceLookup.sourceAsMapAndType(jsonBytesArray);
    }

    @Benchmark
    public Map jacksonBench() throws IOException {
        return jsonMapper.readValue(jsonBytesArray.array(), Map.class);
    }


    @Benchmark
    public Map msgpackBench() throws IOException {
        return msgPackMapper.readValue(msgPackByte, Map.class);
    }
}
{% endhighlight %} 



 
# Results

Ran on a core i5.

| Benchmark  | ms/op |
|:-----------|:---------|
| elasticBench          |  27.826 ± 1.002  ms/op |
| jacksonBench  | 27.423 ± 3.140  ms/op   |
| msgpackBench        | 11.442 ± 2.181  ms/op  | 


`elasticBench` and `jacksonBench` results are side by side. It is not a surprise as ES is using internally Jackson.
`msgpackBench` is 2.4 times more efficient than `elasticBench`. Most of the `elasticBench`
deserialization (and so jackson) time is taken by the UTF8 decoder whereas `msgpack` format is more efficient. 


The interesting part is the throughput and so the impact of the `_source` size on the search timing.

| Benchmark  | throughput |
|:-----------|:---------|
| elasticBench        |  359 MB/s |
| msgpackBench        | 873 MB/s  | 


With a 10MB _source, a search request returning 20 documents will take at least 556ms, and only for the response 
building. Without doubt the performance issue I had to analyze was caused by the `_source` size. 

`_source` handling may not be neutral and before ingesting large field, you should consider the impact on the performance by benchmarking.
on real use case scenarii.

The purpose is to avoid the case where the `_source` handling requires a significant percentage of the total ES processing time. 
And even if large document are not so common in an index, the impact will be visible on the percentile.

This quicky does not address the memory and the network pressure:
- _source is stored in a Lucene stored field (compressed): impact on the mapped file 
- 20 documents of 10MB will take at least 200MB (and even 400MB with UTF-16): loaded in memory then transferred through the network

Note that by using a more efficient storage format, ie. msgpack, ES may improve the efficiency 2.4 times. A technically low hanging fruit, but
potentially high because of the migration.


There are some workarounds:
- Exclude [the large field from the _source](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-source-field.html#include-exclude) with the associated downside.
- [Store](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-store.html) the fields and never returns the `_source` 

And some open issues to address the point:
- [Better storage of `_source`](https://github.com/elastic/elasticsearch/issues/9034)
- [Memory efficient source filtering](https://github.com/elastic/elasticsearch/issues/25168)
 

 
 