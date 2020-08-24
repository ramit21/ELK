# ELK
Elasticsearch-Logstash-Kibana

Logstash ingests, enhances and parses log files and outputs them to elastic search. This data can then be queried and viewed using Kibana dashboard.

## Project Setup

1. Download/install Elastic search, Kibana and Logstash from elastic.co
2. Start ElasticSearch from bin directory, and open http://localhost:9200 on browser.
3. In Kibana downloaded folder, go to config/kibana.yml -> uncomment elastic search url and it should point to above url. Start Kibana. Open http://localhost:5601 to see Kibana dashboard.
4. Try terminating elastic search. You will see error on kibana logs. Kibana can be run only if underlying elasticsearch instance is running.
5. Download the logs file to a location on your system (shared under data folder in this project). These are basically apache logs that we want logstash to ingest and feed into elastic search.
6. Create apache.conf (named so as we are processing apache logs, but can be given any name as such) on your system and update the path to logs created in above step. (see data/apache.conf)
7. Start logstash with the conf file apache.conf 
```
logstash -f C:\Software\data\apache.conf
```
8. Verify on elastic search (or devtools of kibana : GET /logstash*) if all records have been uploaded to ES:
```
http://localhost:9200/logstash*/_count 
```
9. Next, explore Kibana dashboard. Create the index pattern and visualiser for the index created by the data injestion in above steps above on Kibana dashboard:

Kibana -> Discover tab (to analyze data) -> Create Index pattern: index pattern = "logstash-*  with @timestamp as the time filter field.

Kibana -> Visualize tab (to create visualization) -> Create visualization for a type say table, bar chart etc.

Kibana -> Dashboard tab (to see the data) -> Give dashboard a name and choose the visualization created above. You can use different visualizations on this dashboard.

**Docker Setup:** You can also chooose to setup ES, Kibana and Logstash using Docker images. Run below commands and open respective urls to verify that installation has been completed:

```
//ES
docker run -d --name es762 -p 9200:9200 -e "discovery.type=single-node" elasticsearch:7.6.2

http://localhost:9200/

//Kibana (link it with ES container)
docker run --name kibana762 --link es762:elasticsearch -p 5601:5601 kibana:6.8.10

http://localhost:5601/

//Logstash (link it with ES container)
Settings files can also be provided through bind-mounts. Logstash expects to find them at /usr/share/logstash/config/. by default. It’s possible to provide custom directory containing all needed files:

docker run --name logstash762 --link es762:elasticsearch --rm -it -v ~/settings/:/Users/ramit21/git/ELK/data/logstash_mac/ docker.elastic.co/logstash/logstash:7.9.0

```

## Theory

### Logstash

Used to injest, process and push data out to the destination. Source of injestion can be files, S3, beats, Kafka etc. Destination can be elastic-search, AWS, hadoop, mongo db etc. In the processing step, it can parse, transform, filter, derive structure from unstructured data etc.

Can scale across many nodes.

To learn more about logstash, go to below url, and just read about most commonly used sections- input, filter and output plugins.
https://www.elastic.co/guide/en/logstash/current/index.html

## Beats

Suppose we have to collate real time logs from different devices (web, mobile, desktop app). These devices can have beats like say filebeat, that collects and streams the logs onto the central logstash server that ingests all logs and send them to elastic search, which can then be vieweved on the Kibana UI. 

**Backpressure:** In case logstash is overloaded, it sends a message to filebeat to slow down, and then filebeat will wait for sometime before sending the logs onto logstash.

Logstash-ES-Kibana is called 'ELK'.

Beats-Logstash-ES-Kibana is called 'elastic stack'.

### Elastic Search

ES basically is a distributed scalable version of Lucene. ES index is made up multiple shards. Each shard is made up of segments. Each segment is an inverted index. 

Data is partitioned to give scalability when reading. Data can also be replicated among shards. Writes are send to primary shard and then replicated. Read request can be routed to any of the read replcias.

Once index has been created, no. of replicas can be increased on runtime. No. of shards however cannot be changed, and if really required, you need to re-index your data (redistribute among the shards).

**Indexing:** Converting data into inverted index, is a time consuming process. Idea is to keep read fast. ES takes care of text anlysis and indexing. Text anlysis consists of removing stop words, lowercasing, stemming, synonyms. The analyzer used during querying should be the same as the one used for indexing.

Elastic search uses inverted index for indexing the words. ES also maintains a relevance score on the indexed items.

Elastic search is document based. Rel db to ES6 analogy: Table -> Index, row -> Document, column -> Field.

ES sends/receives data as JSON over HTTP(s).

3 ways to interact with ES:
1. ES Rest endpoints via curl/postman etc.
2. From programming languages' client APIs
3. Analytics tools like Kibana.

**Mappings:** ES can automatically assigns data types to the data (JSON fields) inserted into it. But you can also define "mappings" to strictly define the datatype of all or specific fields:

```
curl -H "Content-Type:application/json" -XPUT 127.0.0.1:9200/movies -d '
{
	"mappings": {
		"<index-name>" : {
			"properties": {
				"<field-name eg. year>" : {"type": "date"}
			}
		}
	}
}
```
You can also use mappings to make a field queryable = true/false, or to define your **tokenizer** (that splits strings based on white space, punctuation etc) and **token-filter** (that does lowercasing, stemming, stopwords etc).

**Kibana devtools**

Sample ES queries to execute on Kibana -> devtools:

```
//Insert a document
PUT /vehicles/car/123
{
  "make": "Honda",
  "price": 100.1
}

//Fetch the document
GET /vehicles/car/123

//Delete puts a marker, and purges them later (much like Cassandra tombstones)
DELETE /vehicles/car/123    

//Search all documents in an index
GET /vehicles/car/_search
{
  "query": {
    "match_all": {}
  }
}

//Search using JSON body
GET /vehicles/car/_search
{
  "query": {
     "term": {
       "make": {
         "value": "Honda"
       }
     }
  }
}

//Bulk insert of documents:
POST /vehicles/cars/_bulk
{....}
{....}

```
**Sample response**

GET query returns the data array against 'hits' key in the response json. The _score value in the response gives the relevance of the returned document. _maxScore is the highest score among the the documents returned.

```
[Results]
"hits": {
  "total": 2,
  "max_score": 1.6323128,
  "hits": [
    {
      "_index": "bookdb_index",
      "_type": "book",
      "_id": "3",
      "_score": 1.6323128,
      "_source": {
        "summary": "build scalable search applications using Elasticsearch without having to do complex low-level programming or understand advanced data science algorithms",
        "title": "Elasticsearch in Action",
        "publish_date": "2015-12-03"
      },
      "highlight": {
        "title": [
          "Elasticsearch <em>in</em> <em>Action</em>"
        ]
      }
    },
	{...}
```

**Pagination:** GET by default returns top 10 records. You can modify this behaviour by giving size parameter. You can combine from, size and sort parameters to get paginated results. eg:

```
GET /vehicles/car/_search
{
	"from":0,
	"size":5,
	
  "query": {
	"match_all":{}
  },
  
  "sort":[{"price":{"order":"desc"}}]
}

```
---------------

**Index Structure**

The PUT shown above creates the index structure on the fly by itself. ES assigns datatypes to the fields as per the data being fed.

Common ES 6 data types: text, keyword, date, long, integer, double, float, boolean, binary, ranges (integer/float/long/double ranges). 

We can also create the index explicitly with: 
1. custom settings(for specifying shards and replicas) and 
2. mappings (for data types and analyzer type).

Once the index has been created, above settings cannot be changed. If change is needed, index has to be deleted and re-created.

**Analyzers**

Analyzers are used to define strategy for partial matching of search results. Search results depend on the analyzer used, eg. case-insensitive, stemmed, stopwords removed, synonyms applied etc.

Eg. of index creation, specifying no. of shards, replicas, data-type/analyzer for a speciifc field named "gender". Also note that replicas are created for each primary shard. Hence in the below eg, we will get 3(primary)+ 3X1 (replicas) = 6 shards.
```
PUT /customers
{
	"settings": {
		"number_of_shards: 3,
		"number_of_replicas: 1
		},
	"mapping": {
		"MAPPING_VALUE" : {
			"properties" : {
				"gender": {"type":"text", "analyzer": "standard"}
				}
			}
		},
}
```

Fields of types **Text** allow analyzing. Mappings can be used to suppress analyzing by declaring them of type **keyword**. This is done if you want exact match for the field (as against to partial match of analyzed fields).

Important to note that analyzed fields (text as above) cannot be used for sorting as data exists in the inverted index as individual terms, not the enitre string. If you really want to sort on an analyzed field, map an unanalyzed copy of the field, which is of type keyword:

```
curl.....
{
	"mappings":{
		"movie": {
			"properties": {
				"title": {
					"type":"text",
					"fields": {
						"raw":{   //give any name for the copy
							"type":"keyword"
						}
					}
				}
			}
		}
	}
}

Do reindex after applying above mapping. Now sort using the unalayzed field:

curl -XGET '127.0.0.1:9200/movies/movie/_search?sort=title.raw&pretty'

```

-------------------------

**Schema scrictness:**

After defining the mapping, if you try to insert a document with an extra data column, then it will be accepted and the mapping is automatically updated to include this field as well. If you do not want this and want to enforce the schema, then you can do at time of index creation as : 

> "dynamic":false	  //simply ignores document not in agreed schema
> "dynamic":"strict"  //throw error if document not as per agreed schema

**DSL: Domain Specific Language** is the JSON language that Elastic search understands. DSL has 2 components: **Query and Filter**

1. Query context: return data in terms of relevance

Syntax: 'query:bool:must/must_not/should', 'query:range'

'should' means good to have, so records clearing the should condition get a better score.

2. Filter context: yes/no type of search

Syntax: 'query:bool:filter:match'

filter comes inside query only, it can not be given in a standalone way. 

Main difference bw query and filter is that filter does not do relevance scoring. All filtered records get a _score=0. Hence filters are very fast, query on the other hand give relevance score. Use filters when you can as they are faster and cacheable.

You can give must/must_not/should inside filter as well. The score will stay 0 as well.

If you give must/must_not/should in parallal to filter, then it is a mix of query and filter contexts, and now the results returned will have a relevance score. eg below:

```
curl ......
{
	"query": {
		"bool":{				//bool is used to combine filters/queries
			"must":{"term" : {"title":"trek"}},        //query
			"filter":{range": {"year":{"gte":2010}}}   //filter
		}
	}
}
```

You can change the relevance score based on some fields, by bumping these fields by a boost factor using ^. eg. "match": {"name":"abc^2"} -> boosting name=abc by a factor of 2.

**Aggregation**

ES is very fast when it comes to aggreagating as compared to relational databases and even hadoop or spark. This makes ES popular in big data world.

Performed using **'aggs'** keyword. You can perform aggregation like count, min, max, average, sum etc. using respective keys or using stats keyword that covers all of the aggregations.

```
GET /vehicles/car/_search
{
	"aggs" : {
		"popular_cars" : {
			"terms": {
				"field": <my field to display>
				},
				"aggs": {
					"stats_on_price":{
						"stats":{
							"field":"<field to aggregate upon, like price>"
							}
						}
```
You can also use aggregation to create histogram data by providing the interval on a field value.

ES provides very good in built support for aggregating on fields that contain time and dates.

**Optimistic Concurrency:**

Every document has a _version field. ES documents are immutable. When you update an existing document, a new document is created with an incremented _version and the old document is marked for deletion.

When 2 parallel threads try to update the same document version, ES allows only 1, while the other gets an error. You can configure retry_on_conflicts=N, where the failed thread can try max N number of times to write into ES using the updated version number.

## Advanced ES Concepts

**X-Pack** if enabled, helps with security, cluster monitoring, ML etc. Can be accessed via 'Monitoring' tab on Kibana.

**Full-text vs phrasal queries:**

Phrasal queries can be given using "match_phrase". Returns the results in correct relevance order. You can further add "slop" to your queries that represents how many extra words are allowed between the words given in the search phrase. Of course, lesser the distance, more the relevance score.

```
//allows 2 words bw star and beyond:

curl ......
{
 "query" :{
	 "match_phrase": {
		 "title": {"query" :"star beyond", "slop":2} 
	 }
 }
}
```
**Fuzziness:** A way to account for typos and misspellings in search query. Results are driven by **levenshtein edit distance** which is basically the no. of substitutions/insertions/deleteions required to match search results. When making fuzzy queries, we specify how much fuzziness is permitted. You can also give fuzziness:AUTO that allows different fuzziness tolerance based on the length of the text.

**Search as you type:** Create a custom analyzer to create N-grams of the text field. In this custom analyzer, combine 'autocomplete analyzer' along with filter of type 'autocomplete_filter', latter searches as per specified min/max N-gram values. Next, map your field with this custom analyzer using a 'mapping' and re-index. Lastly, when making the search query, use standard analyzer, and not the custom analyzer that you created, as we dont want to split our search query text into N-grams.

**Scaling using multiple indices:** No need to store all data in one index. You can split it into different indices, and make search queries to both the indices. You can use aliases to point to index, and use alias in your queries. Eg. For time series data, you can have 2 indices with aliases "logs_current", and "logs_3_months", and point these to specific indices as they rotate (ie current data becomes old data in 3 months, so you can delete index having old data, and make logs_3_month point to current, and then create a new empty index and make logs_current point to this new index)

**H/W considerations:** 
1. 64GM RAM is enough (32 for ES, and 32 for OS/disk cache for lucene). If you give more than 32 GB to ES, in memory pointers can blow up.
2. CPU not that important as ES is not heavy on CPU.
3. Use RAID0, cluster is already redundant, hence no need of RAID1 etc.
4. SSD fast disks and fast network are prefered.

**Snapshots:** Used to backup indices. Smart enough to only store changes since the last snapshot. When restoring snapshots, you must first close the index.

**Cloud:** AWS and Elastic Cloud provide ES offerings, where you can create your ES clusters. You get a cluster url and a Kibana url to connect to.

## References

Java-ES POC: https://github.com/ramit21/elasticsearch-java

ELK stack using custom yml configs with Docker: https://medium.com/analytics-vidhya/elk-stack-in-docker-6285ec1ac1aa

Analyzers: https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-analyzers.html
