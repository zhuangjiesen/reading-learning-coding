
```
-server 
-Xms8g 
-Xmx8g 
-Xmn4g
-XX:+UseG1GC 
//每块堆内存的大小
-XX:G1HeapRegionSize=16m 

-XX:G1ReservePercent=25 
-XX:InitiatingHeapOccupancyPercent=30 
-XX:SoftRefLRUPolicyMSPerMB=0 
-XX:SurvivorRatio=8
-verbose:gc 
-Xloggc:/dev/shm/mq_gc_%p.log 
-XX:+PrintGCDetails 
-XX:+PrintGCDateStamps 
-XX:+PrintGCApplicationStoppedTime 
-XX:+PrintAdaptiveSizePolicy
-XX:+UseGCLogFileRotation 
-XX:NumberOfGCLogFiles=5 
-XX:GCLogFileSize=30m
-XX:-OmitStackTraceInFastThrow
-XX:+AlwaysPreTouch
-XX:MaxDirectMemorySize=15g
-XX:-UseLargePages -XX:-UseBiasedLocking

```

