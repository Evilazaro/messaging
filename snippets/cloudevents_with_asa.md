I wrote the code, ChatGPT the text. I'll eventually write an actual article, but this will do:

### Consuming and Creating (Structured, JSON) CloudEvents in Azure Stream Analytics

Hey devs, today we're going to talk about working with structured JSON CloudEvents in Azure Stream Analytics. We'll be using some straightforward SQL-like queries to get this done, and I promise to keep the jargon to a minimum. Let's get started.

#### Setting Up Your Event Stream

We assume that the input is an Event Hub that multiplexes CloudEvents in structured JSON format. 

That means the events have JSON records that have specversion, source, time, type, data, and a few others. Here's how you can set up a basic stream:

```sql
WITH Events AS (
    SELECT  CAST(time AS datetime) AS time,
            type, 
            data,
            subject,
            source,
            id
    FROM [eventdata]
),
```

In this snippet, we're creating a stream named "Events" and selecting various attributes from our input alias `[eventdata]`. We're also casting the 'time' attribute to a datetime data type to work with it more effectively in later queries.

#### Working with Partitions and Grouping

Now, let's talk about how to work with partitions and grouping to organize your data into substreams based on time and context. Here's how you can do it:

```sql
WITH Events AS (
    SELECT  CAST(time AS datetime) AS time,
            type, 
            data,
            PartitionId,
            subject,
            source,
            id
    FROM [eventdata] TIMESTAMP BY CAST(time AS datetime) OVER PartitionId, subject PARTITION BY PartitionId 
),
```

In this query, we're using the `TIMESTAMP BY` clause to specify a custom timestamp using the 'time' attribute. We're also partitioning our data by 'PartitionId' and 'subject' to create substreams.

#### Filtering and Extracting Relevant Information

Next, we're going to filter our events based on the 'type' attribute and extract the relevant information. Here's how you can do it:

```sql
EventsTemperature AS (
    SELECT  *,
    data.temperature as temperature,
    data.device_type as deviceType,
    data.device_id as deviceId
    FROM [Events] PARTITION BY PartitionId
    WHERE type LIKE 'contoso.sensors.%' 
),
```

In this query, we're creating a new stream named "EventsTemperature" where we're selecting events that match a specific pattern in the 'type' attribute and extracting the temperature, device type, and device ID from the 'data' attribute.

#### Flattening the Data Content

If you want to flatten the data content of the CloudEvent, here's how you can do it:

```sql
EventsDeviceA AS (
    SELECT  data.*
    FROM [Events] PARTITION BY PartitionId
    WHERE type LIKE 'contoso.sensors.devicea%' 
),
```

Here, we're creating a stream named "EventsDeviceA" where we're selecting all attributes from the 'data' attribute for events that match a specific pattern in the 'type' attribute.

#### Aggregating Your Data

Now, let's move on to aggregating our data. Here's how you can tally up average temperatures:

```sql
WITH tallyup AS (
    SELECT PartitionId, deviceId, deviceType, AVG(temperature) AS AvgTemperature
    FROM EventsTemperature
    GROUP BY PartitionId, deviceId, deviceType, TumblingWindow(second, 1)
),
```

In this query, we're creating a stream named "tallyup" where we're calculating the average temperature using a tumbling window of one second.

#### Reassembling a Result Set into an Output CloudEvent

Finally, we're going to reassemble our result set into an output CloudEvent that carries a batch of data. Here's how you can do it:

```sql
SELECT '1.0' AS specversion,
        UDF.newguid('') AS id,
       'urn:com:contoso:myjob' AS source,
       System.Timestamp() as time,
       'contoso.reports.devices.temperatures' AS type,
      deviceType AS subject,
       COLLECT() as data
FROM tallyup PARTITION BY PartitionId
GROUP BY TumblingWindow(second, 1), PartitionId
```

In this query, we're creating a new CloudEvent with various attributes including a unique ID generated by a custom UDF named `newguid()`. We're also using the `COLLECT()` function to create a batch of data.

#### Creating a Custom UDF

Speaking of custom UDFs, here's how you can create the `newguid()` function:

```javascript
function main(str) {
    function generateGUID() {
        function s4() {
            return Math.floor((1 + Math.random()) * 0x10000).toString(16).substring(1);
        }
        return s4() + s4() + '-' + s4() + '-' + s4() + '-' + s4() + '-' + s4() + s4() + s4();
    }

    return generateGUID();
}
```

This JavaScript function generates a random GUID using a series of random numbers and returns it.

#### Wrapping Up

So there you have it, a straightforward guide to working with structured JSON CloudEvents in Azure Stream Analytics. Remember, the key to mastering this is practice, so don't hesitate to get your hands dirty and start writing some queries. Good luck!