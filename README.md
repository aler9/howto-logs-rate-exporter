Sample code and instructions on how to build a command line tool to extract the requests rate from log files and push it to Prometheus, a lightweight alternative to mtail and grok_exporter.

## Introduction

Performance and access monitoring are almost mandatory if you're in charge of providing web services and ensuring they stay online. This task can be achieved with the help of Prometheus, a common time-series database that is used to store measurements of numeric parameters over time. When dealing with HTTP servers (Apache, Nginx, Lighttpd), one of the most important stress indicators (but not the only one) is the rate of the requests sent by the clients, that in most cases is not available directly, but can be inferred by examining the logs produced by the server. Please note that we're not interested in the logs content, but only in their rate, and that Prometheus [is not conceived](https://prometheus.io/docs/introduction/faq/#how-to-feed-logs-into-prometheus) to be a event logger, but a metrics collector.

There are at least two existing tools that can be used to compute the requests rate from log files:
* [grok_exporter](https://github.com/fstab/grok_exporter), that makes use of the powerful Grok syntax from Logstash (ELK Stack), it requires some time to configure and depends on a C library, not an ideal situation for a Go program.
* [mtail](https://github.com/google/mtail), that is lighter, uses a more friendly (IMHO) syntax for extracting metrics from logs, but it has not been officially ported to the ARM architecture and during my tests it produced a strange error message ("file type does not support deadline").

Since i needed to provide a logs rate indicator to a series of devices with different setups, different architectures (amd64 and armhf) and memory constraints, and i was not able to solve all the issues i encountered with both tools, i chosed to build a command line tool from scratch that implements this trivial task in no more than 65 lines of code.

## Instructions

1) Save the following code in a file called `logsrate.go` and edit the `patterns` variable to include all the log files you're interested in. The program will monitor the provided log files for new lines and at the same time it will create a server to provide metrics in the Prometheus format.
```go
package main

import (
    "os"
    "fmt"
    "sync"
    "net/http"
    "path/filepath"
    "github.com/hpcloud/tail"
)

var patterns = []string{
    "/var/log/nginx/*.log",
}

var counter = make(map[string]int)
var counterMutex sync.Mutex

func tailer() {
    for _,p := range patterns {
        matches,err := filepath.Glob(p)
        if err != nil {
            panic(err)
        }

        for _,fpath := range matches {
            t,err := tail.TailFile(fpath, tail.Config{
                Location: &tail.SeekInfo{0, os.SEEK_END},
                MustExist: true,
                Follow: true,
                ReOpen: true,
                Logger: tail.DiscardingLogger,
            })
            if err != nil {
                panic(err)
            }

            go func(fpath string, t *tail.Tail) {
                for range t.Lines {
                    func() {
                        counterMutex.Lock()
                        defer counterMutex.Unlock()
                        counter[fpath]++
                    }()
                }
            }(fpath, t)
        }
    }
}

func server() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        counterMutex.Lock()
        defer counterMutex.Unlock()
        for fpath,count := range counter {
            fmt.Fprintf(w, "logs_counter{log=\"%s\"} %d\n", fpath, count)
        }
    })
    http.ListenAndServe(":8934", nil)
}

func main() {
    go tailer()
    server()
}
```

2) Install Go and the tail dependency:
```bash
go get github.com/hpcloud/tail
```

3) Compile the tool:
```bash
go build .
```

4) Run the tool:
```bash
./logsrate
```

5) Insert the following lines in the `scrape_configs` section of the Prometheus configuration (`prometheus.yml`):
```yml
- job_name: logs
  scrape_interval: 5s
  static_configs:
  - targets: ['localhost:8934']
```

6) The logs should appear in the database, with the `logs_counter` label. To compute the rate of each file, it is enough to open the Prometheus web interface (or the popular frontend Grafana) and insert the following query:
```
rate(logs_counter[30s])
```
