# Synch

![pypi](https://img.shields.io/pypi/v/synch.svg?style=flat)
![docker](https://img.shields.io/docker/cloud/build/long2ice/synch)
![license](https://img.shields.io/github/license/long2ice/synch)
![workflows](https://github.com/long2ice/synch/workflows/pypi/badge.svg)
![workflows](https://github.com/long2ice/synch/workflows/ci/badge.svg)

[中文文档](https://github.com/long2ice/synch/blob/dev/README-zh.md)

## Introduction

Sync data from other DB to ClickHouse, current support postgres and mysql, and support full and increment ETL.

![synch](https://github.com/long2ice/synch/raw/dev/images/synch.png)

## Features

- Full data etl and real time increment etl.
- Support DDL and DML sync, current support `add column` and `drop column` and `change column` of DDL, and full support of DML also.
- Custom configurable items.
- Support kafka and redis as broker.
- Multiple source db sync to ClickHouse at the same time。

## Requirements

- [redis](https://redis.io), cache mysql binlog file and position and as broker, support redis cluster also.
- [kafka](https://kafka.apache.org), need if you use kafka as broker.
- [clickhouse-jdbc-bridge](https://github.com/ClickHouse/clickhouse-jdbc-bridge), need if you use postgres and set `auto_full_etl = true`, or exec `synch etl` command.
- [sentry](https://github.com/getsentry/sentry), error reporting, worked if set `dsn` in config.

## Install

```shell
> pip install synch
```

## Usage

### Config

synch will read default config from `./synch.yaml`, or you can use `synch -c` specify config file.

See full example config in [`synch.yaml`](https://github.com/long2ice/synch/blob/dev/synch.yaml).

### Full data etl

Maybe you need make full data etl before continuous sync data from MySQL to ClickHouse or redo data etl with `--renew`.

```shell
> synch etl -h

usage: synch etl [-h] --schema SCHEMA [--tables TABLES] [--renew] [--partition-by PARTITION_BY] [--settings SETTINGS] [--engine ENGINE]

optional arguments:
  -h, --help            show this help message and exit
  --schema SCHEMA       Schema to full etl.
  --tables TABLES       Tables to full etl, multiple tables split with comma.
  --renew               Etl after try to drop the target tables.
  --partition-by PARTITION_BY
                        Table create partitioning by, like toYYYYMM(created_at).
  --settings SETTINGS   Table create settings, like index_granularity=8192
  --engine ENGINE       Table create engine, default MergeTree.

```

Full etl from table `test.test`:

```shell
> synch etl --schema test --tables test
```

### Produce

Listen all MySQL binlog and produce to broker.

```shell
> synch produce
```

### Consume

Consume message from broker and insert to ClickHouse,and you can skip error rows with `--skip-error`. And synch will do full etl at first when set `auto_full_etl = true` in config.

```shell
> synch consume -h

usage: synch consume [-h] --schema SCHEMA [--skip-error] [--last-msg-id LAST_MSG_ID]

optional arguments:
  -h, --help            show this help message and exit
  --schema SCHEMA       Schema to consume.
  --skip-error          Skip error rows.
  --last-msg-id LAST_MSG_ID
                        Redis stream last msg id or kafka msg offset, depend on broker_type in config.
```

Consume schema `test` and insert into `ClickHouse`:

```shell
> synch consume --schema test
```

**One consumer consume one schema**

### ClickHouse Table Engine

Now synch support `MergeTree` and `CollapsingMergeTree`, and performance of `CollapsingMergeTree` is higher than `MergeTree`.

Default you should choice `MergeTree`, and if you pursue a high performance or your database is frequently updated, you can choice `CollapsingMergeTree`, but your `select` sql query should rewrite. More detail see at [CollapsingMergeTree](https://clickhouse.tech/docs/zh/engines/table-engines/mergetree-family/collapsingmergetree/).

## Use docker-compose(recommended)

<details>
<summary>Redis Broker, lightweight and for low concurrency</summary>

```yaml
version: "3"
services:
  producer:
    depends_on:
      - redis
    image: long2ice/synch
    command: synch produce
    volumes:
      - ./synch.yaml:/synch/synch.yaml
  # one service consume on schema
  consumer.test:
    depends_on:
      - redis
    image: long2ice/synch
    command: synch consume --schema test
    volumes:
      - ./synch.yaml:/synch/synch.yaml
  redis:
    hostname: redis
    image: redis:latest
    volumes:
      - redis
volumes:
  redis:
```

</details>

<details>
<summary>Kafka Broker, for high concurrency</summary>

```yaml
version: "3"
services:
  zookeeper:
    image: bitnami/zookeeper:3
    hostname: zookeeper
    environment:
      - ALLOW_ANONYMOUS_LOGIN=yes
    volumes:
      - zookeeper:/bitnami
  kafka:
    image: bitnami/kafka:2
    hostname: kafka
    environment:
      - KAFKA_CFG_ZOOKEEPER_CONNECT=zookeeper:2181
      - ALLOW_PLAINTEXT_LISTENER=yes
      - JMX_PORT=23456
      - KAFKA_CFG_AUTO_CREATE_TOPICS_ENABLE=true
      - KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092
    depends_on:
      - zookeeper
    volumes:
      - kafka:/bitnami
  kafka-manager:
    image: hlebalbau/kafka-manager
    ports:
      - "9000:9000"
    environment:
      ZK_HOSTS: "zookeeper:2181"
      KAFKA_MANAGER_AUTH_ENABLED: "false"
    command: -Dpidfile.path=/dev/null
  producer:
    depends_on:
      - redis
      - kafka
      - zookeeper
    image: long2ice/synch
    command: synch produce
    volumes:
      - ./synch.yaml:/synch/synch.yaml
  # one service consume on schema
  consumer.test:
    depends_on:
      - redis
      - kafka
      - zookeeper
    image: long2ice/synch
    command: synch consume --schema test
    volumes:
      - ./synch.yaml:/synch/synch.yaml
  redis:
    hostname: redis
    image: redis:latest
    volumes:
      - redis:/data
volumes:
  redis:
  kafka:
  zookeeper:
```

</details>

## Important

- Synch don't support composite primary key, and you need always keep a primary key or unique key.
- DDL sync not support postgres.
- Postgres sync is not fully test, be careful use it in production.

## QQ Group

<img width="200" src="https://github.com/long2ice/synch/raw/dev/images/qq_group.png"/>

## Support this project

- Just give a star!
- Join QQ group for communication.
- Donation.

### AliPay

<img width="200" src="https://github.com/long2ice/synch/raw/dev/images/alipay.jpeg"/>

### WeChat Pay

<img width="200" src="https://github.com/long2ice/synch/raw/dev/images/wechatpay.jpeg"/>

### PayPal

Donate money by [paypal](https://www.paypal.me/long2ice) to my account long2ice.

## ThanksTo

Powerful Python IDE [Pycharm](https://www.jetbrains.com/pycharm/?from=synch) from [Jetbrains](https://www.jetbrains.com/?from=synch).

![jetbrains](https://github.com/long2ice/synch/raw/dev/images/jetbrains.svg)

## License

This project is licensed under the [Apache-2.0](https://github.com/long2ice/synch/blob/master/LICENSE) License.
