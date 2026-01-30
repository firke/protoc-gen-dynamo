# protoc-gen-dynamo: Storing protobuf objects in Amazon DynamoDB

`protoc-gen-dynamo` is used to generate Go ('golang') code for storing objects inside Amazon DynamoDB.  It works by 
generating code for `MarshalDynamo*` and `UnmarshalDynamo*` functions based on a Protocol Buffer message.  These 
functions are interfaces in the AWS SDK for Go for serializing between Go and Amazon DynamoDB.

## Contributions Welcome!

Please [open issues in Github](https://github.com/firke/protoc-gen-dynamo/issues) for pull requests, ideas, bugs, and 
general thoughts. Today the project today is focused on Go, but other languages are welcome!

## Features

- *Performance*: TODO: benchmarks
- Compatible with [AWS SDK for Go](https://aws.amazon.com/sdk-for-go/).  Code is generated to implement `MarshalDynamoDBAttributeValue(*dynamodb.AttributeValue) error` and `UnmarshalDynamoDBAttributeValue(*dynamodb.AttributeValue) error`.
- Compatible with [guregu/dynamo](https://github.com/guregu/dynamo), an expressive DynamoDB library for Go. Code is generated to implement `MarshalDynamo() (*dynamodb.AttributeValue, error)` and `UnmarshalDynamo(*dynamodb.AttributeValue) error`.


## Installing `protoc-gen-dynamo`

```
go install github.com/firke/protoc-gen-dynamo@latest
```

## Using `protoc-gen-dynamo`

- Include `dynamo/v1/dynamo.proto` in your protobuf compiler include path.

### Annotations

See the [dynamo.proto](./proto/dynamo/v1/dynamo.proto) for all possible annotations.

#### `dynamo.v1.msg` Annotations on Protobuf Messages

- `dynamo.v1.msg.disabled = <bool>`: Disables generation of DynamoDB Marshalling for this message.
- `dynamo.v1.msg.key`: Defines partition and sort key fields. Multiple keys create GSIs.
  - `pk_fields`: List of field names for the partition key
  - `sk_fields`: List of field names for the sort key
  - `sk_const`: Constant string value for sort key (alternative to sk_fields)
  - `shard`: Sharding options (`enabled`, `shard_count`)

#### `dynamo.v1.field` Annotations on Protobuf Fields

- `dynamo.v1.field.expose = <bool>`: Exposes this field as a top-level DynamoDB attribute.
- `dynamo.v1.field.name = <string>`: Sets the name of the field as stored in DynamoDB.
- `dynamo.v1.field.type.set = <bool>`: Set this field to be a String Set, Number Set, or Binary Set instead of a List. Only valid on `repeated` protobuf fields.
- `dynamo.v1.field.type.unix_second = <bool>`: Set this field to be a Number with the number of seconds since the unix epoch. Only valid for `google.protobuf.Timestamp` protobuf type.
- `dynamo.v1.field.type.unix_milli = <bool>`: Set this field to be a Number with the number of milliseconds since the unix epoch. Only valid for `google.protobuf.Timestamp` protobuf type.
- `dynamo.v1.field.type.unix_nano = <bool>`: Set this field to be a Number with the number of nanoseconds since the unix epoch. Only valid for `google.protobuf.Timestamp` protobuf type.

### Mapping Protocol Buffer types to DynamoDB Types

Amazon DynamoDB supports many types natively, 

[Amazon DynamoDB](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.NamingRulesDataTypes.html#HowItWorks.DataTypes)

| `.proto` Type  | DynamoDB Type |
| ------------ |:-------------|
| double       | N – Number  |
| float        | N – Number  |
| int32        | N – Number  |
| int64        | N – Number  |
| uint32       | N – Number  |
| uint64       | N – Number  |
| sint32       | N – Number  |
| sint64       | N – Number  |
| fixed32      | N – Number  |
| fixed64      | N – Number  |
| sfixed32     | N – Number  |
| sfixed64     | N – Number  |
| bool         | BOOL – Boolean  |
| string       | S – String  |
| bytes        | B – Binary  |
| map<K,V>     | M – Map <sup>[1](#maps-1)</sup>    |
| repeated V   | L – List <sup>[2](#lists-2)</sup>  |
| message      | M – Map <sup>[3](#message-3)</sup> |
| Timestamp    | S - String <sup>[4](#timestamp-4)</sup>  |
| Duration     | S - String  <sup>[5](#duration-5)</sup>  |

<a name="maps-1">1</a>: Maps: Keys must be a string or Numeric type.  Value can be any type.

<a name="lists-2">2</a>: Lists: By default lists are mapped directly.  The `dynamo.type.set = true` annotation can
be used to convert the List into a String Set, Binary Set, or Number Set.

<a name="message-3">3</a>: Nested Messages: By default nested messages are recursively turned into a Map. The 
`dynamo.type.binary = true` annotation can be used to serialize the nested message into the protobuf binary format.

<a name="timestamp-4">4</a>: Timestamps: By default timestamps are converted to an 
[RFC 3339 string](https://tools.ietf.org/html/rfc3339) and stored as a String. The `dynamo.type.binary = true` 
annotation can be used to serialize the timestamp as a protobuf binary format.  The `dynamo.type.unix_seconds = true` 
annotation can be used to serialize the timestamp as a Number, any partial seconds are truncated.  The 
`dynamo.type.unix_ms = true` annotation can be used to serialize the timestamp as a Number, any partial seconds are 
converted to a floating point milliseconds.  

<a name="timestamp-4">4</a>: Duration: By default durations are converted to a String. The `dynamo.type.binary = true` 
annotation can be used to serialize the duration as a protobuf binary format.

## Example

```protobuf
edition = "2023";

package examplepb.v1;

import "dynamo/v1/dynamo.proto";
import "google/protobuf/timestamp.proto";

message Store {
  option (dynamo.v1.msg).key = {
    pk_fields: ["id"]
    sk_fields: ["country", "region", "state", "city"]
  };

  string id = 1 [(dynamo.v1.field).expose = true, (dynamo.v1.field).name = "store_id"];
  string country = 2;
  string region = 3;
  string state = 4;
  string city = 5;
  bool closed = 6;
  google.protobuf.Timestamp opening_date = 7 [(dynamo.v1.field).type.unix_second = true];
  repeated string best_employee_ids = 8 [(dynamo.v1.field).type.set = true];
}
```

When serialized, this will generate the following DynamoDB Attributes:

```json
{
  "pk": {"S": "examplepb_store:1234"},
  "sk": {"S": "united_states:west:california:concord"},
  "store_id": {"S": "1234"},
  "value": {"B": "<zstd-compressed protobuf binary>"},
  "typ": {"S": "examplepb.Store"}
}
```

## License

`protoc-gen-dynamo` is licensed under the [Apache License, Version 2.0](./LICENSE)
