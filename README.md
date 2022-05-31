# CloudWatch Logs Insights

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
