---
layout: post
title: "InfluxDB基础概念"
date: 2018-12-21 15:49:00 +0800
categories: influxdb
---
请看以下示例数据。

name: census

| time                 | butterflies   | honeybees | location | scientist  |
| -------------------- | ------------- | --------- | -------- | ---------- |
| 2015-08-18T00:00:00Z | 12            | 23        | 1        | langstroth |
| 2015-08-18T00:00:00Z | 1             | 30        | 1        | perpetua   |
| 2015-08-18T00:06:00Z | 11            | 28        | 1        | langstroth |
| 2015-08-18T00:06:00Z | 3             | 28        | 1        | perpetua   |
| 2015-08-18T05:54:00Z | 2             | 11        | 2        | langstroth |
| 2015-08-18T06:00:00Z | 1             | 10        | 2        | langstroth |
| 2015-08-18T06:06:00Z | 8             | 23        | 2        | perpetua   |
| 2015-08-18T06:12:00Z | 7             | 22        | 2        | perpetua   |

## time
InfluxDB是时序数据库，所有的数据都带有timestamp。

## field

field是我们要记录的主要数据，由field key和field value组成。上表中，列butterflies和honeybees都是field key，对应的数字即为该field在某个时间的value。

* field value可以是string, float, integer, 或boolean类型
* field value未被索引，对field value的查询必须遍历所有field value值，因此性能较差。
* field是InfluxDB中的基础，没有field就无法存储数据。

拥有同样timestamp的一系列field key-value对构成一个field set，例如butterflies = 12 honeybees = 23就是一个field set

### 实例
在performance测试中，cpu idle可以作为一个field。

### 操作
要查看所有的field key：

```
SHOW FIELD KEYS [ON <database_name>] [FROM <measurement_name>]
```

## tag
tag是给field打上的标签，存储一些额外的信息。上表中，列location和scientist均为tag key，分别表示记录数据的位置和记录者，其对应的值为tag value。

* field是必须的，tag是可选的
* tag value为string类型
* tag有索引，适合搜索。

tag键值对的不同组合，构成一个个tag field：

* location = 1, scientist = langstroth
* location = 2, scientist = langstroth
* location = 1, scientist = perpetua
* location = 2, scientist = perpetua

### 实例
在performance测试中，job id可以作为一个tag。

### 操作
要查看所有的tag key：

```
SHOW TAG KEYS [ON <database_name>] [FROM_clause] [WHERE <tag_key> <operator> ['<tag_value>' | <regular_expression>]] [LIMIT_clause] [OFFSET_clause]
```

## measurement
measurement是tag, field, time的容器，概念上类似table。measurement name是对于相关数据的一个描述。

例如上表中，census是一个measurement，它包含两个field(butterflies和honeybees)、两个tag(location和scientist)。

### 实例
在performance测试中，存在一个叫做cpu的measurement，它有4个field：user, idle, iowait, system。

### 操作
列出所有measurements：
```
show measurements
```

要查看一个measurement中所有field key：

```
show field keys from ads;
```

## retention policy
一个measurement可以属于若干retention policy。retention policy表示数据应该保留多久(duration)、在集群里复制多少份(replication)。

measurement默认属于名为autogen的retention policy：数据永久保留；只复制一份。

## series
一系列拥有相同的retention policy、measurement、tag set的数据，称为一个series。

在上表中，所有retention policy: autogen, measurement: census, tag set: location = 1,scientist = langstroth的数据称为一个series。

### 操作
删除series：

```
DROP SERIES FROM <measurement_name[,measurement_name]> WHERE <tag_key>='<tag_value>'
```
