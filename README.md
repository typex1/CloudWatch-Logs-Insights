# CloudWatch Logs Insights Queries

Good example queries to gain more output from Logs Insights.

## Lambda

Filter for the REPORT line only and then return two fields: requestId and billdDuration:
```
filter @type = "REPORT"
| fields @requestId, @billedDuration
| sort by @billedDuration desc
```
You can see that @maxMemoryUsed is lower than the @memorySize provisioned - good starting point to adjust the memory setting!

Determine the amount of overprovisioned memory:
```
filter @type = "REPORT"
| stats max(@memorySize / 1024 / 1024) as provisonedMemoryMB,
  min(@maxMemoryUsed / 1024 / 1024) as smallestMemoryRequestMB,
  avg(@maxMemoryUsed / 1024 / 1024) as avgMemoryUsedMB,
  max(@maxMemoryUsed / 1024 / 1024) as maxMemoryUsedMB,
  provisonedMemoryMB - maxMemoryUsedMB as overProvisionedMB
```

Visualizing Lambda cold start delays - parse AWS Lambda logs to identify the frequency and severity of Lambda cold start delays. Since AWS Lambda logs a line that indicates the total "Init Delay," first filter for messages that include a cold start, and then summarize those statistics.
```
fields @timestamp, @message
| filter @message like /Init Duration/
| parse 'Init Duration: * ms' as initDuration
| stats avg(initDuration), max(initDuration), count() by bin(1d)
```

Count a number of cold starts, average init time and maximum init duration of a Lambda function
```
filter @type="REPORT"
| fields @memorySize / 1000000 as memorySize
| filter @message like /(?i)(Init Duration)/
| parse @message /^REPORT.*Init Duration: (?<initDuration>.*) ms.*/
| parse @log /^.*\/aws\/lambda\/(?<functionName>.*)/
| stats count() as coldStarts, avg(initDuration) as avgInitDuration, max(initDuration) as maxIntDuration by functionName, memorySize
```

Checking Lambda performance - P90 latency, total invokes, and max latency for a 5 min time window in a table
```
filter @type = "REPORT"
| stats avg(@duration), max(@duration), min(@duration), pct(@duration, 90), count(@duration) by bin(5m)
```

Show the number of log events iin the log group for each log stream:
```
stats count(*) by @logStream
    | limit 100
```    
additional steps: choose the **Visualization** tab, from the **Line** dropdown list, select **Bar**

View latency statistsics for 5-minute intervals:
```
filter @type = "REPORT"
| stats avg(@duration), max(@duration), min(@duration) by bin(5m)
```

Logs Insights expression to filter for a specific request-ID in Lambda:
```
fields @timestamp, @message
| filter @message like /1b8c5a9e-5bc2-4286-94f4-59898c838037/
| sort @timestamp desc
| limit 20
```

Additionally, extract specific value:
```
fields @timestamp, @message
| filter @message like /value too/
| parse @message "value too *" as @value
#| stats count(*) by @userId
| sort @timestamp desc
| limit 20
```

Find 50 most recent errors
```
fields Timestamp, LogLevel, Message
| filter LogLevel == "ERR"
| sort @timestamp desc
| limit 50
```

Exclude informational logs to highlight only Lambda errors
```
fields @timestamp, @message
| sort @timestamp desc
| filter @message not like 'EXTENSION'
| filter @message not like 'Lambda Insights'
| filter @message not like 'INFO'
| filter @message not like 'REPORT'
| filter @message not like 'END'
| filter @message not like 'START'
```

## API Gateway

Find a non-200 error in API Gateway Execution Logs
```
fields @timestamp, @message, @requestId, @duration, @xrayTraceId, @logStream, @logStream
| filter
   @message like /fail/ or
   @message like /timed/ or
   @message like /X-Amz-Function-Error/ or
   @message like /tatus: 4/ or
   @message like /tatus: 5/
| sort @timestamp desc
```

## CloudTrail

Number of log entries by service, event type, and Region:
```
stats count(*) by eventSource, eventName, awsRegion
```

Find the number of records where an exception occurred while invoking the API UpdateTrail.
```
filter eventName="UpdateTrail" and ispresent(errorCode)
    | stats count(*) by errorCode, errorMessage
```

## Common queries

To parse and count fields:
```
fields @timestamp, @message
| filter @message like /User ID/
| parse @message "User ID: *" as @userId
| stats count(*) by @userId
```

Parse a log line that includes a story ID, which comprises the publisher ID followed by a publication ID and a unique ID. I can parse this line using the parse command and then graph how many stories per publication were received, grouping the results into 30-minute intervals.
```
filter @message like /STORY Created/
| parse 'STORY Created *-*-* *' as publisher, publication, story_id, external_id
| stats count() by bin(30m), publication
```

Find all logs for a given request ID or X-Ray trace ID
```
fields @timestamp, @message
| filter @message like /REQUEST_ID_GOES_HERE/
```

