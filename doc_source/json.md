# JSON SerDe Libraries<a name="json"></a>

In Athena, you can use two SerDe libraries for processing data in JSON:
+ The native [Hive JSON SerDe](#hivejson) 
+ The [OpenX JSON SerDe](#openxjson) 

## SerDe Names<a name="serde-names"></a>

 [Hive\-JsonSerDe](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+DDL#LanguageManualDDL-JSON) 

 [Openx\-JsonSerDe](https://github.com/rcongiu/Hive-JSON-Serde) 

## Library Names<a name="library-names"></a>

Use one of the following:

 [org\.apache\.hive\.hcatalog\.data\.JsonSerDe](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+DDL#LanguageManualDDL-JSON) 

 [org\.openx\.data\.jsonserde\.JsonSerDe](https://github.com/rcongiu/Hive-JSON-Serde) 

## Hive JSON SerDe<a name="hivejson"></a>

The Hive JSON SerDe is used to process JSON data, most commonly events\. These events are represented as blocks of JSON\-encoded text separated by a new line\.

You can also use the Hive JSON SerDe to parse more complex JSON\-encoded data with nested structures\. However, this requires having a matching DDL representing the complex data types\. See [Example: Deserializing Nested JSON](#nested-json-serde-example)\.

With this SerDe, duplicate keys are not allowed in `map` \(or `struct`\) key names\.

**Note**  
You can query data in regions other than the region where you run Athena\. Standard inter\-region data transfer rates for Amazon S3 apply in addition to standard Athena charges\. To reduce data transfer charges, replace *myregion* in `s3://athena-examples-myregion/path/to/data/` with the region identifier where you run Athena, for example, `s3://athena-examples-us-west-1/path/to/data/`\.

The following DDL statement uses the Hive JSON SerDe:

```
CREATE EXTERNAL TABLE impressions (
    requestbegintime string,
    adid string,
    impressionid string,
    referrer string,
    useragent string,
    usercookie string,
    ip string,
    number string,
    processid string,
    browsercookie string,
    requestendtime string,
    timers struct
                <
                 modellookup:string, 
                 requesttime:string
                >,
    threadid string, 
    hostname string,
    sessionid string
)   
PARTITIONED BY (dt string)
ROW FORMAT  serde 'org.apache.hive.hcatalog.data.JsonSerDe'
with serdeproperties ( 'paths'='requestbegintime, adid, impressionid, referrer, useragent, usercookie, ip' )
LOCATION 's3://myregion.elasticmapreduce/samples/hive-ads/tables/impressions';
```

## OpenX JSON SerDe<a name="openxjson"></a>

The OpenX SerDe is used by Athena for deserializing data, which means converting it from the JSON format in preparation for serializing it to the Parquet or ORC format\. This is one of two deserializers that you can choose, depending on which one offers the functionality that you need\. The other option is the Hive JsonSerDe\. 

This SerDe has a few useful properties that you can specify when creating tables in Athena, to help address inconsistencies in the data:

**ignore\.malformed\.json**  
Optional\. When set to `TRUE`, lets you skip malformed JSON syntax\. The default is `FALSE`\.

**dots\.in\.keys**  
Optional\. The default is `FALSE`\. When set to `TRUE`, allows the SerDe to replace the dots in key names with underscores\. For example, if the JSON dataset contains a key with the name `"a.b"`, you can use this property to define the column name to be `"a_b"` in Athena\. By default \(without this SerDe\), Athena does not allow dots in column names\.

**case\.insensitive**  
Optional\. The default is `TRUE`\. When set to `TRUE`, the SerDe converts all uppercase columns to lowercase\. Using `WITH SERDEPROPERTIES ("case.insensitive" = "FALSE")` allows you to use case\-sensitive key names in your data\. For every key that is not already all\-lowercase, you must also provide a mapping from the column name to the property name, e\.\g. `WITH SERDEPROPERTIES ("case.insensitive" = "FALSE", "mapping.userid" = "userId")`\. If you have two keys that are the same when lower cased, you can use this property to map them to different names, e\.g\. `WITH SERDEPROPERTIES ("case.insensitive" = "FALSE", "mapping.url1" = "URL", "mapping.url2" = "Url")`\.

**ColumnToJsonKeyMappings**  
Optional\. Maps column names to JSON keys that aren't identical to the column names\. This is useful when the JSON data contains keys that are [keywords](reserved-words.md)\. For example, if you have a JSON key named `timestamp`, set this parameter to `{"ts": "timestamp"}` to map this key to a column named `ts`\. This parameter takes values of type string\. It uses the following key pattern: `^\S+$` and the following value pattern: `^(?!\s*$).+`

With this SerDe, duplicate keys are not allowed in `map` \(or `struct`\) key names\.

The following DDL statement uses the OpenX JSON SerDe:

```
CREATE EXTERNAL TABLE impressions (
    requestbegintime string,
    adid string,
    impressionId string,
    referrer string,
    useragent string,
    usercookie string,
    ip string,
    number string,
    processid string,
    browsercokie string,
    requestendtime string,
    timers struct<
       modellookup:string, 
       requesttime:string>,
    threadid string, 
    hostname string,
    sessionid string
)   PARTITIONED BY (dt string)
ROW FORMAT  serde 'org.openx.data.jsonserde.JsonSerDe'
with serdeproperties ( 'paths'='requestbegintime, adid, impressionid, referrer, useragent, usercookie, ip' )
LOCATION 's3://myregion.elasticmapreduce/samples/hive-ads/tables/impressions';
```

## Example: Deserializing Nested JSON<a name="nested-json-serde-example"></a>

JSON data can be challenging to deserialize when creating a table in Athena\.

 When dealing with complex nested JSON, there are common issues you may encounter\. For more information about these issues and troubleshooting practices, see the AWS Knowledge Center Article [I receive errors when I try to read JSON data in Amazon Athena](https://aws.amazon.com/premiumsupport/knowledge-center/error-json-athena/)\. 

For more information about common scenarios and query tips, see [Create Tables in Amazon Athena from Nested JSON and Mappings Using JSONSerDe](http://aws.amazon.com/blogs/big-data/create-tables-in-amazon-athena-from-nested-json-and-mappings-using-jsonserde/)\.

The following example demonstrates a simple approach to creating an Athena table from data with nested structures in JSON\.To parse JSON\-encoded data in Athena, each JSON document must be on its own line, separated by a new line\. 

This example presumes a JSON\-encoded data with the following structure:

```
{
"DocId": "AWS",
"User": {
        "Id": 1234,
        "Username": "bob1234", 
        "Name": "Bob",
"ShippingAddress": {
"Address1": "123 Main St.",
"Address2": null,
"City": "Seattle",
"State": "WA"
   },
"Orders": [
   {
     "ItemId": 6789,
     "OrderDate": "11/11/2017" 
   },
   {
     "ItemId": 4352,
     "OrderDate": "12/12/2017"
   }
  ]
 }
}
```

The following `CREATE TABLE` command uses the [Openx\-JsonSerDe](https://github.com/rcongiu/Hive-JSON-Serde) with collection data types like `struct` and `array` to establish groups of objects\. Each JSON document is listed on its own line, separated by a new line\. To avoid errors, the data being queried does not include duplicate keys in `struct` and map key names\. Duplicate keys are not allowed in map \(or `struct`\) key names\. 

```
CREATE external TABLE complex_json (
   docid string,
   `user` struct<
               id:INT,
               username:string,
               name:string,
               shippingaddress:struct<
                                      address1:string,
                                      address2:string,
                                      city:string,
                                      state:string
                                      >,
               orders:array<
                            struct<
                                 itemid:INT,
                                  orderdate:string
                                  >
                              >
               >
   )
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
LOCATION 's3://mybucket/myjsondata/';
```
