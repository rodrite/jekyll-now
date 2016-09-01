---
layout: post
title: "Solr Streaming Expression Example: TopicStream function"
---

![solr](../images/Solr.png){:class="img-responsive"}

## Getting started ##
In order to understand this example, you need to be already familiar with the basics of using Solr and you have one Solr node in cloud mode.
More information about getting started with solr cloud here: [Getting Started with SolrCloud](https://cwiki.apache.org/confluence/display/solr/Getting+Started+with+SolrCloud)


## TopicStream function ##
In [Solr Streaming Expression](https://cwiki.apache.org/confluence/display/solr/Streaming+Expressions#StreamingExpressions-topic) topic function is defined as a function that *provides publish/subscribe messaging capabilities built on top of SolrCloud. **The topic function allows users to subscribe to a query**. The function then provides one-time delivery of new or updated documents that match the topic query. The initial call to the topic function establishes the checkpoints for the specific topic ID. Subsequent calls to the same topic ID will return new or updated documents that match the topic query.*

In this post we are going to work with [Java Streaming Api](http://lucene.apache.org/solr/6_1_0/solr-solrj/org/apache/solr/client/solrj/io/stream/package-summary.html).
One description of the [TopicStream function](http://lucene.apache.org/solr/6_1_0/solr-solrj/org/apache/solr/client/solrj/io/stream/TopicStream.html) is :

```java
TopicStream(String zkHost, String checkpointCollection, String collection, String id,
long checkpointEvery, SolrParams params);
```
- **zkHost**: zookeeper host.
- **checkpointColletion**: the collection where the topic checkpoints are stored.
- **collection**: the collection where data are stored and topic query will be run on.
- **id**: id for the topic. This id is unique.
- **checkpointEvery**: checkpoint every X tuples, if set -1 it will checkpoint after each run.
- **params**: query parameters for the TopicStream.

## Daemon function ##

The [daemon function](https://cwiki.apache.org/confluence/display/solr/Streaming+Expressions#StreamingExpressions-daemon) belongs to the [Stream Decorators](https://cwiki.apache.org/confluence/display/solr/Streaming+Expressions#StreamingExpressions-StreamDecorators). *Stream decorators wrap other stream functions or perform operations on the stream.*  This function wraps another function and runs it at intervals. We use this function to provide both with continuous pull streaming.

```java
DaemonStream(TupleStream tupleStream, String id, long runInterval, int queueSize);
```
- **tupleStream**: stream to run, in this case our TopicStream function
- **id**: the id of the daemon
- **runInterval**: the interval at which to run the internal stream
- **queueSize**:the internal queue size for the daemon stream.

## Collections ##

For this example I've used one collection ( the data for these collections are : [mock collection]({{ site.url }}/resources/mock_data.json)), generated with [mockaroo](https://www.mockaroo.com/).
The scheme of this collection is:

- **id**: document id (number).
- **price**: not used in this example (number).
- **area**: not used in this example (number).
- **type**: field to query (String). The types are colors.

## Sample code ##
Here we have the simple code! (This code is based on the [continous pull sample code](https://cwiki.apache.org/confluence/display/solr/Streaming+Expressions#StreamingExpressions-daemon) with some changes).

```java
package rodrite.github.io.solr.streaming;

import java.io.IOException;
import java.util.HashMap;
import java.util.Map;

import org.apache.solr.client.solrj.io.SolrClientCache;
import org.apache.solr.client.solrj.io.Tuple;
import org.apache.solr.client.solrj.io.stream.DaemonStream;
import org.apache.solr.client.solrj.io.stream.StreamContext;
import org.apache.solr.client.solrj.io.stream.TopicStream;
import org.apache.solr.common.params.MultiMapSolrParams;
import org.apache.solr.common.params.SolrParams;

public class StreamTopic
{
    public static void main( String[] args ) {
    	String zookeeperHost = "localhost:9983";
    	StreamContext context = new StreamContext();
  		SolrClientCache cache = new SolrClientCache();
  		context.setSolrClientCache(cache);

		Map<String, String[]> topicQueryParams = new HashMap<String, String[]>();
		topicQueryParams.put("q",new String[]{"Red"});    // The query for the topic
		topicQueryParams.put("rows", new String[]{"500"});// number of rows to fetch during each run
		topicQueryParams.put("fl", new String[]{"*"});    // The field list to return with the documents

		SolrParams solrPararms =  new MultiMapSolrParams(topicQueryParams);

		TopicStream topicStream = new TopicStream(zookeeperHost,  // Host address for the zookeeper
                             "checkpoints",   // The collection to store the topic checkpoints
                             "gettingstarted",// The collection to query for the topic records
                             "topicId",       // The id of the topic
                             -1,              // checkpoint every X tuples, if set -1 it will checkpoin after each run.
                             solrPararms);    // The query parameters for the TopicStream

		DaemonStream daemonStream = new DaemonStream(topicStream, // The underlying stream to run.
                             "test",    // The id of the daemon
                              1000,        // The interval at which to run the internal stream
                              500);        // The internal queue size for the daemon stream. Tuples will be placed in the daemonStream.setStreamContext(context);
		daemonStream.open();
		workWithTuples(daemonStream); //here we work with the Tuples
		daemonStream.close();
  }

	private static void workWithTuples(DaemonStream daemonStream) {
		while(true) {
		    Tuple tuple;
			try {
				tuple = daemonStream.read();
			    if(tuple.EOF) {
			        break;
			    } else {
			        System.out.println(tuple.fields.toString());
			    }
			} catch (IOException e) {
				e.printStackTrace();
			}
		}
	}
}
```

The above code shows that our query our query is **q=Red**, this mean that the topicStream function reads all tuples with **type=Red**.

In order to test this run the script and update the collection with [these documents]({{ site.url }}/resources/mock_input.json).

```json
[{"id":1,"price":61133,"area":759905,"type":"Red"},
{"id":2,"price":802969,"area":23415,"type":"Red"},
{"id":3,"price":611457,"area":529613,"type":"Blue"},
{"id":4,"price":621932,"area":300604,"type":"Red"},
{"id":5,"price":594234,"area":975902,"type":"Black"}]
```

The script output should be this:

```json
[{"id":1,"price":61133,"area":759905,"type":"Red"},
{"id":2,"price":802969,"area":23415,"type":"Red"},
{"id":4,"price":621932,"area":300604,"type":"Red"}]
```

## For more information ##

[Implement Saved Searches a la ElasticSearch Percolator](https://issues.apache.org/jira/browse/SOLR-4587)

[Add checksum to the TopicStream to ensure delivery of all documents within a Topic](https://issues.apache.org/jira/browse/SOLR-8709)
