---
layout: post
title: kafka和clickhouse技术相关
date: 2021-01-13
Author: 来自veryfirefly
categories: [kafka,clickhouse]
tags: [kafka,clickhouse,知识盘点]
comments: true
---

# 目录
* [使用kafka将数据写入到clickhouse](#使用kafka将数据写入到clickhouse)
  * [创建示例表SQL语句](#创建示例表sql语句)
  * [虚拟列](#虚拟列)
  * [可调整的配置选项](#可调整的配置选项)
    * [从KafkaSettings\.h 头文件中已知的kafka可用设置和设置参数类型](#从kafkasettingsh-头文件中已知的kafka可用设置和设置参数类型)
    * [暂时没捋清](#暂时没捋清)
* [Confluent Kafka REST Proxy APIs](#confluent-kafka-rest-proxy-apis)
  * [特征](#特征)
  * [API 接口参考](#api-接口参考)
    * [Content Types](#content-types)
    * [Errors](#errors)
    * [REST Proxy API v2](#rest-proxy-api-v2)
      * [Topic](#topic)
        * [获取topic列表](#获取topic列表)
        * [获取指定topic的元数据](#获取指定topic的元数据)
        * [★发送消息到topic](#发送消息到topic)
      * [Partitions](#partitions)
        * [获取该topic的partitions](#获取该topic的partitions)
        * [获取该topic的单个partitions的元数据](#获取该topic的单个partitions的元数据)
        * [获取该topic中指定partitions的offset摘要](#获取该topic中指定partitions的offset摘要)
        * [★产生消息到topic的指定分区](#产生消息到topic的指定分区)
      * [Consumer](#consumer)
# 使用kafka将数据写入到clickhouse

kafka engine设计用于一次性数据检索。这意味着一旦从kafka表中查询了数据，就将其视为已从队列中使用。因此，不要直接从kafka引擎表中获取数据，而应使用物化视图。一旦数据在kafka表中可用，就会触发物化视图。它会自动将数据从kafka表移至到MergeTree或Distributed Engine表中。因此，你至少需要三个表：

- **源数据表 (kafka engine表)**
- **目标表 (MergeTree系列或Distributed系列表)**
- **物化视图以写入数据**

一个kafka表可以具有任意数量的物化视图，它们不直接从kafka表中读取数据，而是接受新纪录（以Blocks为单位），这样就可以写入具有不同操作级别的多个表（通过分组 **[grouping]** - 聚合 **[aggregation]**）。

## 创建示例表SQL语句

> 使用kafka写入数据到clickhouse中有一定**<font color="#C02C38">延时(目前使用在30秒内)</font>**
>
> 如要更改kafka表，你需要删除并重新创建kafka表。ALTER TABLE MODIFY SETTINGS for Kafka engine tables is planned for later in 2020. (原话)

``` sql
-- 1.创建kafka engine表
CREATE TABLE kafka_queue
(
	Timestamp	UInt64	Comment '时间戳',
	Name		String	Comment	'用户名',
	Level		UInt16	Comment '等级',
	Message		String	Comment '消息'
) ENGINE = Kafka
	SETTINGS kafka_broker_list = 'localhost:9092',
		kafka_topic_list = 'topic',		-- 消费主题
		kafka_group_name = 'group',		-- 消费组
		kafka_format = 'JSONEachRow',	-- 如何格式化数据
		kafka_num_consumers = 1;		-- 启动几个Consumer

-- 2.创建目标表
CREATE TABLE merge_tree_queue
(
	Timestamp	UInt64		Comment '时间戳',
	CreateTime	DateTime	Comment	'创建时间',
	CreateDate	Date		Comment	'创建日期',
	Name		String		Comment	'用户名',
	Level		UInt16		Comment	'等级',
	Message		String		Comment	'消息'
) ENGINE = MergeTree()
	PARTITION BY CreateDate
	ORDER BY (CreateDate, Timestamp);
	
-- 3.创建物化视图
CREATE MATERIALIZED VIEW kafka_to_merge_tree_queue_mv TO merge_tree_queue
AS
-- 定义好的触发规则
SELECT
	Timestamp,
	toDateTime(Timestamp) AS CreateTime,
	toDate(Timestamp) AS CreateDate,
	Name,
	Level,
	Message
FROM
	kafka_queue
```

## 虚拟列

<table>
	<tr>
		<td>字段名</td>
		<td>对应类型</td>
	</tr>
	<tr>
		<td>_topic</td>
		<td>String</td>
	</tr>
	<tr>
		<td>_key</td>
		<td>String</td>
	</tr>
	<tr>
		<td>_offset</td>
		<td>UInt64</td>
	</tr>
	<tr>
		<td>_partition</td>
		<td>UInt64</td>
	</tr>
	<tr>
		<td>_timestamp</td>
		<td>Nullable(DateTime)</td>
	</tr>
</table>
## 手动启停kafka-table-consumer

要停止接收topic的数据或更改转换逻辑，请停止kafka表以分离materialized view：

``` sql
DETACH TABLE kafka_table; -- 停止kafka table consumer
ATTACH TABLE kafka_table; -- 启动kafka table consumer
```

## 可调整的配置选项

- kafka_broker_list  -  以逗号分隔的broker信息
- kafka_group_name  -  一群kafka consumer。分别跟踪每个group的offset。
- kafka_topic_list  -  以逗号分隔的topic信息
- kafka_max_block_size (默认值65536)  -  在表级别配置的consumer#poll()最大批处理的将块提交到clickhouse的阈值（以行数为单位）
- kafka_skip_broken_messages  -  在表级别配置的解析消息时允许的错误数
- stream_flush_interval_ms (默认值为7500)  -  在用户配置文件级别配置的将块提交到ClickHouse的阈值 (以毫秒为单位), 也可能会影响到其他stream表
- kafka_max_wait_ms  -  在用户配置文件级别配置的确认消息等待的超时
- kafka_num_consumers (默认值为0)  -  每张kafka表的消费者数量, 如果一个consumer的吞吐量处理能力不足，请指定更多consumer，consumer的总数不应超过该topic中的partition，因为每个partition只能分配一个consumer。
- kafka_commit_every_batch (默认值为0)  -  写入整个块后，提交每个消耗和处理的批次，而不是单个提交。
- kafka_thread_per_consumer (默认值为0)  -  为每个consumer提供独立的线程。启用后，每个consumer将并行的刷新数据（否则来自多个consumer的行将被压缩成一个块）。
- kafka_row_delimiter  -  分隔符，结束消息。
- kafka_format  -  消息格式。使用与SQL Format函数。[格式](https://clickhouse.tech/docs/en/interfaces/formats/)


### 从KafkaSettings.h 头文件中已知的kafka可用设置和设置参数类型

<table>
	<tr>
		<td>字段名</td>
		<td>对应类型</td>
		<td>对应解释</td>
	</tr>
	<tr>
		<td>kafka_broker_list</td>
		<td>String</td>
		<td>A comma-separated list of brokers for Kafka engine.</td>
	</tr>
	<tr>
		<td>kafka_topic_list</td>
		<td>String</td>
		<td>A list of Kafka topics.</td>
	</tr>
	<tr>
		<td>kafka_group_name</td>
		<td>String</td>
		<td>Client group id string. All Kafka consumers sharing the same group.id belong to the same group.</td>
	</tr>
	<tr>
		<td>kafka_client_id</td>
		<td>String</td>
		<td>Client identifier.</td>
	</tr>
	<tr>
		<td>kafka_num_consumers</td>
		<td>UInt64</td>
		<td>The number of consumers per table for Kafka engine.</td>
	</tr>
	<tr>
		<td>kafka_commit_every_batch</td>
		<td>Bool</td>
		<td>Commit every consumed and handled batch instead of a single commit after writing a whole block</td>
	</tr>
	<tr>
		<td>kafka_poll_timeout_ms</td>
		<td>Milliseconds</td>
		<td>Timeout for single poll from Kafka.</td>
	</tr>
	<tr>
		<td>kafka_poll_max_batch_size</td>
		<td>UInt64</td>
		<td>Maximum amount of messages to be polled in a single Kafka poll.</td>
	</tr>
	<tr>
		<td>kafka_max_block_size</td>
		<td>UInt64</td>
		<td>Number of row collected by poll(s) for flushing data from Kafka.</td>
	</tr>
	<tr>
		<td>kafka_flush_interval_ms</td>
		<td>Milliseconds</td>
		<td>Timeout for flushing data from Kafka.</td>
	</tr>
	<tr>
		<td>kafka_format</td>
		<td>String</td>
		<td>The message format for Kafka engine.</td>
	</tr>
	<tr>
		<td>kafka_row_delimiter</td>
		<td>Char</td>
		<td>The character to be considered as a delimiter in Kafka message.</td>
	</tr>
	<tr>
		<td>kafka_schema</td>
		<td>String</td>
		<td>Schema identifier (used by schema-based formats) for Kafka engine</td>
	</tr>
	<tr>
		<td>kafka_kafka_skip_broken_messagesformat</td>
		<td>UInt64</td>
		<td>Skip at least this number of broken messages from Kafka topic per block</td>
	</tr>
	<tr>
		<td>kafka_thread_per_consumer</td>
		<td>Bool</td>
		<td>Provide independent thread for each consumer</td>
	</tr>
</table>


### 暂时没捋清

<u>单个表的性能取决于行大小，使用的格式，每条消息的行数等。一个Kafka表通常每秒可以处理60K-300K条简单消息。</u>

<u>为了获得单个表的最佳性能，应将“ kafka_max_block_size”设置增加到值512K-1M。默认值为64K，该值太小。我们将在下一个版本中解决它。</u>

<u>如果多个服务器（副本服务器）或同一服务器上的多个Kafka引擎表占用一个主题，则可能会进一步改进。</u>

<u>在当前的实现中，应始终使用“ kafka_num_consumers = 1”，因为增加并不能带来任何改善，它目前被锁定在单个线程中。</u>

<u>取而代之的是，可以创建几个Kafka引擎表，再加上相应的实例化视图，以将数据移至同一目标表。这样，每个Kafka引擎表将在单独的线程中工作。</u> 



<u>为了提高性能，将收到的消息分组为以下大小的Blocks：max_insert_block_size。如果块不是在stream_flush_interval_ms 毫秒内形成的，无论块的完整性如何，数据都将会刷新到表中。</u>



<u>数据导入的方式是通过 kafka table -> materialized view -> mergetree table 实现的，所以一个Kafka topic需要三张clickhouse的table来实现，维护负担略重，同时kafka table本质是一个consumer，数据被读取一次之后就不能重复读取，在配置Kafka topic table的时候，一定要注意用户与表权限的配置，否则万一有人在kafka table执行了select语句，那么这段数据就完全丢失了，不会写入到final table中的。</u>

<u>ClickHouse提供了集成kafka的解决方案，从而使在clickhouse中集成流数据变为可能，但是如果要使用在生产环境中，有以下几个方面的问题不得不考虑：</u>

- <u>消费Kafka数据的语义是at least once, 消费的数据会有重复</u>
- <u>维护的成本与job监控</u>
- <u>扩展性</u>







# Confluent Kafka REST Proxy APIs

Confluent REST Proxy为Apache Kafka®集群提供了RESTful接口，可以轻松生成和使用消息，查看集群状态以及执行管理操作，而无需使用本机Kafka Protocol或客户端。

## 特征

REST Proxy能够公开Java Producer、Consumer和命令行工具的所有功能。以下是当前支持的列表：

- 元数据  -  有关集群的大多数元数据（`brokers`、`topics`、`partitions`、和`configs`）可以使用`GET`对应URL的请求来读取。
- 生产者  -  API不会公开生产者对象，而是接受针对特定主题或分区的生产请求，并通过少量生产路由所有请求。
  - 生产者配置  -  生产者实例是共享的，因此不能基于每个请求设置配置。但是，您可以通过在REST代理配置中传递新的生产者设置来进行全局调整设置。例如，您可以传入`compression.type`选项以启用站点范围的压缩以减少存储和网络开销。
- 消费者  -  使用者是有状态的，因此与特定的REST Proxy instances相关联。Offset Commit可以是自动的，也可以由用户主动请求。当前限制为每个Consumer一个线程。使用多个Consumer以获得更高的吞吐量。~~v1 API已标记为不推荐使用~~。
- 数据格式  -  可以使用JSON、Base64编码的原始字节或使用JSON编码的Avro、Protobuf或JSON Schema读写数据。

## API 接口参考

### Content Types

REST Proxy将Content Type用于请求和响应(Accept)，以指示一下数据属性：

- 序列化格式：json
- API版本：v2、v3
- 嵌入的数据格式：json、binary、avro、protobuf和jsonschema

> 嵌入的数据格式是您正在生成或使用的数据格式。这些格式以序列化嵌入到请求或响应中。

Content类型的格式为：

```
application/vnd.kafka[.embedded_format].[api_version]+[serialization_format]
```

- Avro内容类型为：`application/vnd.kafka.avro.v2+json`
- Protobuf的内容类型为：`application/vnd.kafka.protobuf.v2+json`
- JSON Schema的内容类型为：`application/vnd.kafka.jsonschema.v2+json`
- JSON的内容类型为：`application/vnd.kafka.v2+json`

您的请求应通过HTTP `Accept` Header指定具体的返回格式和版本信息：

```
Accept: application/vnd.kafka.v2+json
```

REST Proxy还支持Content Type协商，因此您可以指定多个加权首选项，**<u>当首选新版本的API，但您不确定它是否可用时，此功能很有用</u>**：

```
Accept: application/vnd.kafka.v2+json; q=0.9, application/json; q=0.5
```

### Errors

所有API端点都是用标准的错误消息格式来处理返回指示错误的HTTP状态(任何4xx或5xx的状态)的任何请求。例如，省略必填字段的请求实体可能会生成以下响应：

```http
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/vnd.kafka.v1+json

{
	"error_code": 422,
	"message": "records may not be empty"
}
```

某些错误代码在整个API中经常使用，您可能需要通用代码来处理这些错误代码，而其他大多数错误代码则需要根据每个请求进行处理。


- 401  Unauthorized
  - Error code 40101 – Kafka Authentication Error.
- 403  Forbidden
  - Error code 40301 – Kafka Authorization Error. 
- 404  Not Found
  - **Error code 40401 – Topic not found.**
  - Error code 40402 – Partition not found.
- 422  **Unprocessable Entity**  -  The request payload is either improperly formatted or contains semantic errors（请求有效载荷的格式不正确或包含语义错误）
- 500 Internal Server Error
  - **Error code 50001 – Zookeeper error.** 
  - **Error code 50002 – Kafka error.** 
  - Error code 50003 – Retriable Kafka error. Although the operation failed, it’s possible that retrying the request will be successful. （可重试的kafka Error。尽管该操作失败，但是重试请求的操作可能还是会成功。）
  - Error code 50101 – Only SSL endpoints were found for the specified broker, but SSL is not supported for the invoked API yet. （仅找到了指定代理的SSL端点，但是调用的API尚不支持SSL）

### REST Proxy API v2

#### Topic

Topic Resources(指定路径资源) 提供有关Kafka群集中的主题及其当前状态的信息。它还允许您通过向特定主题发出POST请求来生成消息。

##### 获取topic列表

```http
GET /topics
```
响应JSON对象：
- topic(数组)  -  topic名称列表

Request：

``` http
GET /topics HTTP/1.1
Accept: application/vnd.kafka.v2+json;
```
Response：

``` http
HTTP/1.1 200 OK
Content-Type: application/vnd.kafka.v2+json

["topic1", "topic2"]
```

##### 获取指定topic的元数据

```http
GET /topics/(string:topic_name)
```
参数：

- topic_name（string）-  用于获取有关元数据的topic名称

响应JSON对象：

- **name**（*字符串*）– topic名称
- **configs**（*map*）– 按topic的配置覆盖
- **partitions**（*数组*）– 本topic的partition列表
  - **partitions [i] .partition**（*int*）– 该partition的ID
  - **partitions [i] .leader**（*int*）– 该partition的leader的broker ID
  - **partitions [i] .replicas**（*array*）– 该partition的副本列表
  - **partitions [i] .replicas [j] .broker**（*数组*）– 副本的broker ID
  - **partitions [i] .replicas [j] .leader**（*boolean*）– 如果此副本是partition的leader，则为true
  - **partitions [i] .replicas [j] .in_sync**（*boolean*）– 如果此副本当前与**引导**者同步，则为true

Request：

``` http
GET /topics/test HTTP/1.1
Accept: application/vnd.kafka.v2+json
```
Response：

``` http
HTTP/1.1 200 OK
Content-Type: application/vnd.kafka.v2+json

{
  "name": "test",
  "configs": {
     "cleanup.policy": "compact"
  },
  "partitions": [
    {
      "partition": 1,
      "leader": 1,
      "replicas": [
        {
          "broker": 1,
          "leader": true,
          "in_sync": true,
        },
        {
          "broker": 2,
          "leader": false,
          "in_sync": true,
        }
      ]
    },
    {
      "partition": 2,
      "leader": 2,
      "replicas": [
        {
          "broker": 1,
          "leader": false,
          "in_sync": true,
        },
        {
          "broker": 2,
          "leader": true,
          "in_sync": true,
        }
      ]
    }
  ]
}
```

##### ★发送消息到topic

**生产**(Produce)对应topic的消息，可以选择为消息指定key或partition。**<u>如果没有提供partition，将根据key的hash值选择一个partition。如果没有提供key，将以循环方式为每条消息选择topic。</u>**如果该topic未创建，则该接口会自动创建topic并将消息存入到topic中。

```http
POST /topics/(string:topic_name) HTTP/1.1
```
参数：

- topic_name （字符串） -  创建topic的名称

请求JSON对象数组：

- records  -  产生给该topic的record数组
  - records[i].key (object)  -  消息key，根据嵌入格式设置，或者为null以省略（可选）
  - records[i].value（object）-  消息值，根据嵌入格式设置
  - records[i].partition（int）-  存储消息的分区（可选） 

响应JSON对象：

- key_schema_id（int）-  key的格式ID（schema_registry）
- value_schema_id（int）-  value的格式ID（schema_registry）
- offsets（object）-  消息发布到的partition和offset列表
  - **offsets[i].partition**（*int*）–将消息发布到的分区，如果发布消息失败，则为null
  - **offsets[i].offset**（*long*）–消息的偏移量；如果发布消息失败，则为null
  - **offsets[i].error_code**（long）–  错误代码，分类了此操作失败的原因；如果成功，则返回null。
    - 1-不可重试的Kafka例外
    - 2-可重试的Kafka例外；重试该消息可能会成功发送
  - **offsets[i].error**（*string*）–描述操作失败原因的错误消息；如果成功，则返回null

错误码：

- 404 Not Found – 
  - Error code 40401 – Topic not found
- 422 Unprocessable Entity –  请求有效载荷的格式不正确或包含语义错误
  - Error code 42201 – Request includes keys and uses a format that requires schemas, but does not include the `key_schema` or `key_schema_id fields`
  - Error code 42202 – Request includes values and uses a format that requires schemas, but does not include the `value_schema` or `value_schema_id` fields
  - Error code 42205 – Request includes invalid schema.
- 408 Request Timeout – 请求超时
  - Error code 40801 – Schema registration or lookup failed.

Request：

``` http
POST /topics/test HTTP/1.1
Content-Type: application/vnd.kafka.json.v2+json
Accept: application/vnd.kafka.v2+json, application/vnd.kafka+json, application/json

{
  "records": [
    {
      "key": "somekey",
      "value": {"foo": "bar"}
    },
    {
      "value": [ "foo", "bar" ],
      "partition": 1
    },
    {
      "value": 53.5
    }
  ]
}
```
Response：

``` http
HTTP/1.1 200 OK
Content-Type: application/vnd.kafka.v2+json

{
  "key_schema_id": null,
  "value_schema_id": null,
  "offsets": [
    {
      "partition": 2,
      "offset": 100
    },
    {
      "partition": 1,
      "offset": 101
    },
    {
      "partition": 2,
      "offset": 102
    }
  ]
}
```

#### Partitions

Partitions Resources提供每个分区的元数据，包括每个分区的当前leader和副本。它还允许您使用GET和POST请求在单个分区中消费和生产消息。

##### 获取该topic的partitions

``` http
GET /topics/(string:topic_name)/partitions
```

参数：

- topic_name  -  topic名称

响应JSON对象：

- partition（int）-  partition的ID
- leader（int）-  此partition的broker ID
- **replicas**（array）-  该partition的副本broker列表
  - **replicas[i].broker**（int）-  副本的broker ID
  - **replicas[i].leader** (*boolean*)  -  如果此broker是partition的leader，则为true
  - **replicas[i].in_sync** (*boolean*)  -  如果此副本与leader同步，则为true

错误码：

- 404 Not Found
  - Error code 40401 – Topic not found

Request:

``` http
GET /topics/test/partition HTTP/1.1
Accept: application/vnd.kafka.v2+json, application/vnd.kafka+json, application/json
```

Response：

```http
HTTP/1.1 200 OK
Content-Type: application/vnd.kafka.v2+json

[
  {
    "partition": 1,
    "leader": 1,
    "replicas": [
      {
        "broker": 1,
        "leader": true,
        "in_sync": true,
      },
      {
        "broker": 2,
        "leader": false,
        "in_sync": true,
      },
      {
        "broker": 3,
        "leader": false,
        "in_sync": false,
      }
    ]
  },
  {
    "partition": 2,
    "leader": 2,
    "replicas": [
      {
        "broker": 1,
        "leader": false,
        "in_sync": true,
      },
      {
        "broker": 2,
        "leader": true,
        "in_sync": true,
      },
      {
        "broker": 3,
        "leader": false,
        "in_sync": false,
      }
    ]
  }
]
```

##### 获取该topic的单个partitions的元数据

``` http
GET /topics/(string:topic_name)/partitions/(int:partition_id)
```

参数：

- topic_name（string）-  topic名称
- partition_id（int）-  要获取的partition ID

响应JSON对象：

- partition（int）-  partition的ID
- leader（int）-  此partition的broker ID
- **replicas**（array）-  该partition的副本broker列表
  - **replicas[i].broker**（int）-  副本的broker ID
  - **replicas[i].leader** (*boolean*)  -  如果此broker是partition的leader，则为true
  - **replicas[i].in_sync** (*boolean*)  -  如果此副本与leader同步，则为true

错误码：

- 404 Not Found
  - Error code 40401 – Topic not found
  - Error code 40402 – Partition not found

Request:

``` http
GET /topics/test/partition/1 HTTP/1.1
Accept: application/vnd.kafka.v2+json, application/vnd.kafka+json, application/json
```

Response：

```http
HTTP/1.1 200 OK
Content-Type: application/vnd.kafka.v2+json

{
  "partition": 1,
  "leader": 1,
  "replicas": [
    {
      "broker": 1,
      "leader": true,
      "in_sync": true,
    },
    {
      "broker": 2,
      "leader": false,
      "in_sync": true,
    },
    {
      "broker": 3,
      "leader": false,
      "in_sync": false,
    }
  ]
}
```

##### 获取该topic中指定partitions的offset摘要

``` http
GET /topics/(string:topic_name)/partitions/(int:partition_id)/offsets
```

参数：

- topic_name（string）-  topic名称
- partition_id（int）-  要获取的partition ID

响应JSON对象：

- **beginning_offset** （int）-  该分区中的起始偏移量
- **end_offset**（int）-  该分区中的最后偏移量

错误码：

- 404 Not Found
  - Error code 40401 – Topic not found
  - Error code 40402 – Partition not found

Request:

``` http
GET /topics/test/partition/1/offsets HTTP/1.1
Accept: application/vnd.kafka.v2+json, application/vnd.kafka+json, application/json
```

Response：

```http
HTTP/1.1 200 OK
Content-Type: application/vnd.kafka.v2+json

{
  "beginning_offset": 10,
  "end_offset": 50,
}
```

##### ★产生消息到topic的指定分区

``` http
POST /topics/(string:topic_name)/partitions/(int:partition_id)
```

参数：

- topic_name（string）-  topic名称
- partition_id（int）-  要获取的partition ID

请求JSON对象：

- ~~key_schema~~（string）-  schema registry用不到，省略
- ~~key_schema_id~~（int）-  schema registry用不到，省略
- ~~value_schema~~（string）-  schema registry用不到，省略
- ~~value_schema_id~~（int）-  schema registry用不到，省略
- **records**  -  要产生到分区的消息列表
  - **records[i].key**（object）-  消息的key，根据嵌入格式设置，或者以null省略key（可选）
  - **records[i].value**（object）-  消息值，根据嵌入格式设置格式

响应JSON对象：

- **key_schema_id**（int）-  消息key的格式ID；如果未使用，则为null
- **value_schema_id**（int）- 消息value的格式ID；如果未使用，则为null
- **offsets**（object）-  消息发布到partition中的offset列表
  - **offsets[i].offset**（*long*）– 消息的偏移量
  - **offsets[i].error_code**（*long*）–  错误代码，分类了此操作失败的原因；如果成功，则返回null。
    - 1-不可重试的Kafka例外
    - 2-可重试的Kafka例外；重试该消息可能会成功发送
  - **offsets[i].error**（*string*）–描述操作失败原因的错误消息；如果成功，则返回null

错误码：

- 404 Not Found
  - Error code 40401 – Topic not found
  - Error code 40402 – Partition not found
- 422 Unprocessable Entity  -  无法处理的实体
  - Error code 42201 – Request includes keys and uses a format that requires schemas, but does not include the `key_schema` or `key_schema_id` fields
  - Error code 42202 – Request includes values and uses a format that requires schemas, but does not include the `value_schema` or `value_schema_id` fields
  - Error code 42205 – Request includes invalid schema.

Request:

``` http
POST /topics/test/partitions/1 HTTP/1.1
Content-Type: application/vnd.kafka.json.v2+json
Accept: application/vnd.kafka.v2+json, application/vnd.kafka+json, application/json

{
  "records": [
    {
      "key": "somekey",
      "value": {"foo": "bar"}
    },
    {
      "value": 53.5
    }
  ]
}
```

Response：

```http
HTTP/1.1 200 OK
Content-Type: application/vnd.kafka.v2+json

{
  "key_schema_id": null,
  "value_schema_id": null,
  "offsets": [
    {
      "partition": 1,
      "offset": 100,
    },
    {
      "partition": 1,
      "offset": 101,
    }
  ]
}
```

#### Consumer

消费者资源提供对消费者组当前状态的访问，允许您在消费者组中创建消费者，并使用来自主题和分区的消息。REST代理可以将存储在Kafka中的序列化数据转换为与JSON兼容的嵌入式格式。支持以下格式：

- 原始二进制数据被编码为base64的字符串
- Avro数据格式
- JSON对象
- Protobuf
- JSON Schema

因为使用者是有状态的，所以使用REST API创建的任何consumer实例都绑定到特定的REST Proxy实例中。在创建实例时需要提供一个完整的URL用于构造任何后续的请求。未能使用返回的URL进行将来的consumer请求将导致404错误，因为找不到consumer实例（response会返回一个consumer可用实例的URL）。如果REST Proxy实例关闭，它将尝试在终止之前彻底销毁任何使用者。