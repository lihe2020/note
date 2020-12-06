CPU分析：go tool pprof http://172.17.24.204:5005/debug/pprof/profile?seconds=60
内存分析：go tool pprof http://172.17.24.204:5005/debug/pprof/heap?seconds=60

使用web方式打开pprof
```bash
go tool pprof -http=:8080 pprof.rc_strategy_go.samples.cpu.002.pb.gz
```