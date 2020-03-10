# ELK
Elasticsearch-Logstash-kibana

Logstash ingests, enhances and parses log files and outputs them to elastic search. This data can then be queried and viewed using Kibana dashboard.

## Project Setup:

1. Download/install Elastic search, Kibana and Logstash from elastic.co
2. Start ElasticSearch from bin directory, and open http://localhost:9200 on browser.
3. In Kibana downloaded folder, go to config/kibana.yml -> uncomment elastic search url and it should point to above url. Open http://localhost:5601 to see Kibana dashboard.
4. Try terminating elastic search. You will see error on kibana logs. Kibana can be run only if underlying elasticsearch instance is running.
5. Download the logs file to a location on your system. These are basically apache logs that we want logstash to ingest and feed into elastic search.
6. Create apache.conf (named so as we are processing apache logs) on your system and update the path to logs created in above step.
7. Execute logstash for the conf file apache.conf 

```
logstash -f C:\Software\data\apache.conf
```
8. Verify on elastic search (or devtools of kibana : GET /logstash*) if all records have been uploaded to ES:

```
http://localhost:9200/logstash*/_count 
```

9.Next, create the index pattern and visualiser for the dashboard on Kibana:

Kibana -> Discover tab (to analyze data) -> Create Index pattern: index pattern = "logstash-*  with @timestamp as the time filter field.

Kibana -> Visualize tab (to create visualization) -> Create visualization fo a type say table, bar chart etc.

Kibana -> Dashboard tab (to see the data) -> Give dashboard a name and choose the visualization created above. You can use multiple visualizations on this dashboard.

----------------------------

## Theory:

**Logstash**
To learn about logstash, go to below url, and just read about most commonly used sections- input, filter and output plugins.
https://www.elastic.co/guide/en/logstash/current/index.html

**Elastic Search**
Elastic search uses inverted index for indexing the words. ES also maintains a relevance score on the indexed items.
Elastic search is document based. Rel db to ES6 analogy: Table-> index, row-> document, column->field.
ES sends/receives data as JSON over HTTP(s).

ES automatically assigns data types to the data (JSON fields) inserted into it.

Some basic ES queries that you can execute on Kibana -> devtools:

```
PUT /vehicles/car/123
{
  "make":"Honda",
  "price": 100.1
}

GET /vehicles/car/123

DELETE /vehicles/car/123    //Delete puts a marker, and purges them later (much like Cassandra tombstones)


GET /vehicles/car/_search
{
  "query": {
    "match_all": {}
  }
}

_search:::->

GET /vehicles/car/_search
{
  "query": {
    "match_all": {}
  }
}

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

```
GET query returns the data array against 'hits' key in the response json. The _score value in the response gives the relevance of the returned document. _maxScore is the highest score among the the documents returned.

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

ES basically is a distributed version of Lucene. ES index is made up multiple shards. Each shard is made up of segments. Each segment is an inverted index. 

Indexing: converting data into inverted index, is a time consuming process. Idea is to keep read fast. ES takes care of text anlysis and indexing. Text anlysis consists of removing stop words, lowercasing, stemming, synonyms. the analyzer used during querying should be the same as the one used for indexing.

The PUT shown above creates the index structure on the fly by itself. But ideally we must specify the structure ourselves : 1. settings(for specifying shards and replicas) and 2. mappings(for data types and analyzer type).

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

Bulk insert of documents:

```
POST /vehicles/cars/_bulk
{....}
{....}
```

After defining the mapping if you try to insert a document with an extra data column, then it will be accepted and the mapping is automatically updated to include this field as well. If you do not want this and want to enforce the schema, then you can do at time of index creation as : 

> "dynamic":false		//simply ignores document not in agreed schema
> "dynamic":"strict"  //throws error

Explore different analyzers: https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-analyzers.html

------------------

**DSL: Domain Specific Language** - JSON language that Elastic search understands. DSL has 2 components: Query and Filter
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

-----------------------

**Aggregation**

ES is very fast when it comes to aggreagating as compared to rel. databases. This makes ES a big player in big data world.
Performed using "aggs" keyword. You can perform aggregation like count, min, max, average, sum etc using respective keys or using stats keyword that covers all of the aggregations.

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

**Beats**

Suppose we have to collate real time logs from different devices (web, mobile, desktop app). These devices can have beats like say filebeat, that collects and streams the logs onto the central logstash server that ingests all logs and send them to elastic search, which can then be vieweved on the Kibana UI. 

In case logstash is overloaded, it sends a message to filebeat to slow down, and then filebeat will wait for sometime before sending the logs onto logstash.




