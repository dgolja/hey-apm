Basic load generation for apm-server built on [hey](https://github.com/rakyll/hey).

# Install

```
# populate vendor/
go get github.com/golang/dep/cmd/dep
dep ensure -v
```

# Docker build

```
docker build -f docker/Dockerfile .
```

# Versions

To work with the apm-server V1, use the v1 branch of this repo.

Support for apm-server V2 is under active development, so master will likely be unstable.
 

# Interactive shell mode

To run hey-apm in interactive shell mode pass the `interactive` flag without additional arguments.

```
./hey-apm -interactive
``` 

You then can connect to the port 8234 and start sending commands. It is recommended to use a readline wrapper. 

```
rlwrap telnet localhost 8234
``` 

### elasticsearch use and apm use

If it is your first session, you need to provide an elasticsearch url/username/password and a apm-server clone directory.

```
elasticsearch use local
apm use local
```

`local` above is short for `http://localhost:9200` / `no user` / `no password`; and `GOPATH/src/github.com/elastic/apm-server`.
If you have a Docker daemon running in the same host as hey-apm, you can also try `apm use docker`.

This expects elasticserch running locally in the default port. 
If you type `status`, you should see something like the following:

```
ElasticSearch [http://localhost:9200]: yellow ,  0 docs
ApmServer [http://0.0.0.0:8200]: not running
Using docker: unknown branch (hint: apm switch <branch>)
0 commands queued
```

Number of docs in elasticsearch always come from `apm-*` indices.

### apm switch and test

Workload tests generate reports that are saved to elasticsearch, and they include some information of the apm-server under test. 
For this you need to inform the branch and revision that you want to test, eg:

```
apm switch -v axw/trace-apm-server c961d664fbe5893d523bd9077be64617f57a96e7
```  

If you are using Docker, that will build an image ready to be used for tests.
The revision is not mandatory (it defaults to `HEAD`), but it is a good idea to always set it to leverage caching.
You can see images created by hey-apm with `apm list`. 

If you are not using Docker, you will likely want to pass `-fcm` options to fetch the remote, checkout the branch and run make.   

You can execute `status` again to verify that `apm switch` worked, and you should be ready to run some workload tests:

`test 30s 1 2 3 -E apm-server.tracing.enabled=true` 

That will start an apm-server process, run a test for 30 seconds, stop the apm-server, and save a report in elasticsearch. 
There are 4 arguments describing the test, plus any optional flags passed to the apm-server:

`1` - number of transactions per request

`2` - number of spans per transaction

`3` - size of the stacktrace, ie number of frames per span

You can specify `--errors N` to also ingest error events, configure the number of agents sending request concurrently, 
throttling, and more. 

After 30 seconds you should see something like the following:

```
sent 3 events per request with 1 agent(s)

[202] 9423 responses (100.00%)
total 9423 responses (314.09 rps)

on branch trace-apm-server
pushed 1.5Mb / sec , accepted 1.5Mb / sec

24671 docs indexed (822.33 / sec)
87.27% of expected
3.18 ms / request
186.9Mb (max RSS)
0.483 memory efficiency (accepted data volume per minute / memory used)

saved report to Elasticsearch
```
 
Since the number of documents per request and the number of accepted requests is known, hey-apm expects a specific count after a test is run. 

The actual count respect to the expected count is expressed as a percentage (79.18% in the example above).
A value bellow 90 usually indicates that apm-server produces more than what elasticsearch can ingest.
That can happen for a few reasons, eg. suboptimal configuration, or a saturated network link.  

_Memory efficiency_ relates data volume ingestion and maximum RSS used by the apm-server process.
It indicates how much memory is needed in order to validate and queue successfully X bytes of data during 1 minute.  
A higher value is better, and there isn't an upper bound.

### collate   
 
To analyze the report data you can read the test summaries, query elasticsearch at will, or use the `collate` command.
`collate` takes a required argument indicating which attribute interests you (the "variable"), and any number of query filters.

It will groups reports with all the independent variables having the same value except for the specified variable, and show a digest of them. 
Independent variables are the ones such as number of transactions per request, duration of the test, elasticsearch host, memory limit passed to Docker, and so on.  
 
Some useful possibilities are: 
- `collate branch` to see how branches compare to each other.
- `collate duration` to see if performance degrades over time (by comparing eg 1 minute tests with 20 minute tests). 
- `collate agents` to see how apm-server scales with heavier traffic.
- `collate events` to see how apm-server scales with heavier payloads.
- `collate revision branch=master --sort revision_date` to see how a branch (master in this case) changes over time. 
- `collate <flag>` to see how configuration  affect performance.

Try the following on `axw/trace-apm-server` branch:

```
test 30s 10 10 10 1 -E apm-server.tracing.enabled=false
test 30s 10 10 10 1 -E apm-server.tracing.enabled=true
test 30s 10 10 10 20 -E apm-server.tracing.enabled=false
test 30s 10 10 10 20 -E apm-server.tracing.enabled=true
```

When all the tests complete (`status` informs there are no queued commands and apm-server is not running), type:

```
collate apm-server.tracing.enabled branch=trace-apm-server
```

That will show 2 groups of reports, one with `agents=1` and another one with `agents=20`. 

If you run tests with different parameter values (transactions, duration, etc), you will get 1 group per variant.
Then, per each group you will see 2 reports, one with tracing enabled and other with tracing disabled:

```
duration 30s  errors 0 transactions 10  spans 10  frames 10  agents 20  branch trace-apm-server  revision c961d664fbe5893d523bd9077be64617f57a96e7
report id  revision date   pushed     accepted    throughput  latency  index  max rss  effic  apm-server.tracing.enabled
v5yruht0   18-05-18 10:21  7.5Mbps    4.8Mbps     435.3dps    185ms    73.3%  432.9Mb  0.669  false
958l1oly   18-05-18 10:21  7.7Mbps    4.0Mbps     349.6dps    222ms    70.6%  428.3Mb  0.564  true
```
```
duration 30s  transactions 10  spans 10  frames 10  agents 1  branch trace-apm-server  revision c961d664fbe5893d523bd9077be64617f57a96e7
report id  revision date   pushed     accepted    throughput  latency  index  max rss  effic  apm-server.tracing.enabled
xmlob0sh   18-05-18 10:21  5.2Mbps    5.3Mbps     491.2dps    169ms    75.7%  422.1Mb  0.750  false
bcl41bfu   18-05-18 10:21  5.1Mbps    5.1Mbps     494.5dps    174ms    78.4%  354.5Mb  0.868  true
``` 

- `report id` is included so you can filter by it and see full details in elasticsearch.
- `pushed` and `accepted` inform the amount of data per second sent to the apm-server and processed successfully (202 responses). 
- `throughput` is measured in number of elasticsearch documents inserted per second.

If one row is much more performing than other in the same group, it will be printed in green.

If you now want to compare tracing enabled/disabled with no tracing at all, you can checkout a revision from master 
without instrumentation code and run similar tests:

```
apm switch -v master d7722eb8de3e3bea60ac82472b05a4611aed78f5
test 30s 10 10 10 1
test 30s 10 10 10 20
collate branch branch=trace-apm-server
``` 

Full semantics for filters, sorting, etc. are given in the `help` command. 
 
### verify

`verify` can be used to ensure that the last commit of a branch is not performing worse than any previous commit dating back up to some time ago.
A number of filters are required:

```
verify -n 168h branch=master duration=30s errors=0 transactions=10 spans=10 frames=10 agents=1 limit=-1
```

### define

`define` can be used to create arbitrary aliases, with some limited form of variable substitution. 
 
 ```
define co apm switch -fcmv $
define ones test 30s 1 1 1 1
define tens test 30s 10 10 10 1
define do co ; ones ; tens
 ```

You then can type `do ; do master axw/trace-apm-server`. 
That will be first expanded according to the definitions above to:

```
apm switch -fcmv $ ; test 30s 1 1 1 1 ; test 30s 10 10 10 1 ;
apm switch -fcmv $ ; test 30s 1 1 1 1 ; test 30s 10 10 10 1
master axw/trace-apm-server
```

Only then the 2 `$` placeholders will be substituted with the last 2 words:
```
apm switch -fcmv master ; test 30s 1 1 1 1 ; test 30s 10 10 10 1 ;
apm switch -fcmv axw/trace-apm-server ; test 30s 1 1 1 1 ; test 30s 10 10 10 1
```
 
Which are the commands that will be queued and executed (try `status` again) 
 
### help

The `help` command describes the full semantics of the commands outlined above and some more things that hey-apm can do, like tail apm-server logs, etc. 
 
 
# Testing
 
 To run `hey-apm` unit tests:
```
SKIP_STRESS=1 go test -v ./...
```
Some tests require r/w access to disk, you can also supply SKIP_EXTERNAL=1 to omit them. 

To run apm-server stress tests:
```
ELASTICSEARCH_URL=<URL> ELASTICSEARCH_USR=<USR> ELASTICSEARCH_PWD=<PWD> go test -v github.com/elastic/hey-apm/server/client/
```
This requires a running Elasticsearch instance and its connection parameters; and apm-server binary and config file in the GOPATH
 
# Known issues / Todo

* The race detector is unhappy - https://github.com/rakyll/hey/issues/85
* Response latency distribution
