# Query Profiler (Enterprise only)
Studio 2.2 includes a new functionality called [Auditing](Auditing.md). To understand how Profiler works, please read the [Profiler](https://github.com/orientechnologies/orientdb-docs/blob/master/Profiler.md) page on the [OrientDB Manual](http://orientdb.com/docs/last/index.html).

In the bottom section you can choose the server in order to investigate queries executed on it and manage the local cache.

## Query
This panel shows all the query executed on a specific server grouped by command content. For each query the following information are reported:
- `Type`, as the query type
- `Command`, as the content of the query
- `Users`, as the users who executed the query
- `Entries`, as the number of times the query it was executed
- `Average`, as the average required time by the queries
- `Total`, asthe total required time by all the queries
- `Max`, as the maximum required time
- `Min`, as the minimum required time
- `Last`, as the time required by the last query
- `Last execution`, as the timestamp of the last query execution


![Query](images/studio-queryprofiler-query.png)

## Command Cache
Through this panel you can manage the cache of the specific server and consult the cached query results.
You can filter the queries by the "Query" field and purge the whole cache.

![Command Cache](images/studio-queryprofiler-commandcache.png)
