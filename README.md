# TempoDB PHP API Client

The TempoDB PHP API Client makes calls to the [TempoDB API](http://tempo-db.com/api/).

# Classes

## TempoDB(key, secret, *host="api.tempo-db.com"*, *port=443*, *secure=true*)
Stores the session information for authenticating and accessing TempoDB. Your api key and secret is required. The TempoDB class
also allows you to specify the hostname, port, and protocol (http or https). This is used if you are on a private cluster.
The default hostname and port should work for the standard cluster.

All access to data is made through a client instance.
### Members
* key - api key (string)
* secret - api secret (string)
* hostname - hostname for the cluster (string)
* port - port for the cluster (int)
* secure - protocol to use (true=https, false=http)

## DataPoint(ts, value)
Represents one timestamp/value pair.
### Members
* ts - timestamp (DateTime)
* value - the datapoint's value (double, long, boolean)

## Series(id, key, *name=""*, *tags=array()*, *attributes=array()*)
Respresents metadata associated with the series. Each series has a globally unique id that is generated by the system
and a user defined key. The key must be unique among all of your series. Each series may have a set of tags and attributes
that can be used to filter series during bulk reads. Attributes are key/value pairs. Both the key and attribute must be
strings. Tags are keys with no values. Tags must also be strings.
### Members
* id - unique series id (string)
* key - user defined key (string)
* name - human readable name for the series (string)
* attributes - key/value pairs providing metadata for the series (array - keys and values are strings)
* tags - (array of strings)

## DataSet(series, start, stop, *data=array()*, *summary=null*)
Respresents data from a time range of a series. This is essentially a list of DataPoints with some added metadata. This is the object
returned from a query. The DataSet contains series metadata, the start/end times for the queried range, a list of the DataPoints
and a statistics summary table. The Summary table contains statistics for the time range (sum, mean, min, max, count, etc.)
### Members
* series - series metadata (Series)
* start - start time for the queried range (DateTime)
* stop - end time for the queried range (DateTime)
* data - datapoints (array of DataPoints)
* summary - a summary table of statistics for the queried range (Summary)

# Client API

## create_series(*key=""*)

Creates and returns a series object, optionally with the given key.

### Parameters
* key - key for the series (string)

### Returns
The newly created Series object

### Example

The following example creates two series, one with a given key of "my-custom-key", one with a randomly generated key.

    require('./tempodb.php');

    $tdb = new TempoDB("your-api-key", "your-api-secret");

    $series1 = $tdb->create_series("my-custom-key");
    $series2 = $tdb->create_series();

## get_series(*options=array()*)
Gets a list of series objects, optionally filtered by the provided parameters. Series can be filtered by id, key, tag and
attribute.

### Parameters
* ids - an array of ids to include (array of strings)
* keys - an array of keys to include (array of strings)
* tags - an array of tags to filter on. These tags are and'd together (array of strings)
* attributes - an array of key/value pairs to filter on. These attributes are and'd together. (array of key/value pairs)

### Returns
An array of Series objects

### Example

The following example returns all series with tags "tag1" and "tag2" and attribute "attr1" equal to "value1".

    require('./tempodb.php');

    $tdb = new TempoDB("your-api-key", "your-api-secret");

    $tags = array("tag1", "tag2");
    $attributes = array(
        "attr1" => "value1"
    );

    $series_list = $tdb->get_series(array("tags" => $tags, "attributes" => attributes));

## update_series(series)
Updates a series. The series id is taken from the passed-in series object. Currently, only tags and attributes can be
modified. The easiest way to use this method is through a read-modify-write cycle.
### Parameters
* series - the new series (Series)

### Returns
The updated Series

### Example

The following example reads the list of series with key *test1* (should only be one) and replaces the tags with *tag3*.

    require('./tempodb.php');

    $tdb = new TempoDB("your-api-key", "your-api-secret");

    $keys = array("test1");
    $series_list = $tdb->get_series(array("keys" => $keys));

    if ($series_list) {
        $series = $series_list[0];
        $series->tags = array("tag3");
        $tdb->update_series($series);
    }

## read(start, stop, *options=array()*)

Gets an array of DataSets for the specified start/stop times.

### Parameters
* start - start time for the query (DateTime)
* stop - end time for the query (DateTime)
* interval - the rollup interval (string)
* function - the rollup folding function (string)
* ids - an array of ids to include (array of strings)
* keys - an array of keys to include (array of strings)
* tags - an array of tags to filter on. These tags are and'd together (array of strings)
* attributes - an array of key/value pairs to filter on. These attributes are and'd together. (array of key/value pairs)
* tz - the time zone of the output data points (string)

The interval parameter allows you to specify a rollup period. For example, "1hour" will roll the data up on the hour using the provided function. The function parameter specifies the folding function
to use while rolling the data up. A rollup is selected automatically if no interval or function is given. The auto rollup interval
is calculated by the total time range (end - start) as follows:

* range <= 2 days - raw data is returned
* range <= 30 days - data is rolled up on the hour
* else - data is rolled up by the day

Rollup intervals are specified by a number and a time period. For example, 1day or 5min. Supported time periods:

* min
* hour
* day
* month
* year

Supported rollup functions:

* sum
* max
* min
* avg or mean
* stddev (standard deviation)
* count

You can also retrieve raw data by specifying "raw" as the interval. The series to query can be filtered using the
remaining parameters.

### Returns
An array of DataSets

### Example

The following example returns an array of series from 2012-01-01 to 2012-01-02 for the series with key "my-custom-key",
with the maximum value for each hour.

    require('./tempodb.php');

    $tdb = new TempoDB("your-api-key", "your-api-secret");

    $start = new DateTime("2012-01-01");
    $end = new DateTime("2012-01-02");
    $keys = array("my-custom-key");

    $data = $tdb->read($start, $end, array("keys" => $keys, "interval" => "1hour", "function" => "max"));

## read_id(series_id, start, stop, *options=array()*)

Gets a DataSet by series id. The id, start, and stop times are required. The same rollup rules apply as for the multi series
read (above).

### Parameters
* series_id - id for the series to read from (string)
* start - start time for the query (DateTime)
* stop - end time for the query (DateTime)
* interval - the rollup interval (string)
* function - the rollup folding function (string)
* tz - the time zone of the output data points (string)

### Returns

A DataSet

### Example

The following example reads data for the series with id "38268c3b231f1266a392931e15e99231" from 2012-01-01 to 2012-02-01 and
returns a minimum datapoint per day.

    require('./tempodb.php');

    $tdb = new TempoDB("your-api-key", "your-api-secret");

    $start = new DateTime("2012-01-01");
    $end = new DateTime("2012-01-02");

    $data = $tdb->read_id("38268c3b231f1266a392931e15e99231", $start, $end, array(interval" => "1day", "function" => "min"));

## read_key(series_key, start, stop, *options=array()*)

Gets a DataSet by series key. The key, start, and stop times are required. The same rollup rules apply as for the multi series
read (above).

### Parameters
* series_key - key for the series to read from (string)
* start - start time for the query (DateTime)
* stop - end time for the query (DateTime)
* interval - the rollup interval (string)
* function - the rollup folding function (string)
* tz - the time zone of the output data points (string)

### Returns

A DataSet

### Example

The following example reads data for the series with key "my-custom-key" from 2012-01-01 to 2012-02-01 and
returns a minimum datapoint per day.

    require('./tempodb.php');

    $tdb = new TempoDB("your-api-key", "your-api-secret");

    $start = new DateTime("2012-01-01");
    $end = new DateTime("2012-01-02");

    $data = $tdb->read_key("my-custom-key", $start, $end, array(interval" => "1day", "function" => "min"));

## write_id(series_id, data)

Writes datapoints to the specified series. The series id and an array of DataPoints are required.

### Parameters
* series_id - id for the series to write to (string)
* data - the data to write (array of DataPoints)

### Returns
Nothing

### Example

The following example write three datapoints to the series with id "38268c3b231f1266a392931e15e99231".

    require('./tempodb.php');

    $tdb = new TempoDB("your-api-key", "your-api-secret");

    $data = array(
        new DataPoint(new DateTime("2012-01-01T01:00:00"), 12.34),
        new DataPoint(new DateTime("2012-01-01T01:01:00"), 1.874),
        new DataPoint(new DateTime("2012-01-01T01:02:00"), 21.52)
    );

    $tdb->write_id("38268c3b231f1266a392931e15e99231", $data);

## write_key(series_key, data)
Writes datapoints to the specified series. The series key and an array of DataPoints are required. Note: a series will be created
if the provided key does not exist.

### Parameters
* series_key - key for the series to write to (string)
* data - the data to write (array of DataPoints)

### Returns
Nothing

### Example

The following example write three datapoints to the series with key "my-custom-key".

    require('./tempodb.php');

    $tdb = new TempoDB("your-api-key", "your-api-secret");

    $data = array(
        new DataPoint(new DateTime("2012-01-01T01:00:00"), 12.34),
        new DataPoint(new DateTime("2012-01-01T01:01:00"), 1.874),
        new DataPoint(new DateTime("2012-01-01T01:02:00"), 21.52)
    );

    $tdb->write_key("my-custom-key", $data);

## write_bulk(ts, data)
Writes values to multiple series for a particular timestamp. This function takes a timestamp and a parameter called data, which is an
array of arrays containing the series id or key and the value. For example:

    $data = array(
        array("id" => "01868c1a2aaf416ea6cd8edd65e7a4b8", "v" => 4.164),
        array("id" => "38268c3b231f1266a392931e15e99231", "v" => 73.13),
        array("key" => "your-custom-key", "v" => 55.423),
        array("key" => "foo", "v" => 324.991)
    );

### Parameters
* ts - a timestamp of the data points (DateTime)
* data - an array of dictionaries containing an id or key and the value

### Returns
Nothing

### Example

The following example writes datapoints to four separate series at the same timestamp.

    require('./tempodb.php');

    $tdb = new TempoDB("your-api-key", "your-api-secret");

    $ts = new DateTime("2012-01-08T01:21:00");
    $data = array(
        array("id" => "01868c1a2aaf416ea6cd8edd65e7a4b8", "v" => 4.164),
        array("id" => "38268c3b231f1266a392931e15e99231", "v" => 73.13),
        array("key" => "your-custom-key", "v" => 55.423),
        array("key" => "foo", "v" => 324.991)
    );

    $tdb->write_bulk($ts, $data);

## increment_id(series_id, data)
Increments the value of the specified series at the given timestamp. The value of the datapoint is the amount to increment. This is similar to a write. However the value is incremented by the datapoint value
instead of overwritten. Values are incremented atomically, so this is useful for counting events. The series id and an array of DataPoints are required.

### Parameters
* series_id - id for the series to increment (string)
* data - the data to write (array of DataPoints)

### Returns
Nothing

### Example

The following example increments three datapoints of the series with id "38268c3b231f1266a392931e15e99231".

    require('./tempodb.php');

    $tdb = new TempoDB("your-api-key", "your-api-secret");

    $data = array(
        new DataPoint(new DateTime("2012-01-01T01:00:00"), 1),
        new DataPoint(new DateTime("2012-01-01T01:01:00"), 2),
        new DataPoint(new DateTime("2012-01-01T01:02:00"), 1)
    );

    $tdb->increment_id("38268c3b231f1266a392931e15e99231", $data);

## increment_key(series_key, data)
Increments the value of the specified series at the given timestamp. The value of the datapoint is the amount to increment. This is similar to a write. However the value is incremented by the datapoint value
instead of overwritten. Values are incremented atomically, so this is useful for counting events. The series key and an array of DataPoints are required. Note: a series will be created
if the provided key does not exist.

### Parameters
* series_key - key for the series to increment (string)
* data - the data to write (array of DataPoints)

### Returns
Nothing

### Example

The following example increments three datapoints to the series with key "my-custom-key".

    require('./tempodb.php');

    $tdb = new TempoDB("your-api-key", "your-api-secret");

    $data = array(
        new DataPoint(new DateTime("2012-01-01T01:00:00"), 1),
        new DataPoint(new DateTime("2012-01-01T01:01:00"), 2),
        new DataPoint(new DateTime("2012-01-01T01:02:00"), 4)
    );

    $tdb->increment_key("my-custom-key", $data);

## increment_bulk(ts, data)
Increments values of multiple series for a particular timestamp. This function takes a timestamp and a parameter called data, which is an
array of arrays containing the series id or key and the value. For example:

    $data = array(
        array("id" => "01868c1a2aaf416ea6cd8edd65e7a4b8", "v" => 4),
        array("id" => "38268c3b231f1266a392931e15e99231", "v" => 2),
        array("key" => "your-custom-key", "v" => 1),
        array("key" => "foo", "v" => 1)
    );

### Parameters
* ts - a timestamp of the data points (DateTime)
* data - an array of dictionaries containing an id or key and the value

### Returns
Nothing

### Example

The following example increments datapoints to four separate series at the same timestamp.

    require('./tempodb.php');

    $tdb = new TempoDB("your-api-key", "your-api-secret");

    $ts = new DateTime("2012-01-08T01:21:00");
    $data = array(
        array("id" => "01868c1a2aaf416ea6cd8edd65e7a4b8", "v" => 4),
        array("id" => "38268c3b231f1266a392931e15e99231", "v" => 2),
        array("key" => "your-custom-key", "v" => 1),
        array("key" => "foo", "v" => 1)
    );

    $tdb->increment_bulk($ts, $data);
