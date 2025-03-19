# MinishopHighLatency

This alert is triggered when the minishop application is underperforming.
This application is fully instrumented with opentelemetry and traces are available in Tempo.


# Root cause analysis

## 1. Confirm the alert

Always run the following prometheus range query. Use start and end time that surround the alert trigger time with +/- 15 minutes:

```
histogram_quantile(0.9, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))
```

This should show confirm the alert and give a sense of how critical the performance issue is.

## 2. Check traces

Because the minishop application is fully instrumented with opentelemetry, consult the Tempo traces.
Here is a good traceql query to get started because the entry point for most of the activity for the minishop is the backend-service:

```
{resource.service.name="backend-service" && duration>XXs}
```

Change the XXs above to a value slightly below the maximum number you received from the prometheus query. This means you will get traces that are the lowest performing.
For example if the prometheus data shows a max 90%ile http_request_duration_seconds at 8.581s, get the traces that have duration > 8s.

### 3. Check the logs

Once trace data points to one or more culprits, you can fetch the logs for that service, pod or node and identify why is the performance poor.

Do so with accessing loki logs.
