# Zero Allocation JSON Logger

[![godoc](http://img.shields.io/badge/godoc-reference-blue.svg?style=flat)](https://godoc.org/github.com/rs/zerolog) [![license](http://img.shields.io/badge/license-MIT-red.svg?style=flat)](https://raw.githubusercontent.com/rs/zerolog/master/LICENSE) [![Build Status](https://travis-ci.org/rs/zerolog.svg?branch=master)](https://travis-ci.org/rs/zerolog) [![Coverage](http://gocover.io/_badge/github.com/rs/zerolog)](http://gocover.io/github.com/rs/zerolog)

The zerolog package provides a fast and simple logger dedicated to JSON output. It is inspired by uber's [zap](https://godoc.org/go.uber.org/zap) but with a simpler API and a smaller code base.

## Features

* Level logging
* Sampling
* Contextual fields

## Performance

All operations are allocation free (those numbers *include* JSON encoding):

```
BenchmarkLogEmpty-8            50000000      19.8 ns/op     0 B/op      0 allocs/op
BenchmarkDisabled-8           100000000       4.73 ns/op    0 B/op      0 allocs/op
BenchmarkInfo-8                10000000      85.1 ns/op     0 B/op      0 allocs/op
BenchmarkContextFields-8       10000000      81.9 ns/op     0 B/op      0 allocs/op
BenchmarkLogFields-8            5000000     247 ns/op       0 B/op      0 allocs/op
```

Using Uber's zap [comparison benchmark](https://github.com/uber-go/zap#performance):

Log a message and 10 fields:

| Library | Time | Bytes Allocated | Objects Allocated |
| :--- | :---: | :---: | :---: |
| zerolog | 787 ns/op | 80 B/op | 6 allocs/op |
| :zap: zap | 848 ns/op | 704 B/op | 2 allocs/op |
| :zap: zap (sugared) | 1363 ns/op | 1610 B/op | 20 allocs/op |
| go-kit | 3614 ns/op | 2895 B/op | 66 allocs/op |
| lion | 5392 ns/op | 5807 B/op | 63 allocs/op |
| logrus | 5661 ns/op | 6092 B/op | 78 allocs/op |
| apex/log | 15332 ns/op | 3832 B/op | 65 allocs/op |
| log15 | 20657 ns/op | 5632 B/op | 93 allocs/op |

Log a message with a logger that already has 10 fields of context:

| Library | Time | Bytes Allocated | Objects Allocated |
| :--- | :---: | :---: | :---: |
| zerolog | 80 ns/op | 0 B/op | 0 allocs/op |
| :zap: zap | 283 ns/op | 0 B/op | 0 allocs/op |
| :zap: zap (sugared) | 337 ns/op | 80 B/op | 2 allocs/op |
| lion | 2702 ns/op | 4074 B/op | 38 allocs/op |
| go-kit | 3378 ns/op | 3046 B/op | 52 allocs/op |
| logrus | 4309 ns/op | 4564 B/op | 63 allocs/op |
| apex/log | 13456 ns/op | 2898 B/op | 51 allocs/op |
| log15 | 14179 ns/op | 2642 B/op | 44 allocs/op |

Log a static string, without any context or `printf`-style templating:

| Library | Time | Bytes Allocated | Objects Allocated |
| :--- | :---: | :---: | :---: |
| zerolog | 76.2 ns/op | 0 B/op | 0 allocs/op |
| :zap: zap | 236 ns/op | 0 B/op | 0 allocs/op |
| standard library | 453 ns/op | 80 B/op | 2 allocs/op |
| :zap: zap (sugared) | 337 ns/op | 80 B/op | 2 allocs/op |
| go-kit | 508 ns/op | 656 B/op | 13 allocs/op |
| lion | 771 ns/op | 1224 B/op | 10 allocs/op |
| logrus | 1244 ns/op | 1505 B/op | 27 allocs/op |
| apex/log | 2751 ns/op | 584 B/op | 11 allocs/op |
| log15 | 5181 ns/op | 1592 B/op | 26 allocs/op |


## Usage

```go
import "github.com/rs/zerolog/log"
```

### A global logger can be use for simple logging

```go
log.Info().Msg("hello world")

// Output: {"level":"info","time":1494567715,"message":"hello world"}
```

NOTE: To import the global logger, import the `log` subpackage `github.com/rs/zerolog/log`.

```go
log.Fatal().
    Err(err).
    Str("service", service).
    Msgf("Cannot start %s", service)

// Output: {"level":"fatal","time":1494567715,"message":"Cannot start myservice","error":"some error","service":"myservice"}
// Exit 1
```

NOTE: Using `Msgf` generates an allocation even when the logger is disabled.

### Fields can be added to log messages

```go
log.Info().
    Str("foo", "bar").
    Int("n", 123).
    Msg("hello world")

// Output: {"level":"info","time":1494567715,"foo":"bar","n":123,"message":"hello world"}
```

### Create logger instance to manage different outputs

```go
logger := zerolog.New(os.Stderr).With().Timestamp().Logger()

logger.Info().Str("foo", "bar").Msg("hello world")

// Output: {"level":"info","time":1494567715,"message":"hello world","foo":"bar"}
```

### Sub-loggers let you chain loggers with additional context

```go
sublogger := log.With().
                 Str("component": "foo").
                 Logger()
sublogger.Info().Msg("hello world")

// Output: {"level":"info","time":1494567715,"message":"hello world","component":"foo"}
```

### Level logging

```go
zerolog.SetGlobalLevel(zerolog.InfoLevel)

log.Debug().Msg("filtered out message")
log.Info().Msg("routed message")

if e := log.Debug(); e.Enabled() {
    // Compute log output only if enabled.
    value := compute()
    e.Str("foo": value).Msg("some debug message")
}

// Output: {"level":"info","time":1494567715,"message":"routed message"}
```

### Sub dictionary

```go
log.Info().
    Str("foo", "bar").
    Dict("dict", zerolog.Dict().
        Str("bar", "baz").
        Int("n", 1)
    ).Msg("hello world")

// Output: {"level":"info","time":1494567715,"foo":"bar","dict":{"bar":"baz","n":1},"message":"hello world"}
```

### Customize automatic field names

```go
zerolog.TimestampFieldName = "t"
zerolog.LevelFieldName = "l"
zerolog.MessageFieldName = "m"

log.Info().Msg("hello world")

// Output: {"l":"info","t":1494567715,"m":"hello world"}
```

### Log with no level nor message

```go
log.Log().Str("foo","bar").Msg("")

// Output: {"time":1494567715,"foo":"bar"}
```

### Add contextual fields to the global logger

```go
log.Logger = log.With().Str("foo", "bar").Logger()
```

### Log Sampling

```go
sampled := log.Sample(10)
sampled.Info().Msg("will be logged every 10 messages")

// Output: {"time":1494567715,"sample":10,"message":"will be logged every 10 messages"}
```

## Global Settings

Some settings can be changed and will by applied to all loggers:

* `log.Logger`: You can set this value to customize the global logger (the one used by package level methods).
* `zerolog.SetGlobalLevel`: Can raise the mimimum level of all loggers. Set this to `zerolog.Disable` to disable logging altogether (quiet mode).
* `zerolog.DisableSampling`: If argument is `true`, all sampled loggers will stop sampling and issue 100% of their log events.
* `zerolog.TimestampFieldName`: Can be set to customize `Timestamp` field name.
* `zerolog.LevelFieldName`: Can be set to customize level field name.
* `zerolog.MessageFieldName`: Can be set to customize message field name.
* `zerolog.ErrorFieldName`: Can be set to customize `Err` field name.
* `zerolog.SampleFieldName`: Can be set to customize the field name added when sampling is enabled.
* `zerolog.TimeFieldFormat`: Can be set to customize `Time` field value formatting.

## Field Types

### Standard Types

* `Str`
* `Bool`
* `Int`, `Int8`, `Int16`, `Int32`, `Int64`
* `Uint`, `Uint8`, `Uint16`, `Uint32`, `Uint64`
* `Float32`, `Float64`

### Advanced Fields

* `Timestamp`: Insert UNIX timestamp field with `zerolog.TimestampFieldName` field name.
* `Time`: Add a field with the time formated with the `zerolog.TimeFieldFormat`.
* `Err`: Takes an `error` and render it as a string using the `zerolog.ErrorFieldName` field name.
