# 网络通信库 Benchmark

## 环境
百度云1核2G

Intel(R) Xeon(R) Gold 6148 CPU @ 2.40GHz

```
uname -a
Linux 4.4.0-141-generic #167-Ubuntu SMP Wed Dec 5 10:40:15 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
```

## libuv
第一次
```
1751679 0.98KiB messages in 30 seconds
Latency: min 0.01ms; max 8.65ms; mean 0.051ms; std: 0.011ms (21.41%)
Latency distribution: 25% under 0.045ms; 50% under 0.051ms; 75% under 0.056ms; 90% under 0.059ms; 99% under 0.075ms; 99.99% under 0.25ms
Requests/sec: 58389.3
Transfer/sec: 55.68MiB
```
第二次
```
1736091 0.98KiB messages in 30 seconds
Latency: min 0.01ms; max 8.36ms; mean 0.051ms; std: 0.014ms (26.83%)
Latency distribution: 25% under 0.045ms; 50% under 0.051ms; 75% under 0.056ms; 90% under 0.059ms; 99% under 0.078ms; 99.99% under 0.324ms
Requests/sec: 57869.7
Transfer/sec: 55.19MiB
```
第三次
```
1776545 0.98KiB messages in 30 seconds
Latency: min 0.01ms; max 8.59ms; mean 0.051ms; std: 0.011ms (21.03%)
Latency distribution: 25% under 0.045ms; 50% under 0.05ms; 75% under 0.055ms; 90% under 0.058ms; 99% under 0.067ms; 99.99% under 0.193ms
Requests/sec: 59218.17
Transfer/sec: 56.47MiB
```
10kb
```
1581419 10.0KiB messages in 30 seconds
Latency: min 0.02ms; max 7.47ms; mean 0.052ms; std: 0.014ms (26.65%)
Latency distribution: 25% under 0.046ms; 50% under 0.051ms; 75% under 0.056ms; 90% under 0.06ms; 99% under 0.086ms; 99.99% under 0.344ms
Requests/sec: 52713.97
Transfer/sec: 514.78MiB
```

100kb
```
455640 100.0KiB messages in 30 seconds
Latency: min 0.05ms; max 7.52ms; mean 0.194ms; std: 0.037ms (18.99%)
Latency distribution: 25% under 0.185ms; 50% under 0.191ms; 75% under 0.198ms; 90% under 0.207ms; 99% under 0.28ms; 99.99% under 1.112ms
Requests/sec: 15188.0
Transfer/sec: 1483.2MiB
```

## libco
第一次
```
1709739 0.98KiB messages in 30 seconds
Latency: min 0.01ms; max 7.37ms; mean 0.051ms; std: 0.012ms (23.44%)
Latency distribution: 25% under 0.045ms; 50% under 0.051ms; 75% under 0.056ms; 90% under 0.059ms; 99% under 0.075ms; 99.99% under 0.321ms
Requests/sec: 56991.3
Transfer/sec: 54.35MiB
```
第二次
```
1744872 0.98KiB messages in 30 seconds
Latency: min 0.01ms; max 7.86ms; mean 0.051ms; std: 0.01ms (19.42%)
Latency distribution: 25% under 0.045ms; 50% under 0.051ms; 75% under 0.056ms; 90% under 0.059ms; 99% under 0.074ms; 99.99% under 0.309ms
Requests/sec: 58162.4
Transfer/sec: 55.47MiB
```
第三次
```
1678365 0.98KiB messages in 30 seconds
Latency: min 0.01ms; max 7.36ms; mean 0.052ms; std: 0.014ms (27.98%)
Latency distribution: 25% under 0.046ms; 50% under 0.051ms; 75% under 0.056ms; 90% under 0.059ms; 99% under 0.081ms; 99.99% under 0.329ms
Requests/sec: 55945.5
Transfer/sec: 53.35MiB
```
10kb
```
1535979 10.0KiB messages in 30 seconds
Latency: min 0.02ms; max 8.21ms; mean 0.052ms; std: 0.012ms (23.58%)
Latency distribution: 25% under 0.046ms; 50% under 0.051ms; 75% under 0.056ms; 90% under 0.059ms; 99% under 0.087ms; 99.99% under 0.339ms
Requests/sec: 51199.3
Transfer/sec: 499.99MiB
```
100kb
```
2381 100.0KiB messages in 30 seconds
Latency: min 0.07ms; max 43.78ms; mean 37.775ms; std: 9.061ms (23.99%)
Latency distribution: 25% under 39.852ms; 50% under 39.944ms; 75% under 40.064ms; 90% under 40.115ms; 99% under 40.209ms; 99.99% under 43.785ms
Requests/sec: 79.37
Transfer/sec: 7.75MiB
```

## python3 + uvloop
第一次
```
will connect to: ('127.0.0.1', 25000)
Sending 200000 messages
Sending 200000 messages
Sending 200000 messages
600000 in 15.040270805358887
39892.89872268912 requests/sec

---
983184 0.98KiB messages in 30 seconds
Latency: min 0.03ms; max 5.66ms; mean 0.03ms; std: 0.012ms (40.19%)
Latency distribution: 25% under 0.025ms; 50% under 0.03ms; 75% under 0.035ms; 90% under 0.038ms; 99% under 0.044ms; 99.99% under 0.114ms
Requests/sec: 32772.8
Transfer/sec: 31.25MiB

```
第二次
```
will connect to: ('127.0.0.1', 25000)
Sending 200000 messages
Sending 200000 messages
Sending 200000 messages
600000 in 14.316159725189209
41910.680763382596 requests/sec
---
990978 0.98KiB messages in 30 seconds
Latency: min 0.03ms; max 1.15ms; mean 0.03ms; std: 0.005ms (16.59%)
Latency distribution: 25% under 0.025ms; 50% under 0.03ms; 75% under 0.035ms; 90% under 0.038ms; 99% under 0.045ms; 99.99% under 0.202ms
Requests/sec: 33032.6
Transfer/sec: 31.5MiB

```
第三次
```
will connect to: ('127.0.0.1', 25000)
Sending 200000 messages
Sending 200000 messages
Sending 200000 messages
600000 in 14.586745500564575
41133.23290495315 requests/sec
---
980106 0.98KiB messages in 30 seconds
Latency: min 0.03ms; max 5.81ms; mean 0.03ms; std: 0.012ms (40.78%)
Latency distribution: 25% under 0.025ms; 50% under 0.03ms; 75% under 0.035ms; 90% under 0.038ms; 99% under 0.046ms; 99.99% under 0.162ms
Requests/sec: 32670.2
Transfer/sec: 31.16MiB
```

10kb
```
922949 10.0KiB messages in 30 seconds
Latency: min 0.03ms; max 4.14ms; mean 0.031ms; std: 0.008ms (27.48%)
Latency distribution: 25% under 0.025ms; 50% under 0.031ms; 75% under 0.036ms; 90% under 0.039ms; 99% under 0.049ms; 99.99% under 0.161ms
Requests/sec: 30764.97
Transfer/sec: 300.44MiB
```
100kb
```
450081 100.0KiB messages in 30 seconds
Latency: min 0.06ms; max 4.52ms; mean 0.063ms; std: 0.02ms (31.36%)
Latency distribution: 25% under 0.056ms; 50% under 0.061ms; 75% under 0.067ms; 90% under 0.07ms; 99% under 0.111ms; 99.99% under 0.402ms
Requests/sec: 15002.7
Transfer/sec: 1465.11MiB
```

## boost.asio
第一次
```
1426941 0.98KiB messages in 30 seconds
Latency: min 0.02ms; max 8.06ms; mean 0.061ms; std: 0.017ms (28.32%)
Latency distribution: 25% under 0.055ms; 50% under 0.061ms; 75% under 0.066ms; 90% under 0.069ms; 99% under 0.088ms; 99.99% under 0.353ms
Requests/sec: 47564.7
Transfer/sec: 45.36MiB
```
第二次
```
1433938 0.98KiB messages in 30 seconds
Latency: min 0.02ms; max 7.18ms; mean 0.061ms; std: 0.017ms (28.39%)
Latency distribution: 25% under 0.055ms; 50% under 0.061ms; 75% under 0.066ms; 90% under 0.069ms; 99% under 0.085ms; 99.99% under 0.354ms
Requests/sec: 47797.93
Transfer/sec: 45.58MiB
```
第三次
```
1426918 0.98KiB messages in 30 seconds
Latency: min 0.02ms; max 7.52ms; mean 0.061ms; std: 0.025ms (40.57%)
Latency distribution: 25% under 0.055ms; 50% under 0.061ms; 75% under 0.066ms; 90% under 0.069ms; 99% under 0.08ms; 99.99% under 0.435ms
Requests/sec: 47563.93
Transfer/sec: 45.36MiB
```
10Kb
```
2253 10.0KiB messages in 30 seconds
Latency: min 0.09ms; max 47.84ms; mean 39.921ms; std: 1.49ms (3.73%)
Latency distribution: 25% under 39.837ms; 50% under 39.984ms; 75% under 40.101ms; 90% under 40.145ms; 99% under 40.236ms; 99.99% under 47.845ms
Requests/sec: 75.1
Transfer/sec: 0.73MiB
```

100kb
```
2252 100.0KiB messages in 30 seconds
Latency: min 0.48ms; max 47.89ms; mean 39.909ms; std: 1.885ms (4.72%)
Latency distribution: 25% under 39.77ms; 50% under 39.985ms; 75% under 40.168ms; 90% under 40.214ms; 99% under 40.36ms; 99.99% under 47.895ms
Requests/sec: 75.07
Transfer/sec: 7.33MiB
```
## Netty 1 thread
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
10 kb
```
1377536 10.0KiB messages in 30 seconds
Latency: min 0.03ms; max 54.86ms; mean 0.064ms; std: 0.113ms (176.59%)
Latency distribution: 25% under 0.056ms; 50% under 0.061ms; 75% under 0.066ms; 90% under 0.07ms; 99% under 0.1ms; 99.99% under 5.425ms
Requests/sec: 45917.87
Transfer/sec: 448.42MiB
```
100kb
```
401695 100.0KiB messages in 30 seconds
Latency: min 0.16ms; max 49.27ms; mean 0.219ms; std: 0.213ms (97.09%)
Latency distribution: 25% under 0.199ms; 50% under 0.206ms; 75% under 0.216ms; 90% under 0.23ms; 99% under 0.334ms; 99.99% under 6.928ms
Requests/sec: 13389.83
Transfer/sec: 1307.6MiB
```

## Netty 2 thread
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

