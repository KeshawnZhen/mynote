# Netty Benchmark

## 环境
百度云1核2G

Intel(R) Xeon(R) Gold 6148 CPU @ 2.40GHz

```
uname -a
Linux 4.4.0-141-generic #167-Ubuntu SMP Wed Dec 5 10:40:15 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
```

## 1 thread
第一次
```
1482424 0.98KiB messages in 30 seconds
Latency: min 0.02ms; max 7.67ms; mean 0.06ms; std: 0.057ms (95.26%)
Latency distribution: 25% under 0.053ms; 50% under 0.059ms; 75% under 0.065ms; 90% under 0.069ms; 99% under 0.099ms; 99.99% under 3.953ms
Requests/sec: 49414.13
Transfer/sec: 47.12MiB
```
第二次
```
1222009 0.98KiB messages in 30 seconds
Latency: min 0.02ms; max 40.11ms; mean 0.072ms; std: 0.111ms (152.95%)
Latency distribution: 25% under 0.059ms; 50% under 0.069ms; 75% under 0.081ms; 90% under 0.088ms; 99% under 0.121ms; 99.99% under 5.188ms
Requests/sec: 40733.63
Transfer/sec: 38.85MiB
```
第三次
```
1426429 0.98KiB messages in 30 seconds
Latency: min 0.02ms; max 36.44ms; mean 0.063ms; std: 0.099ms (156.39%)
Latency distribution: 25% under 0.055ms; 50% under 0.061ms; 75% under 0.066ms; 90% under 0.069ms; 99% under 0.104ms; 99.99% under 4.977ms
Requests/sec: 47547.63
Transfer/sec: 45.34MiB

```

## 2 thread
第一次
```
1426814 0.98KiB messages in 30 seconds
Latency: min 0.02ms; max 37.44ms; mean 0.058ms; std: 0.108ms (187.05%)
Latency distribution: 25% under 0.046ms; 50% under 0.052ms; 75% under 0.058ms; 90% under 0.069ms; 99% under 0.126ms; 99.99% under 5.079ms
Requests/sec: 47560.47
Transfer/sec: 45.36MiB
```
第二次
```
1401785 0.98KiB messages in 30 seconds
Latency: min 0.02ms; max 46.41ms; mean 0.063ms; std: 0.106ms (166.54%)
Latency distribution: 25% under 0.055ms; 50% under 0.061ms; 75% under 0.066ms; 90% under 0.07ms; 99% under 0.113ms; 99.99% under 4.756ms
Requests/sec: 46726.17
Transfer/sec: 44.56MiB
```
第三次
```
1424237 0.98KiB messages in 30 seconds
Latency: min 0.02ms; max 58.67ms; mean 0.062ms; std: 0.123ms (198.29%)
Latency distribution: 25% under 0.053ms; 50% under 0.059ms; 75% under 0.065ms; 90% under 0.07ms; 99% under 0.116ms; 99.99% under 4.948ms
Requests/sec: 47474.57
Transfer/sec: 45.28MiB
```