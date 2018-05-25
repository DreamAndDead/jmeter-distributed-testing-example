# Collector

## influxdb

### setup

influxdb 同样使用 docker 启动，非常简单，参考 [docker hub][influxdb docker]。

因为在这个结构中，influxdb 收集的数据来自多个地方，使用了多种数据采集方式
- graphite for jmeter
- http for jmeter and cadvisor
- udp for gitlab

配置已经在 influxdb 的[配置文件](./conf/influxdb.conf)中打开，包括端口地址等相关参数。

在启动 influxdb 容器之后，就要为要采集的数据建立数据库，分门别类。

```
curl -X POST http://localhost:8086/query --data-urlencode "q=CREATE DATABASE jmeter"
curl -X POST http://localhost:8086/query --data-urlencode "q=CREATE DATABASE graphite"
curl -X POST http://localhost:8086/query --data-urlencode "q=CREATE DATABASE cadvisor"
curl -X POST http://localhost:8086/query --data-urlencode "q=CREATE DATABASE gitlab"
```

剩下就需要应用自身接入相应的数据库，传输 metric 数据。

### jmeter metric

在文章中也提到过，jmeter Backend Listener 有两种，influxdb 与 graphite，这里分别来介绍。

#### influxdb way

首先，在 Jmeter 测试计划中，添加 Backend Listener，选择 influxdb implementation。参考[官方文档][jmeter metric influxdb]

![](./jmeter-influxdb-listener.png)

其中的相关配置，[应该这样解读][jmeter backend listener config]

- influxdbMetricsSender，保持不动
- influxdbUrl，选择db服务的地址，注意后面的 db 参数，和之前设置的数据库保持一致
- application，自己命名，后期作为筛选条件
- measurement，保持jmeter不动
- summaryOnly，改为false，我们需要实时统计而不是一个总结
- samplersRegex，可以与application结合起来，绑定一组samplers，这里使用全部
- percentiles，保持默认即可，百分比统计
- testTitle，测试名称，存储在influxdb events 中的 text 字段中
- eventTags，用来进行标注，暂时用不到

在一次测试之后，jmeter 数据库中存储了部分数据，这里来解读下数据结构是怎样的。


series 可以理解为表，最前面的为表名，如 events，jmeter，后续的键值可以理解为选择表的条件，必需将
所有列限定，才算作指定一个表，取到其中存储的数据。

```
$ docker exec -it collector_db_1 influx --database 'jmeter' -execute 'show series' 
key
---
events,application=gitlab,title=ApacheJMeter
jmeter,application=gitlab,statut=all,transaction=HTTP请求
jmeter,application=gitlab,statut=all,transaction=all
jmeter,application=gitlab,statut=all,transaction=创建用户
jmeter,application=gitlab,statut=all,transaction=删除用户
jmeter,application=gitlab,statut=all,transaction=查询用户ID
jmeter,application=gitlab,statut=ok,transaction=HTTP请求
jmeter,application=gitlab,statut=ok,transaction=创建用户
jmeter,application=gitlab,statut=ok,transaction=删除用户
jmeter,application=gitlab,statut=ok,transaction=查询用户ID
jmeter,application=gitlab,transaction=internal
```

field keys 可以理解为表中的列，每一个 key 其中都存储了相应数据，类型由 fieldType 标识。

```
$ docker exec -it collector_db_1 influx --database 'jmeter' -execute 'show field keys'
name: events
fieldKey fieldType
-------- ---------
text     string

name: jmeter
fieldKey   fieldType
--------   ---------
avg        float
count      float
countError float
endedT     float
hit        float
max        float
maxAT      float
meanAT     float
min        float
minAT      float
pct90.0    float
pct95.0    float
pct99.0    float
startedT   float
```

- events
  - text, ？？
- jmeter
  - avg, 平均响应时间
  - count, 请求数量
  - countError, 请求失败数量
  - hit, 服务器每秒hit数
  - max, 最大响应时间
  - min, 最小响应时间
  - maxAT, Max active threads
  - minAT, Min active threads
  - meanAT, Mean active threads
  - startedT, Started threads
  - endedT, Finished threads
  - pct90.0, 90%响应时间，意思是所有有90%的响应都可以在此时间内完成
  - pct95.0, 类上
  - pct99.0, 类上

##### config grafana

理解了数据结构之后，再来配置grafana就简单一些。

![](./grafana-config.png)

- db 指的是所连接的 influxdb 相应的数据库，这里是 jmeter，db 是别名
- jmeter 指定的是表名
- where 标明了 series 的限定条件，这里使用了3个条件，才完全指定了一个 series
- select 标明了 field keys 的 key 的取值
- group by 一般不用

这样就可以 query 到自己需要的数据了。以下所有的数据都是如法炮制，
关键在于先解读其中的数据结构。


#### graphite way

jmeter 生成 graphite 类型的数据，传输至 2003 端口。

![](./jmeter-graphite-listener.png)

其中配置项的含义，参考 [influxdb](#influxdb-way)。


在收集数据之后，[官方][jmeter graphite metric]对相应数据的含义作出了解读。

rootMetricsPrefix， samplerName, percentileValue 是变量
- rootMetricsPrefix 是在配置项中定义的前缀
- samplerName 是 jmeter sampler 的名称 （取样器），包括一个特殊的 all （全部 sampler）
- percentileValue 是配置项中定义的相应百分比的值，默认是 90，95，99

- rootMetricsPrefix.test.minAT, Min active threads
- rootMetricsPrefix.test.maxAT, Max active threads
- rootMetricsPrefix.test.meanAT, Mean active threads
- rootMetricsPrefix.test.startedT, Started threads
- rootMetricsPrefix.test.endedT, Finished threads 
- rootMetricsPrefix.samplerName.ok.count, Number of successful responses for sampler name
- rootMetricsPrefix.samplerName.h.count, Server hits per seconds, this metric cumulates Sample Result and Sub results (if using Transaction Controller, "Generate parent sampler" should be unchecked)
- rootMetricsPrefix.samplerName.ok.min, Min response time for successful responses of sampler name
- rootMetricsPrefix.samplerName.ok.max, Max response time for successful responses of sampler name
- rootMetricsPrefix.samplerName.ok.avg, Average response time for successful responses of sampler name.
- rootMetricsPrefix.samplerName.ok.pct *percentileValue* , Percentile computed for successful responses of sampler name. There will be one metric for each calculated value.
- rootMetricsPrefix.samplerName.ko.count, Number of failed responses for sampler name
- rootMetricsPrefix.samplerName.ko.min, Min response time for failed responses of sampler name
- rootMetricsPrefix.samplerName.ko.max, Max response time for failed responses of sampler name
- rootMetricsPrefix.samplerName.ko.avg, Average response time for failed responses of sampler name.
- rootMetricsPrefix.samplerName.ko.pct *percentileValue* , Percentile computed for failed responses of sampler name. There will be one metric for each calculated value.
- rootMetricsPrefix.samplerName.a.count, Number of responses for sampler name (sum of ok.count and ko.count)
- rootMetricsPrefix.samplerName.a.min, Min response time for responses of sampler name (min of ok.count and ko.count)
- rootMetricsPrefix.samplerName.a.max, Max response time for responses of sampler name (max of ok.count and ko.count)
- rootMetricsPrefix.samplerName.a.avg, Average response time for responses of sampler name (avg of ok.count and ko.count)
- rootMetricsPrefix.samplerName.a.pct *percentileValue* , Percentile computed for responses of sampler name. There will be one metric for each calculated value. (calculated on the totals for OK and failed samples) 

实际从 influxdb 来看，数据结构是这样的。

```
$ docker exec -it collector_db_1 influx --database 'graphite' -execute 'show field keys'
name: jmeter.test.endedT
fieldKey fieldType
-------- ---------
value    float

name: jmeter.test.maxAT
fieldKey fieldType
-------- ---------
value    float

name: jmeter.test.meanAT
fieldKey fieldType
-------- ---------
value    float

name: jmeter.test.minAT
fieldKey fieldType
-------- ---------
value    float

name: jmeter.test.startedT
fieldKey fieldType
-------- ---------
value    float

name: jmeter.HTTP请求.a.avg
fieldKey fieldType
-------- ---------
value    float

name: jmeter.HTTP请求.a.count
fieldKey fieldType
-------- ---------
value    float

name: jmeter.HTTP请求.a.max
fieldKey fieldType
-------- ---------
value    float

name: jmeter.HTTP请求.a.min
fieldKey fieldType
-------- ---------
value    float

name: jmeter.HTTP请求.a.pct90
fieldKey fieldType
-------- ---------
value    float

name: jmeter.HTTP请求.a.pct95
fieldKey fieldType
-------- ---------
value    float

name: jmeter.HTTP请求.a.pct99
fieldKey fieldType
-------- ---------
value    float

name: jmeter.HTTP请求.h.count
fieldKey fieldType
-------- ---------
value    float

name: jmeter.HTTP请求.ko.avg
fieldKey fieldType
-------- ---------
value    float

name: jmeter.HTTP请求.ko.count
fieldKey fieldType
-------- ---------
value    float

name: jmeter.HTTP请求.ko.max
fieldKey fieldType
-------- ---------
value    float

name: jmeter.HTTP请求.ko.min
fieldKey fieldType
-------- ---------
value    float

name: jmeter.HTTP请求.ko.pct90
fieldKey fieldType
-------- ---------
value    float

name: jmeter.HTTP请求.ko.pct95
fieldKey fieldType
-------- ---------
value    float

name: jmeter.HTTP请求.ko.pct99
fieldKey fieldType
-------- ---------
value    float

name: jmeter.HTTP请求.ok.avg
fieldKey fieldType
-------- ---------
value    float

name: jmeter.HTTP请求.ok.count
fieldKey fieldType
-------- ---------
value    float

name: jmeter.HTTP请求.ok.max
fieldKey fieldType
-------- ---------
value    float

name: jmeter.HTTP请求.ok.min
fieldKey fieldType
-------- ---------
value    float

name: jmeter.HTTP请求.ok.pct90
fieldKey fieldType
-------- ---------
value    float

name: jmeter.HTTP请求.ok.pct95
fieldKey fieldType
-------- ---------
value    float

name: jmeter.HTTP请求.ok.pct99
fieldKey fieldType
-------- ---------
value    float

```


### cadvisor metric


收集到的数据，结构如下

```
$ docker exec -it collector_db_1 influx --database 'cadvisor' -execute 'show field keys'
name: cpu_usage_per_cpu
fieldKey fieldType
-------- ---------
value    integer

name: cpu_usage_system
fieldKey fieldType
-------- ---------
value    integer

name: cpu_usage_total
fieldKey fieldType
-------- ---------
value    integer

name: cpu_usage_user
fieldKey fieldType
-------- ---------
value    integer

name: fs_limit
fieldKey fieldType
-------- ---------
value    integer

name: fs_usage
fieldKey fieldType
-------- ---------
value    integer

name: load_average
fieldKey fieldType
-------- ---------
value    integer

name: memory_usage
fieldKey fieldType
-------- ---------
value    integer

name: memory_working_set
fieldKey fieldType
-------- ---------
value    integer

name: rx_bytes
fieldKey fieldType
-------- ---------
value    integer

name: rx_errors
fieldKey fieldType
-------- ---------
value    integer

name: tx_bytes
fieldKey fieldType
-------- ---------
value    integer

name: tx_errors
fieldKey fieldType
-------- ---------
value    integer

```

对于每一项指标，并没有完全官方的解读，只能参考 [docker stat][docker stat] 与 [源代码][cadvisor influxdb code]

- cpu_usage_per_cpu
- cpu_usage_system
- cpu_usage_total
- cpu_usage_user
- fs_limit
- fs_usage
- load_average
- memory_usage
- memory_working_set
- rx_bytes, 收到网络字节数
- rx_errors，
- tx_bytes， 发出网络字节数
- tx_errors

### gitlab metric


相应数据结构如下

```
$ docker exec -it collector_db_1 influx --database 'gitlab' -execute 'show field keys'
name: events
fieldKey fieldType
-------- ---------
count    float

name: rails_file_descriptors
fieldKey fieldType
-------- ---------
value    float

name: rails_gc_statistics
fieldKey                                fieldType
--------                                ---------
count                                   float
heap_allocatable_pages                  float
heap_allocated_pages                    float
heap_available_slots                    float
heap_eden_pages                         float
heap_final_slots                        float
heap_free_slots                         float
heap_live_slots                         float
heap_marked_slots                       float
heap_sorted_length                      float
heap_swept_slots                        float
heap_tomb_pages                         float
major_gc_count                          float
malloc_increase_bytes                   float
malloc_increase_bytes_limit             float
minor_gc_count                          float
old_objects                             float
old_objects_limit                       float
oldmalloc_increase_bytes                float
oldmalloc_increase_bytes_limit          float
remembered_wb_unprotected_objects       float
remembered_wb_unprotected_objects_limit float
total_allocated_objects                 float
total_allocated_pages                   float
total_freed_objects                     float
total_freed_pages                       float
total_time                              float

name: rails_memory_usage
fieldKey fieldType
-------- ---------
value    float

name: rails_method_calls
fieldKey     fieldType
--------     ---------
call_count   float
cpu_duration float
duration     float

name: rails_object_counts
fieldKey fieldType
-------- ---------
count    float

name: rails_transactions
fieldKey                           fieldType
--------                           ---------
allocated_memory                   float
banzai_cacheless_render_call_count float
banzai_cacheless_render_cpu_time   float
banzai_cacheless_render_real_time  float
cache_count                        float
cache_duration                     float
cache_read_count                   float
cache_read_duration                float
cache_read_hit_count               float
cache_read_miss_count              float
cache_write_count                  float
cache_write_duration               float
duration                           float
new_redis_connections              float
rails_queue_duration               float
request_method                     string
request_uri                        string
sql_count                          float
sql_duration                       float
view_duration                      float

name: rails_views
fieldKey fieldType
-------- ---------
duration float

name: sidekiq_file_descriptors
fieldKey fieldType
-------- ---------
value    float

name: sidekiq_gc_statistics
fieldKey                                fieldType
--------                                ---------
count                                   float
heap_allocatable_pages                  float
heap_allocated_pages                    float
heap_available_slots                    float
heap_eden_pages                         float
heap_final_slots                        float
heap_free_slots                         float
heap_live_slots                         float
heap_marked_slots                       float
heap_sorted_length                      float
heap_swept_slots                        float
heap_tomb_pages                         float
major_gc_count                          float
malloc_increase_bytes                   float
malloc_increase_bytes_limit             float
minor_gc_count                          float
old_objects                             float
old_objects_limit                       float
oldmalloc_increase_bytes                float
oldmalloc_increase_bytes_limit          float
remembered_wb_unprotected_objects       float
remembered_wb_unprotected_objects_limit float
total_allocated_objects                 float
total_allocated_pages                   float
total_freed_objects                     float
total_freed_pages                       float
total_time                              float

name: sidekiq_memory_usage
fieldKey fieldType
-------- ---------
value    float

name: sidekiq_method_calls
fieldKey     fieldType
--------     ---------
call_count   float
cpu_duration float
duration     float

name: sidekiq_object_counts
fieldKey fieldType
-------- ---------
count    float

name: sidekiq_transactions
fieldKey                           fieldType
--------                           ---------
allocated_memory                   float
banzai_cacheless_render_call_count float
banzai_cacheless_render_cpu_time   float
banzai_cacheless_render_real_time  float
cache_count                        float
cache_duration                     float
cache_read_count                   float
cache_read_duration                float
cache_read_hit_count               float
duration                           float
new_redis_connections              float
sidekiq_queue_duration             float
sql_count                          float
sql_duration                       float
view_duration                      float

name: sidekiq_views
fieldKey fieldType
-------- ---------
duration float

```

gitlab 的[官方文档][gitlab metric influxdb]描述的很清楚

- PROCESS_file_descriptors
  - 打开的文件描述符数量
- PROCESS_gc_statistics
  - 涉及ruby虚拟机中gc相关的数据
- PROCESS_memory_usage
  - 进程内存使用量，单位byte
- PROCESS_method_calls，方法调用数据统计
  - call_count，调用次数
  - cpu_duration，消耗cpu时间
  - duration, 方法调用消耗时间
- PROCESS_object_counts
  - 对class的对象计数，涉及ruby底层
- PROCESS_transactions
  - duration，一次transcation消耗的时间
  - allocated_memory, bytes，一次transcation中新分配的内存
  - method_duration，在method calls中耗费的时间
  - sql_duration，在sql查询中耗费的时间
  - view_duration, 生成view的时间
- PROCESS_views
  - duration, view渲染花费的时间
- events, 统计事件发生的次数


## grafana

grafana 也使用 docker 部署，参考 [docker hub][docker hub grafana]。

grafana 如何制定自己的数据呈现规则，参考 [config grafana](#config-grafana) 一节，
关键在于解读 influxdb 的数据结构。



[influxdb docker]: https://hub.docker.com/_/influxdb/
[jmeter metric influxdb]: http://jmeter.apache.org/usermanual/realtime-results.html
[jmeter backend listener config]: http://jmeter.apache.org/usermanual/component_reference.html#Backend_Listener
[jmeter graphite metric]: http://jmeter.apache.org/usermanual/realtime-results.html#metrics-response-times
[docker stat]: https://docs.docker.com/engine/reference/commandline/stats/#examples
[cadvisor influxdb code]: https://github.com/google/cadvisor/blob/c094ef0d2a3de380d516ff26bca13d2585ffe58f/storage/influxdb/influxdb.go#L52
[gitlab metric influxdb]: https://docs.gitlab.com/ee/administration/monitoring/performance/influxdb_schema.html
[docker hub grafana]: https://hub.docker.com/r/grafana/grafana/
