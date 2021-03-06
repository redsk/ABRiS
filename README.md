    Copyright 2018 ABSA Group Limited
    
    Licensed under the Apache License, Version 2.0 (the "License");
    you may not use this file except in compliance with the License.
    You may obtain a copy of the License at
    
        http://www.apache.org/licenses/LICENSE-2.0
    
    Unless required by applicable law or agreed to in writing, software
    distributed under the License is distributed on an "AS IS" BASIS,
    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    See the License for the specific language governing permissions and
    limitations under the License.

# ABRiS - Avro Bridge for Spark

Pain free Spark/Avro integration.

Seamlessly convert your Avro records from anywhere (e.g. Kafka, Parquet, HDFS, etc) into Spark Rows. 

Convert your Dataframes into Avro records without even specifying a schema.

Seamlessly integrate with Confluent platform, including Schema Registry.


## Motivation

Among the motivations for this project, it is possible to highlight:

- Avro is Confluent's recommended payload to Kafka (https://www.confluent.io/blog/avro-kafka-data/).

- Current solutions do not support reading Avro record as Spark structured streams (e.g. https://github.com/databricks/spark-avro).

- Lack tools for writing Spark Dataframes directly into Kafka as Avro records.

- Right now, all the efforts to integrated Avro-Spark streams or Dataframes depend on the user.

- Confluent's Avro payload requires specialized deserializers, since they send schema metadata with it, which makes the integration with Spark cumbersome.

- In some use cases you may want to keep your Dataframe schema, adding your data as a nested structure, whereas in other cases you may want to use the schema for your data as the schema for the whole Dataframe.

## Usage

### Schema retention policies

Before using this library it is important to understand the concept behind ```za.co.absa.abris.avro.schemas.policy.SchemaRetentionPolicies```.

There are two policies: ```RETAIN_SELECTED_COLUMN_ONLY``` and ```RETAIN_ORIGINAL_SCHEMA```.

In the first option, the schema used to define your data becomes the schema of the resulting Dataframe. In the second, the original schema for the Dataframe is kept, and your data is added to a target column as a nested structure.

Taking Kafka as an example source, the resulting Dataframe contains the following fields: *key*, *value*, *topic*, *partition*, *offset*, *timestap* and *timestampType*. The field *value* contains the Avro payload.

Now assume you have an Avro schema like this:
```
{
       "type" : "record",
       "name" : "userInfo",
       "namespace" : "my.example",
       "fields" : [{"name" : "username",
                   "type" : "string",
                   "default" : "NONE"},

                   {"name" : "age",
                   "type" : "int",
                   "default" : -1}
                  ]
   }
```

If you use ```RETAIN_SELECTED_COLUMN_ONLY```, your final Dataframe will look like this:

| username   |age |
| ---------- |---:|
| femel      | 35 |
| user2      | 28 |
| ...        |... |

which means that the Dataframe now has the schema of your data. On the other hand, if you use ```RETAIN_ORIGINAL_SCHEMA```, the final result would be like this:

| key   |              value            | topic      | partition | offset | timestamp | timestampType |
| ------|:-----------------------------:| ----------:| ---------:| ------:| ---------:| -------------:|
| key 1 | {'name': 'femel', 'age': 35'} | test_topic |     1     |  21    | 2018-...  |  ...          |
| key 2 | {'name': 'user2', 'age': 28'} | test_topic |     2     |  22    |  ...      |  ...          |
| ...   | ...                           | test_topic |    ...    |  ...   |  ...      |  ...          |

which means that the original schema was retained, and the Avro payload was extracted as a nested structure inside the destination (configurable) column *value*.


### Reading Avro binary records from Kafka as Spark structured streams (loading schema from file system) and performing regular SQL queries on them

1. Import the library: ```import za.co.absa.abris.avro.AvroSerDe._```

2. Open a Kafka connection into a structured stream: ```spark.readStream.format("kafka"). ...```

3. Invoke the library on your structured stream: ```... .fromAvro("column_containing_avro_data", "path_to_Avro_schema or org.apache.avro.Schema instance")(SchemaRetentionPolicy)```

4. Pass on your query and start the stream: ```... .select("your query").start().awaitTermination()``` 

Below is an example whose full version can be found at ```za.co.absa.abris.examples.KafkaAvroReader```

```scala
    // import Spark Avro Dataframes
    import za.co.absa.abris.avro.AvroSerDe._

    val stream = spark
      .readStream
      .format("kafka")
      .option("kafka.bootstrap.servers", "localhost:9092")
      .option("subscribe", "test-topic")
      .fromAvro("column_containing_avro_data", "path_to_Avro_schema or org.apache.avro.Schema instance")(RETAIN_SELECTED_COLUMN_ONLY) // invoke the library in this case setting the data schema as the Dataframe schema

    stream.filter("field_x % 2 == 0")
      .writeStream.format("console").start().awaitTermination()
```

### Reading Avro binary records from Kafka as Spark structured streams (loading schema from Schema Registry) and performing regular SQL queries on them

1. Gather Schema Registry configurations:

```scala
val schemaRegistryConfs = Map(
  SchemaManager.PARAM_SCHEMA_REGISTRY_URL   -> "url_to_schema_registry",
  SchemaManager.PARAM_SCHEMA_REGISTRY_TOPIC -> "topic_name",
  SchemaManager.PARAM_VALUE_SCHEMA_ID       -> "current_schema_id" // set to "latest" if you want the latest schema version to used
)
```

2. Import the library: ```import za.co.absa.abris.avro.AvroSerDe._```

3. Open a Kafka connection into a structured stream: ```spark.readStream.format("kafka"). ...```

4. Invoke the library on your structured stream: ```... .fromAvro("column_containing_avro_data", "path_to_Avro_schema or org.apache.avro.Schema instance")(SchemaRetentionPolicy)```

5. Pass on your query and start the stream: ```... .select("your query").start().awaitTermination()```

The library will automatically retrieve the Avro schema from Schema Registry and configure Spark according to it.

Although the Avro schema was retrieved from Schema Registry, this API expects Avro records to be "standard", not Confluent ones. For consuming Confluent Avro records please refer to the next example.

Below is an example whose full version can be found at ```za.co.absa.abris.examples.KafkaAvroReader```

```scala
    // import Spark Avro Dataframes
    import za.co.absa.abris.avro.AvroSerDe._

    val stream = spark
      .readStream
      .format("kafka")
      .option("kafka.bootstrap.servers", "localhost:9092")
      .option("subscribe", "test-topic")
      .fromAvro("column_containing_avro_data", "path_to_Avro_schema or org.apache.avro.Schema instance")(RETAIN_ORIGINAL_SCHEMA) // invoke the library adding the extracted Avro data to the column originally containing it.

    stream.filter("field_x % 2 == 0")
      .writeStream.format("console").start().awaitTermination()
```

### Reading Avro binary records from Confluent platform (using Schema Registry) as Spark structured streams and performing regular SQL queries on them 

1. Gather Schema Registry configurations:

```scala
val schemaRegistryConfs = Map(
  SchemaManager.PARAM_SCHEMA_REGISTRY_URL   -> "url_to_schema_registry",
  SchemaManager.PARAM_SCHEMA_REGISTRY_TOPIC -> "topic_name",
  SchemaManager.PARAM_VALUE_SCHEMA_ID             -> "current_value_schema_id" // set to "latest" if you want the latest schema version to used
)
```

2. Import the library: ```import za.co.absa.abris.avro.AvroSerDe._```

3. Open a Kafka connection into a structured stream: ```spark.readStream.format("kafka"). ...```

4. Invoke the library on your structured stream: ```... .fromConfluentAvro("column_containing_avro_data", None, Some(schemaRegistryConfs))(SchemaRetentionPolicy)```

5. Pass on your query and start the stream: ```... .select("your query").start().awaitTermination()``` 

The library will automatically retrieve the Avro schema from Schema Registry and configure Spark according to it.

We strongly recommend you to read the documentation of ```za.co.absa.abris.avro.read.confluent.ScalaConfluentKafkaAvroDeserializer.deserialize()``` in order to better understand how the integration with the Confluent platform works.

Below is an example whose full version can be found at ```za.co.absa.abris.examples.ConfluentKafkaAvroReader```

```scala
    val schemaRegistryConfs = Map(
      SchemaManager.PARAM_SCHEMA_REGISTRY_URL   -> "url_to_schema_registry",
      SchemaManager.PARAM_SCHEMA_REGISTRY_TOPIC -> "topic_name",
      SchemaManager.PARAM_VALUE_SCHEMA_ID             -> "latest" // otherwise, just specify an id
    )
      
    // import Spark Avro Dataframes
    import za.co.absa.abris.avro.AvroSerDe._

    val stream = spark
      .readStream
      .format("kafka")
      .option("kafka.bootstrap.servers", "localhost:9092")
      .option("subscribe", "test-topic")
      .fromConfluentAvro("column_containing_avro_data", None, Some(schemaRegistryConfs))(RETAIN_SELECTED_COLUMN_ONLY) // invoke the library passing over parameters to access the Schema Registry

    stream.filter("field_x % 2 == 0")
      .writeStream.format("console").start().awaitTermination()
```

### Reading Avro binary records from Confluent platform (WITHOUT Schema Registry), as Spark structured streams and performing regular SQL queries on them 

1. Import the library: ```import za.co.absa.abris.avro.AvroSerDe._```

2. Open a Kafka connection into a structured stream: ```spark.readStream.format("kafka"). ...```

3. Invoke the library on your structured stream: ```... .fromConfluentAvro("column_containing_avro_data", Some(""path_to_avro_schema_in_some_file_system"), None)(SchemaRetentionPolicy)```

4. Pass on your query and start the stream: ```... .select("your query").start().awaitTermination()``` 

The Avro schema will be loaded from the file system and used to configure Spark Dataframes. The loaded schema will be considered both, the reader and the writer one.

We strongly recommend you to read the documentation of ```za.co.absa.abris.avro.read.confluent.ScalaConfluentKafkaAvroDeserializer.deserialize()``` in order to better understand how the integration with the Confluent platform works.

Below is an example whose full version can be found at ```za.co.absa.abris.examples.ConfluentKafkaAvroReader```


```scala
    // import Spark Avro Dataframes
    import za.co.absa.abris.avro.AvroSerDe._

    val stream = spark
      .readStream
      .format("kafka")
      .option("kafka.bootstrap.servers", "localhost:9092")
      .option("subscribe", "test-topic")
      .fromConfluentAvro("column_containing_avro_data", Some("path_to_avro_schema_in_some_file_system"), None)(RETAIN_SELECTED_COLUMN_ONLY) // invoke the library passing over the path to the Avro schema

    stream.filter("field_x % 2 == 0")
      .writeStream.format("console").start().awaitTermination()
```

### Writing Dataframes to Kafka as Avro records specifying a schema

1. Create your Dataframe. It MUST have a SQL schema.

2. Import the library: ```import za.co.absa.abris.avro.AvroSerDe._```

3. Invoke the library informing the path to the Avro schema to be used to generate the records: ```dataframe.toAvro("path_to_existing_Avro_schema")```

4. Send your data to Kafka as Avro records: ```... .write.format("kafka") ...```

Below is an example whose full version can be found at ```za.co.absa.abris.examples.KafkaAvroWriter```

```scala
      import spark.implicits._
      
      // import library
      import za.co.absa.abris.avro.AvroSerDe._
      
      val sparkSchema = StructType( .... // your SQL schema
      implicit val encoder = RowEncoder.apply(sparkSchema)
      val dataframe = spark.parallelize( .....
      
      dataframe
      	.toAvro("path_to_existing_Avro_schema or org.apache.avro.Schema instance") // invoke library
      	.write
      	.format("kafka")    
      	.option("kafka.bootstrap.servers", "localhost:9092")
      	.option("topic", "test-topic")
      	.save()  
```


### Writing Dataframes to Kafka as Avro records without specifying a schema

Follow the same steps as above, however, when invoking the library, instead of informing the path to the Avro schema, you can just inform the expected name and namespace for the records, and the library will infer the complete schema from the Dataframe: ```dataframe.toAvro("schema_name", "schema_namespace")```

Below is an example whose full version can be found at ```za.co.absa.abris.examples.KafkaAvroWriter```

```scala
      import spark.implicits._
            
      val sparkSchema = StructType( .... // your SQL schema
      implicit val encoder = RowEncoder.apply(sparkSchema)
      val dataframe = spark.parallelize( .....
      
      // import library
      import za.co.absa.abris.avro.AvroSerDe._
      
      dataframe
      	.toAvro("dest_schema_name", "dest_schema_namespace") // invoke library            
      	.write
      	.format("kafka")    
      	.option("kafka.bootstrap.servers", "localhost:9092")
      	.option("topic", "test-topic")
      	.save()         
```

### Writing Dataframes to Confluent Kafka as Avro records without specifying a schema.

It is possible to infer the schema from the Dataframe, register it into Schema Registry and then attach its id to the beginning of the Avro binary payload, so that the records can be seamlessly consumed by Confluent tools.

Follow the same steps as above, however, when invoking the library, instead of informing the path to the Avro schema, you can just inform the expected name and namespace for the records, and the library will infer the complete schema from the Dataframe: ```dataframe.toAvro("schema_name", "schema_namespace")```

In this option it is mandatory to provide access to Schema Registry.

Below is an example whose full version can be found at ```za.co.absa.abris.examples.ConfluentKafkaAvroWriter```

```scala
      import spark.implicits._

      val sparkSchema = StructType( .... // your SQL schema
      implicit val encoder = RowEncoder.apply(sparkSchema)
      val dataframe = spark.parallelize( .....

      val schemaRegistryConfs = Map(
        SchemaManager.PARAM_SCHEMA_REGISTRY_URL   -> "url_to_schema_registry"
      )

      val topic = "your_destination_topic"

      // import library
      import za.co.absa.abris.avro.AvroSerDe._

      dataframe
      	.toConfluentAvro(topic, "dest_schema_name", "dest_schema_namespace")(schemaRegistryConfs) // invoke library
      	.write
      	.format("kafka")
      	.option("kafka.bootstrap.servers", "localhost:9092"))
      	.option("topic", topic)
      	.save()
```

### Writing/reading keys and values as Avro to/from Kafka

ABRiS also supports the simultaneous conversion of keys and values in Kafka Dataframes to/from Avro.

The API usage is exactly the same, however, the entry point is different: ```za.co.absa.abris.avro.AvroSerDeWithKeyColumn```.

Also, when configuring Schema Registry parameters, one extra entry will be required, the id of the expected key schema:

```scala
val schemaRegistryConfs = Map(
  SchemaManager.PARAM_SCHEMA_REGISTRY_URL   -> "url_to_schema_registry",
  SchemaManager.PARAM_SCHEMA_REGISTRY_TOPIC -> "topic_name",
  SchemaManager.PARAM_VALUE_SCHEMA_ID       -> "current_value_schema_id", // set to "latest" if you want the latest schema version to used for the 'value' column
  SchemaManager.PARAM_KEY_SCHEMA_ID         -> "current_key_schema_id" // set to "latest" if you want the same for the 'key' column
)
```

You can find examples for both, reading and writing from and to Apache and Confluent Kafka in package ```za.co.absa.abris.examples.using-keys```.

### Writing/reading values as Avro and plain keys as String to/from Kafka

ABRiS also supports writing to Kafka using plain keys while still converting the content of the *value* column (the payload) into Avro.

The API usage is exactly the same as in the previous item (*Writing/reading keys and values as Avro to/from Kafka*), however, instead of invoking ```toAvro()``` on the DataFrame, the plain key feature requires users to invoke ```toAvroWithPlainKey()```. For the rest, the behavior is exactly the same.

The keys will be converted to their string representation, thus, for instance, if it is an integer value, let's say 8, it will be retrieved as the string "8".

There are two complete examples of this feature. One at ```za.co.absa.abris.examples.using_keys.KafkaAvroWriterWithPlainKey```, for regular Avro payload, and another one at ```za.co.abris.examples.using_keys.ConfluentKafkaAvroWriterWithPlainKey``` for Confluent Kafka.

## Other Features

### Schema registration for subject into Schema Registry
This library provides utility methods for registering schemas with topics into Schema Registry. Below is an example of how it can be done.

```scala
    val schemaRegistryConfs = Map(
      SchemaManager.PARAM_SCHEMA_REGISTRY_URL   -> "url_to_schema_registry"
    )
    SchemaManager.configureSchemaRegistry(schemaRegistryConfs)

    val topic = "example_topic"
    val subject = SchemaManager.getSubjectName(topic, false) // create a subject for the value

    val schema = AvroSchemaUtils.load("path_to_the_schema_in_a_file_system")

    val schemaId = SchemaManager.register(schema, subject)
```

### Data Conversions
This library also provides convenient methods to convert between Avro and Spark schemas. 

If you have an Avro schema which you want to convert into a Spark SQL one - to generate your Dataframes, for instance - you can do as follows: 

```scala
val avroSchema: Schema = AvroSchemaUtils.load("path_to_avro_schema")
val sqlSchema: StructType = SparkAvroConversions.toSqlType(avroSchema) 
```  

You can also do the inverse operation by running:

```scala
val sqlSchema = new StructType(new StructField ....
val avroSchema = SparkAvroConversions.toAvroSchema(sqlSchema, avro_schema_name, avro_schema_namespace)
```

### Alternative Data Sources
Your data may not come from Spark and, if you are using Avro, there is a chance you are also using Java. This library provides utilities to easily convert Java beans into Avro records and send them to Kafka. 

To use you need to: 

1. Store the Avro schema for the class structure you want to send (the Spark schema will be inferred from there later, in your reading job):

```Java
Class<?> dataType = YourBean.class;
String schemaDestination = "/src/test/resources/DestinationSchema.avsc";
AvroSchemaGenerator.storeSchemaForClass(dataType, Paths.get(schemaDestination));
```

2. Write the configuration to access your cluster:

```Java
Properties config = new Properties();
config.put("bootstrap.servers", "...
...
```

3. Create an instance of the writer:

```Java
KafkaAvroWriter<YourBean> writer = new KafkaAvroWriter<YourBean>(config);
```

4. Send your data to Kafka:

```Java
List<YourBean> data = ...
long dispatchWait = 1l; // how much time the writer should expect until all data are dispatched
writer.write(data, "destination_topic", dispatchWait);
```

A complete application can be found at ```za.co.absa.abris.utils.examples.SimpleAvroDataGenerator``` under Java source.

## Maven dependency
Dependency:
```
<dependency>
    <groupId>za.co.absa</groupId>
	<artifactId>abris_2.11</artifactId>
	<version>2.1.0</version>
</dependency>
```

## Dependencies

The environment dependencies are below. For the other dependencies, the library POM is configured with all dependencies scoped as ```compile```, thus, you can understand it as a self-contained piece of software. In case your environment already provides some of those dependencies, you can specify it in your project POM.

- Scala 2.11

- Spark 2.2.0

- Spark SQL Kafka 0-10

- Spark Streaming Kafka 0-8 or higher


## Performance

### Setup

- Windows 7 Enterprise 64-bit 

- Intel i5-3550 3.3GHz

- 16 GB RAM


Tests serializing 50k records using a fairly complex schemas show that Avro records can be up to 16% smaller than Kryo ones, i.e. converting Dataframes into Avro records save up to 16% more space than converting them into case classes and using Kryo as a serializer.

In local tests with a single core, the library was able to parse, per second, up to 100k Avro records into Spark rows per second, and up to 5k Spark rows into Avro records.

The tests can be found at ```za.co.absa.abris.performance.SpaceTimeComplexitySpec```.

 
