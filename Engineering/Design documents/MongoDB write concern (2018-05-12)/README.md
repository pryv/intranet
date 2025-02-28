|         |                       |
| ------- | --------------------- |
| Author  | Thiébaud Modoux 	  |
| Date    | 05.12.2018            |
| Version | 1                     |

# Activation of Mongo J write concern

Through this document, we want to reconsider our current Mongo configuration and review what we currently do to provide durability in the event of a database failure. We are particularly interested in Mongo [journaling](https://docs.mongodb.com/manual/core/journaling/) and [write concern](https://docs.mongodb.com/manual/reference/write-concern/).

What preventive measures are currently activated within our Mongo instances? Would using j:true improve our persistence story in the face of a crash? Will it make pryv.io so slow that we cannot go there? We will try to answer these questions in the below sections.

## Current situation

We currently use the default Mongo configuration, i.e. activating the journal and using w:1 option as write concern. This value of write concern means that we request acknowledgement that each write operation has propagated to the standalone mongod or the primary in a replica set.

The other write concern option j:false being deactivated, we do not wait acknowledgement that write operations have reached the on-disk journal. Hence, we only have in-memory guarantee, which is not great in the face of a crash.

## Introducing j:true

In order to improve resilience to crashes, we would like to introduce j:true write concern, which will prevails in front of w:1. Since it's mainly a tradeoff between performance and data safety, we first want to ensure that the potential gain in data safety does not come with an overhead that will totally kill Pryv performance.

This is why we present in this section some results from a benchmark that compares j:false and j:true.

### Service-core test suite

The following measures represent the total duration in seconds of a run of the full service-core test-suite. There are 3 runs with j:false and 3 runs with j:true.

| Runs |  J:false                |  J:true                |
| ---- | --------------------- |--------------------- |
| 1    | 107.65 real, 104.16 user, 11.97 sys | 130.41 real, 113.41 user, 13.44 sys
| 2    | 106.51 real, 104.62 user, 11.87 sys | 131.59 real, 112.93 user, 13.24 sys
| 3    | 112.44 real, 110.38 user, 13.04 sys | 124.74 real, 105.76 user, 11.88 sys

### Batch event creation against pryv.li

We performed batch calls against a pryv.li core using the following command:

> ab -n REQUESTS -c CONCURENCY -p batch.json -T HEADER $URL >> "results/{DOMAIN}_{REQUESTS}R_{CONCURENCY}C.txt"

and with the following event creation batch:

```
[{"method":"events.create","params":{"time":1385046854.282,"streamId":"diary","type":"frequency/bpm","content":90}},{"method":"events.create","params":{"time":1385046854.282,"streamId":"diary","type":"pressure/mmhg","content":120}},{"method":"events.create","params":{"time":1385046854.282,"streamId":"diary","type":"pressure/mmhg","content":80}},{"method":"events.create","params":{"time":1385046854.282,"streamId":"diary","type":"frequency/bpm","content":90}},{"method":"events.create","params":{"time":1385046854.282,"streamId":"diary","type":"pressure/mmhg","content":120}},{"method":"events.create","params":{"time":1385046854.282,"streamId":"diary","type":"pressure/mmhg","content":80}},{"method":"events.create","params":{"time":1385046854.282,"streamId":"diary","type":"frequency/bpm","content":90}},{"method":"events.create","params":{"time":1385046854.282,"streamId":"diary","type":"pressure/mmhg","content":120}},{"method":"events.create","params":{"time":1385046854.282,"streamId":"diary","type":"pressure/mmhg","content":80}},{"method":"events.create","params":{"time":1385046854.282,"streamId":"diary","type":"frequency/bpm","content":90}},{"method":"events.create","params":{"time":1385046854.282,"streamId":"diary","type":"pressure/mmhg","content":120}},{"method":"events.create","params":{"time":1385046854.282,"streamId":"diary","type":"pressure/mmhg","content":80}}]
```

For REQUESTS and CONCURENCY, we went through the following set of values: (10,1), (10,10), (100,1), (100,10), (100,100), (1000,1), (1000,10), (1000,100), (1000,1000).

Results (raw and graphs) can be seen in the following files: [openoffice](results.ods), [word](resultsword.xls).

## Conclusion

We did not observe any significative loss of performance so we decide to go ahead with the introduction of j:true.