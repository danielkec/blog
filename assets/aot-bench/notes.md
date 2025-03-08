```
Name                     |   AOT/build sec| Warmup start ms| Warmup req/s|  Startup ms| 5ss run req/s| 15ss run req/s
----------------------------------------------------------------------------------------------------------------------
Crac se                  |           16.03|             396|    284275.39|          18|     276701.81|     305240.68
Leyden se                |           31.88|             559|    283802.31|         129|     280346.42|     312333.01
Nativeimage se           |           65.42|                |             |           8|     211232.47|     206078.92
Nativeimage-pgo se       |          143.67|              38|    159198.70|           6|     228060.33|     222656.22
Vanilla se               |            8.43|                |             |         380|     265584.96|     295197.66
Crac mp                  |           17.55|            1630|     77914.09|          35|      84960.78|      95574.95
Leyden mp                |           84.38|            2823|     70548.20|         612|      49436.27|      76994.16
Nativeimage mp           |          136.71|                |             |          36|      65350.96|      67567.64
Nativeimage-pgo mp       |          301.41|             136|     27779.39|          29|      78072.71|      82528.88
Vanilla mp               |            7.99|                |             |        1700|      42254.71|      92442.97
----------------------------------------------------------------------------------------------------------------------
Results stored in /tmp/startup-benchmark-1741293625/results.csv

Image details
Operating system: Oracle Linux
Version: 8
Image: Oracle-Linux-8.10-2025.01.31-0
Shape configuration
Shape: VM.Optimized3.Flex
OCPU count: 8
Network bandwidth (Gbps): 32
Memory (GB): 64
Local disk: Block storage only


sudo dnf -y install git docker
git clone --single-branch --branch benchmarks/startup https://github.com/danielkec/helidon-labs.git
cd helidon-labs/benchmarks/startup
bash benchmark.sh

git clone https://github.com/danielkec/helidon-labs.git
cd helidon-labs/
git switch -c benchmarks/startup
git pull origin benchmarks/startup
cd benchmarks/startup
bash benchmark.sh

```

```csv
Name,AOT/build sec,Warmup start ms,Warmup req/s,Startup ms,5s run req/s,15s run req/s
Crac se,16.03,396,284275.39,18,276701.81,305240.68
Leyden se,31.88,559,283802.31,129,280346.42,312333.01
Nativeimage se,65.42,,,8,211232.47,206078.92
Nativeimage-pgo se,143.67,38,159198.70,6,228060.33,222656.22
Vanilla se,8.43,,,380,265584.96,295197.66
Crac mp,17.55,1630, 77914.09,35, 84960.78, 95574.95
Leyden mp,84.38,2823, 70548.20,612, 49436.27, 76994.16
Nativeimage mp,136.71,,,36, 65350.96, 67567.64
Nativeimage-pgo mp,301.41,136, 27779.39,29, 78072.71, 82528.88
Vanilla mp,7.99,,,1700, 42254.71, 92442.97
```


=================================================================================== 2nd run



```
Name                     |   AOT/build sec| Warmup start ms| Warmup req/s|  Startup ms| 5ss run req/s| 15ss run req/s
----------------------------------------------------------------------------------------------------------------------
Crac se                  |           16.06|             384|    369144.07|          20|     349244.56|     375627.50
Leyden se                |           28.99|             531|    366131.32|         126|     362764.13|     382723.54
Nativeimage se           |           50.70|                |             |           8|     374634.48|     378269.27
Nativeimage-pgo se       |          114.04|              38|    247108.15|           6|     438387.36|     429550.39
Vanilla se               |            5.73|                |             |         383|     347570.57|     379447.77
Crac mp                  |           17.62|            1620|     95163.17|          32|      92553.57|      99893.59
Leyden mp                |           82.71|            3374|     88586.15|         615|      73935.23|      85436.14
Nativeimage mp           |           96.47|                |             |          37|      77167.15|      79602.40
Nativeimage-pgo mp       |          239.37|             142|     27249.80|          31|      94230.85|      96838.39
Vanilla mp               |           11.64|                |             |        1597|      73078.18|     101808.88
----------------------------------------------------------------------------------------------------------------------
Results stored in /tmp/startup-benchmark-1741357244/results.csv

Name,AOT/build sec,Warmup start ms,Warmup req/s,Startup ms,5s run req/s,15s run req/s
Crac se,16.06,384,369144.07,20,349244.56,375627.50
Leyden se,28.99,531,366131.32,126,362764.13,382723.54
Nativeimage se,50.70,,,8,374634.48,378269.27
Nativeimage-pgo se,114.04,38,247108.15,6,438387.36,429550.39
Vanilla se,5.73,,,383,347570.57,379447.77
Crac mp,17.62,1620, 95163.17,32, 92553.57, 99893.59
Leyden mp,82.71,3374, 88586.15,615, 73935.23, 85436.14
Nativeimage mp,96.47,,,37, 77167.15, 79602.40
Nativeimage-pgo mp,239.37,142, 27249.80,31, 94230.85, 96838.39
Vanilla mp,11.64,,,1597, 73078.18,101808.88


Shape: VM.Standard3.Flex
OCPU count: 16
Network bandwidth (Gbps): 16
Memory (GB): 64
Local disk: Block storage only
Image: Oracle-Linux-8.10-2025.01.31-0
Architecture:        x86_64
CPU op-mode(s):      32-bit, 64-bit
Byte Order:          Little Endian
CPU(s):              32
On-line CPU(s) list: 0-31
Thread(s) per core:  2
Core(s) per socket:  16
Socket(s):           1
NUMA node(s):        1
Vendor ID:           GenuineIntel
CPU family:          6
Model:               106
Model name:          Intel(R) Xeon(R) Platinum 8358 CPU @ 2.60GHz
Stepping:            6
CPU MHz:             2593.994
BogoMIPS:            5187.98
Virtualization:      VT-x
Hypervisor vendor:   KVM
Virtualization type: full
L1d cache:           32K
L1i cache:           32K
L2 cache:            4096K
L3 cache:            16384K
NUMA node0 CPU(s):   0-31
Flags:               fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 
ss ht syscall nx pdpe1gb rdtscp lm constant_tsc arch_perfmon rep_good nopl xtopology cpuid tsc_known_freq pni pclmulqdq
 vmx ssse3 fma cx16 pdcm pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand hypervisor
  lahf_lm abm 3dnowprefetch cpuid_fault invpcid_single ssbd ibrs ibpb stibp ibrs_enhanced tpr_shadow flexpriority ept
   vpid ept_ad fsgsbase tsc_adjust bmi1 avx2 smep bmi2 erms invpcid avx512f avx512dq rdseed adx smap avx512ifma clflushopt
    clwb avx512cd sha_ni avx512bw avx512vl xsaveopt xsavec xgetbv1 xsaves nt_good wbnoinvd arat vnmi avx512vbmi umip pku
     ospke avx512_vbmi2 gfni vaes vpclmulqdq avx512_vnni avx512_bitalg avx512_vpopcntdq la57 rdpid overflow_recov succor
      fsrm md_clear arch_capabilities
```
