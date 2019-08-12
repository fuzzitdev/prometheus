This document provides step-by-step instructions on how to fuzz prometheus.

## What is fuzzing

Fuzzing is a way of finding bugs by exercising code with randomly generated inputs and watching for crashes or hangs. It's mostly used for parsing code.

## Fuzzing in prometheus

Prometheus has 4 fuzzing targets in [promql/fuzz.go](./promql/fuzz.go).

Each function starting with `Fuzz` (e.g. `FuzzParseMetric`) is a fuzzing target.

Fuzzing is most effective if generated inputs are based on a seed corpus of initial inputs.

Prometheus has 2 seed corpuses:

* `promql/fuzz-data/ParseExpr`
* `promql/fuzz-data/ParseMetric`

## Fuzzing with `go-fuzz`

Prometheus uses [go-fuzz](https://github.com/dvyukov/go-fuzz) for fuzzing.

Notice that `promql/fuzz.go` contains:

```go
// +build gofuzz
```

This code is only built when built with `go-fuzz`. 

Here's how to fuzz `FuzzParseOpenMetric` function.

Checkout code into `$GOPATH`:

```bash
cd `go env GOPATH`
mkdir -p src/github.com/prometheus
git clone https://github.com/prometheus/prometheus.git
cd prometheus
```

Install `go-fuzz`:

```bash
# go-fuzz doesn't support modules so have to disable them
export GO111MODULE=off
go get -u github.com/dvyukov/go-fuzz/go-fuzz
go get -u github.com/dvyukov/go-fuzz/go-fuzz-build
```

Create a corpus and working directory for the fuzzing process:

```bash
# create a working directory for the fuzzer and seed corpus
mkdir -p workdir-parse-open-metric/corpus

# copy the files that seed fuzzing to corpus sub-directory of working directory
cp promql/fuzz-data/ParseMetric/corpus/* workdir-parse-open-metric/corpus

# generate the fuzzing program. This compiles fuzz.go
# generates fuzzer executable and promql-fuzz.zip that packages
# data to drive fuzzing process
go-fuzz-build -func FuzzParseOpenMetric github.com/prometheus/prometheus/promql
```

Start fuzzing process:

```bash
go-fuzz -bin=./promql-fuzz.zip -workdir=workdir-parse-open-metric
```

Fuzzing exercises a target function with randomly generated inputs. The longer you run fuzzing process, the more likely it is to find a bug (e.g. a crash or a hang).

Thanks to keeping state in a working directory, fuzzing process can be stoped and when restarted will resume processing (as opposed to starting from scratch).

## Fuzzing with libFuzzer

`libFuzzer` implements fuzzing in `clang` toolchain. `go-fuzz` can produce binaries compatible with `libFuzzer`.

Currently `libFuzzer` support is only available on Linux.

Those instructions show how to do it on Mac or Windows, using docker.

If you use Linux and have `clang` installed, you can skip docker related instructions.

Start docker image and mount current directory (promethous sources):

```bash
docker run -v`pwd`:/go2/src/github.com/prometheus/prometheus -it golang:1.12.7-buster /bin/bash
```

The following commands are executed inside the docker container:

```bash
# install clang
echo "deb http://apt.llvm.org/buster/ llvm-toolchain-buster main" >> /etc/apt/sources.list
echo "deb-src http://apt.llvm.org/buster/ llvm-toolchain-buster main" >> /etc/apt/sources.list
wget -O - https://apt.llvm.org/llvm-snapshot.gpg.key| apt-key add -
apt update && apt install -y clang-9 lldb-9 lld-9

## setup go inside the container
# to access code, we've mounted host's $GOPATH under /go2
export GOPATH=/go2
echo $GOPATH
# ensure binaries installed with go get are in $PATH
export PATH="$PATH":/go2/bin
echo $PATH

## run fuzzing locally to make sure it works
# go-fuzz doesn't support Go modules, so let's explicitly turn them off
export GO111MODULE=off

go get -u github.com/dvyukov/go-fuzz/go-fuzz github.com/dvyukov/go-fuzz/go-fuzz-build
```

Build fuzz targets with `-libfuzzer` option:

```bash
cd /go2/src/github.com/prometheus/prometheus

go-fuzz-build -libfuzzer -func FuzzParseMetric -o fuzzer-parse-metric.a ./promql
clang-9 -fsanitize=fuzzer fuzzer-parse-metric.a -o fuzzer-parse-metric

go-fuzz-build -libfuzzer -func FuzzParseOpenMetric -o fuzzer-parse-open-metric.a ./promql
clang-9 -fsanitize=fuzzer fuzzer-parse-open-metric.a -o fuzzer-parse-open-metric

go-fuzz-build -libfuzzer -func FuzzParseMetricSelector -o fuzzer-parse-metric-selector.a ./promql
clang-9 -fsanitize=fuzzer fuzzer-parse-metric-selector.a -o fuzzer-parse-metric-selector

go-fuzz-build -libfuzzer -func FuzzParseExpr -o fuzzer-parse-expr.a ./promql
clang-9 -fsanitize=fuzzer fuzzer-parse-expr.a -o fuzzer-parse-expr
```

This generate fuzzing binaries `fuzzer-parser-metric` etc.

To run fuzzing, execute generated binary, optionally providing seed corpus directory as first argument:

```bash
./fuzzer-parse-metric ./promql/fuzz-data/ParseMetric/corpus
./fuzzer-parse-open-metric ./promql/fuzz-data/ParseMetric/corpus
./fuzzer-parse-metric-selector ./promql/fuzz-data/ParseMetric/corpus
./fuzzer-parse-expr ./promql/fuzz-data/ParseExpr/corpus
```

## Running continous fuzzing on fuzzit.dev

The longer the fuzzing process runs, the more likely it is to generate input that exposes a bug.

One option to run fuzzing process for a long time is to use a hosted service like [fuzzit.dev](https://fuzzit.dev) (free for open source projects).

The process is:

* create an account
* generate at least one api key for authentication (i.n. Settings)
* use `fuzzit` application (https://github.com/fuzzitdev/fuzzit/releases) to create one or more fuzzing targets with `fuzzit create target <target> -seed <seed-corpus.tgz>`
* build a `libFuzzer`-compatible fuzzing binary
* start a long-running fuzzing job for a given target with `fuzzit create job <target> <fuzzing-binary>`

Here's how it would look like for e.g. `promql-parse-metric` target, assuming we're submitting from Linux.

This assumes that we already built `libFuzzer` fuzzing target following the above instruction.

```bash
# install fuzzit binary
wget -q -O fuzzit https://github.com/fuzzitdev/fuzzit/releases/latest/download/fuzzit_Linux_x86_64
chmod ug+x fuzzit

# authenticate with fuzzit backend. authorization is persisted
./fuzzit auth <api_key>

# one time: create a target with seed corpus on the backend
tar -czvf corpus-metric.tar.gz -C ./promql/fuzz-data/ParseMetric/corpus .
./fuzzit create target promql-parse-metric --seed corpus-metric.tar.gz

# every time you update the code, build fuzzing binary and submit
# to the server for continous fuzzing (it'll replace the previously submitted binary):
./fuzzit create job --type fuzzing fuzzer-parse-metric ./fuzzer-parse-metric
```

## Integrating continous fuzzing with CI system

It's best to integrate this process with CI system like Travis CI or CircleCI.

Basic idea is to re-build targets on every checkin and upload for fuzzing with `fuzzit create job`.

Additionally, a short fuzzing job using past crashing test cases can be run on CI server directly to guard against regression.

The details for such integration is in https://github.com/fuzzitdev/example-go
