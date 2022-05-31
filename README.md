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


## CloudTrail
