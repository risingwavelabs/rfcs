---
feature: Create Connection
authors:
  - "Bohan Zhang"
start_date: "2023/01/15"
---

# RFC: Create Connection

## Motivation

A connection is used to describe how to connect to an external system that users want to read data from. Once created, a connection is reusable across `CREATE SOURCE` and `CREATE SINK` statements. This RFC proposes a new `CREATE CONNECTION` statement to create a connection.

## Design

### Syntax

```sql
-- create connection
CREATE CONNECTION (IF NOT EXIST) {{ connection_name }}
  TO {{ connection_type }}
  ( {{ field }} = {{ value }}, ...);

-- create source with connection
CREATE SOURCE {{ source_name }} ( {{ field }} = {{ value }}, ... )
  FROM CONNECTION {{ connection_name }}
  with ( {{ field }} = {{ value }}, ...);

-- create sink with connection
CREATE SINK {{ sink_name }} ( {{ field }} = {{ value }}, ... )
  FROM {{ source_name }}
  INTO CONNECTION {{ connection_name }}
  with ( {{ field }} = {{ value }}, ...);
```

If any field is specified both in `with` clause and `CREATE CONNECTION` statement, the value in `with` clause will be used.

#### AWS Private Link

An AWS Private Link establishes a private connection between a VPC and a service hosted on AWS. It is a secure and scalable way to connect to AWS services from within a VPC. The following fields are required to create an AWS Private Link connection:

```sql
CREATE CONNECTION demo_connection TO AWS_PRIVATE_LINK
  (SERVICE NAME = 'com.amazonaws.vpce.us-east-1.vpce-svc-xxxxxxxxxxxxxxxxx',
  AVAILABILITY ZONES = array('us-east-1a', 'us-east-1b', 'us-east-1c');
```

|Field|Value|Required|Description|
|--|--|--|--|
|`SERVICE NAME`| varchar | yes | The name of the AWS PrivateLink service. |
|`AVAILABILITY ZONES`| varchar[] | yes | The AWS availability zones in which the service is accessible. |

> **Notice**:
>
> * After creating a connection, users must manually approve the connection from a Risingwave VPC in AWS console.
> * When creating a private link to AWS MSK, users must fill in all availability zones of the MSK cluster.

##### Internal table for AWS Private Link

The `rw_aws_private_link` table stores the information of the AWS Private Link connection. The following fields are stored in the table:

|Field|Value|Description|
|--|--|--|
|`id`| varchar | The name of the connection. |
|`principle`| varchar | The AWS Principal that Risingwave will use to connect to the VPC endpoint. |

#### Kafka

A Kafka connection establishes a link to a Kafka cluster. Users can use Kafka connections to create Kafka sources and sinks. The following fields are required to create a Kafka connection:

```sql
CREATE CONNECTION demo_connection_1 TO KAFKA
  (PROPERTIES.BOOTSTRAP.SERVER = 'broker1:9092,broker2:9092');

CREATE CONNECTION demo_connection_2 TO KAFKA
  (PROPERTIES.BOOTSTRAP.SERVER.PRIVATE_LINK = ['broker1:9092' USING AWS_PRIVATE_LINK {{ connection_name }} (PORT 9092), 'broker2:9092' USING AWS_PRIVATE_LINK {{ connection_name }} (PORT 9092)]];
```

For general usage, the following fields are allowed:

|Field|Value|Required|Description|
|--|--|--|--|
|`PROPERTIES.BOOTSTRAP.SERVER`| varchar | Conditional | The Kafka bootstrap server address. |
|`PROPERTIES.BOOTSTRAP.SERVER.PRIVATE_LINK`| varchar[] | Conditional | The Kafka bootstrap server address with AWS Private Link settings. This field is not allowed to set at the same time as field `PROPERTIES.BOOTSTRAP.SERVER`. |

For security related fields (SSL/SASL), the fields are identical to [this doc](https://materialize.com/docs/sql/create-connection/#kafka-ssl).

**Notice**: Risingwave will not check the connection to Kafka is valid or not.

> Some notes for implementation
>
> 1. Implement libkafka's DNS resolve callback
> 2. Store the dns mapping from the user's brokers address to private link endpoint ip
> 3. Interface for creating and modifying dns mappings

#### Kinesis

A Kinesis connection establishes a link to a Kinesis stream. Users can use Kinesis connections to create Kinesis sources and sinks. The following fields are required to create a Kinesis connection:

```sql
CREATE CONNECTION demo_connection TO KINESIS
  (aws.region='user_test_topic',
   endpoint='172.10.1.1:9090,172.10.1.2:9090',
   aws.credentials.role.arn='arn:aws-cn:iam::602389639824:role/demo_role',
   aws.credentials.role.external_id='demo_external_id',
   ...
  );
```

Accepted fields are listed below, which is a subset of [the doc](https://www.risingwave.dev/docs/current/create-source-kinesis/#parameters).

|Field|Required|Description|
|--|--|--|
|`aws.region`| yes | AWS service region. For example, US East (N. Virginia). |
|`endpoint`| Optional |URL of the entry point for the AWS Kinesis service.|
|`aws.credentials.access_key_id`| Conditional | This field indicates the access key ID of AWS. It must appear in pairs with aws.credentials.secret_access_key.|
|`aws.credentials.secret_access_key`|Conditional | This field indicates the secret access key of AWS. It must appear in pairs with aws.credentials.access_key_id.|
|`aws.credentials.session_token`|Optional| The session token associated with the temporary security credentials.|
|`aws.credentials.role.arn`|Optional| The Amazon Resource Name (ARN) of the role to assume.|
|`aws.credentials.role.external_id`|Optional|The [external id](https://aws.amazon.com/blogs/security/how-to-use-external-id-when-granting-access-to-your-aws-resources/) used to authorize access to third-party resources.|

**Notice**: Risingwave will not check the connection to Kinesis is valid or not.

#### Pulsar

```sql
CREATE CONNECTION demo_connection TO PULSAR
  (service.url = 'pulsar://localhost:6650/',
  admin.url='http://localhost:8080',);
```

The accepted fields are listed below, which is a subset of [the doc](https://www.risingwave.dev/docs/current/create-source-pulsar/#parameters).

|Field|Required|Description|
|--|--|--|
|`service.url`| Required| Address of the Pulsar service.	|
|`admin.url`|Required| Address of the Pulsar admin.|

**Notice**: Risingwave will not check the connection to Pulsar is valid or not.

## Future possibilities

* store some sensitive information, eg. passwords and SSL keys, in a encrypted way (use `CREATE SECRET`)
