# ELK
Elasticsearch-Logstash-kibana

Logstash ingests, enhances and parses log files and outputs them to elastic search. This data can then be queried and viewed using Kibana dashboard.

## Project Setup:

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
9. Next, create the index pattern and visualiser for the index created by the data injestion in above steps above on Kibana dashboard:

Kibana -> Discover tab (to analyze data) -> Create Index pattern: index pattern = "logstash-*  with @timestamp as the time filter field.

Kibana -> Visualize tab (to create visualization) -> Create visualization for a type say table, bar chart etc.

Kibana -> Dashboard tab (to see the data) -> Give dashboard a name and choose the visualization created above. You can use different visualizations on this dashboard.

## Theory:

### Logstash

To learn about logstash, go to below url, and just read about most commonly used sections- input, filter and output plugins.
https://www.elastic.co/guide/en/logstash/current/index.html

## Beats

Suppose we have to collate real time logs from different devices (web, mobile, desktop app). These devices can have beats like say filebeat, that collects and streams the logs onto the central logstash server that ingests all logs and send them to elastic search, which can then be vieweved on the Kibana UI. 

**Backpressure:** In case logstash is overloaded, it sends a message to filebeat to slow down, and then filebeat will wait for sometime before sending the logs onto logstash.


### Elastic Search

ES basically is a distributed version of Lucene. ES index is made up multiple shards. Each shard is made up of segments. Each segment is an inverted index. 

**Indexing:** Converting data into inverted index, is a time consuming process. Idea is to keep read fast. ES takes care of text anlysis and indexing. Text anlysis consists of removing stop words, lowercasing, stemming, synonyms. The analyzer used during querying should be the same as the one used for indexing.

Elastic search uses inverted index for indexing the words. ES also maintains a relevance score on the indexed items.

Elastic search is document based. Rel db to ES6 analogy: Table-> index, row-> document, column->field.

ES sends/receives data as JSON over HTTP(s).

ES automatically assigns data types to the data (JSON fields) inserted into it.

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

//Search using queries
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
**TODO: add a sample response**

GET query returns the data array against 'hits' key in the response json. The _score value in the response gives the relevance of the returned document. _maxScore is the highest score among the the documents returned.

**Search with pagination:**

GET by default returns top 10 records. You can modify this behaviour by giving size parameter. You can combine from, size and sort parameters to get paginated results. eg:

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
**Index Structure:**

The PUT shown above creates the index structure on the fly by itself. ES assigns datatypes to the fields as per the data being fed.
//TODO: available data types

We can also create the index explicitly with: 
1. custom settings(for specifying shards and replicas) and 
2. mappings (for data types and analyzer type).

Once the index has been created, above settings cannot be changed. If change is needed, index has to be deleted and re-created.

TODO: standard analyzer
Explore different analyzers: https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-analyzers.html

```
PUT /customers
{
	"settings": {
		"number_of_shards: 2,
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

**Schema scrictness:**

After defining the mapping, if you try to insert a document with an extra data column, then it will be accepted and the mapping is automatically updated to include this field as well. If you do not want this and want to enforce the schema, then you can do at time of index creation as : 

> "dynamic":false	  //simply ignores document not in agreed schema
> "dynamic":"strict"  //throw error if document not as per agreed schema

**DSL: Domain Specific Language** - JSON language that Elastic search understands. DSL has 2 components: **Query and Filter**
1. Query context

query:bool:must/must_not/should(good to have), query:range

should is goot to have, so records clearing the should condition get a better score.

2. Filter context

filter comes inside query only, it can not be given in a standalone way. 
query:bool:filter:match

Main difference bw query and filter is that filter does not do relevance scoring. All filtered records get a _score=0. Hence filters are very fast, query on the other hand give relevance score.

You can give must/must_not/should inside filter as well. The score will stay 0 as well.

If you give must/must_not/should in parallal to filter, then it is a mix of query and filter contexts, and now the results returned will have a relevance score.

You can change the relevance score based on some fields, by bumping these fields by a boost factor using ^. eg. "match": {"name":"abc^2"} -> boosting name=abc by a factor of 2.

**Aggregation**

ES is very fast when it comes to aggreagating as compared to relational databases. This makes ES popular in big data world.

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

## References

Java-ES POC: https://github.com/ramit21/elasticsearch-java

