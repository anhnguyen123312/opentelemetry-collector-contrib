# OpenTelemetry Collector Configuration Error Report

## Summary
This report identifies and provides fixes for configuration errors in the OpenTelemetry Collector deployment.

## Errors Found

### Error 1: Transform Processor - Invalid `drop()` function
**Location:** `processors.transform.trace_statements`

**Error Message:**
```
processors::transform: unable to parse OTTL statement "drop()": undefined function "drop"
```

**Root Cause:**
The `drop()` function does not exist in OTTL (OpenTelemetry Transformation Language). The transform processor is designed for **transforming** telemetry data, not for dropping/filtering it. For filtering, use the **filter processor** instead.

**Impact:**
- The collector fails to start
- No telemetry is processed

---

### Error 2: Filter Processor - Invalid Metric Context Path
**Location:** `processors.filter.metrics.metric`

**Error Message:**
```
processors::filter: unable to parse OTTL condition "name == \"http_server_duration\" and attributes[\"http.route\"] == \"/health\"":
segment "attributes" from path "attributes[http.route]" is not a valid path nor a valid OTTL keyword for the metric context
```

**Root Cause:**
In the `metrics.metric` context, you cannot access `attributes["http.route"]` directly because:
- The `metric` context operates at the **metric level** (name, type, description)
- Attributes are stored at the **datapoint level** (individual data points within a metric)

**Valid paths for `metrics.metric` context:**
- `metric.name`
- `metric.description`
- `metric.type`
- `resource.attributes`
- `instrumentation_scope.attributes`
- `metric.data_points`

**Impact:**
- The collector fails to start
- No telemetry is processed

---

## Solutions

### Solution 1: Remove Transform Processor (Recommended)

**Remove the entire transform processor section:**
```yaml
# DELETE THIS SECTION
transform:
  error_mode: ignore
  trace_statements:
    - context: span
      conditions:
        - duration < 10000000
      statements:
        - drop()
```

**Replace with filter processor additions:**

```yaml
filter:
  error_mode: ignore
  traces:
    span:
      # Existing filters...
      - 'attributes["http.target"] == "/oauth2/auth"'
      - 'attributes["http.route"] == "/oauth2/auth"'

      # NEW: Drop short-duration spans (< 10ms)
      - 'duration < 10000000'

      # NEW: Drop metrics endpoint
      - 'attributes["http.target"] == "/metrics"'

      # Existing filters...
      - 'attributes["user_agent.original"] == "kube-probe/*"'
```

**Why this approach?**
- The filter processor is designed for dropping/filtering telemetry
- Better performance and simpler configuration
- No need for complex transformation logic

---

### Solution 2: Fix Metric Filtering (Required)

**Option A: Use `datapoint` context (Recommended)**

```yaml
filter:
  error_mode: ignore
  metrics:
    datapoint:  # Changed from 'metric' to 'datapoint'
      - 'metric.name == "http_server_duration" and attributes["http.route"] == "/health"'
```

**Benefits:**
- Direct access to datapoint attributes
- More precise filtering (can filter individual datapoints)
- Clearer intent

---

**Option B: Use `metric` context with `HasAttrOnDatapoint` function**

```yaml
filter:
  error_mode: ignore
  metrics:
    metric:
      - 'name == "http_server_duration" and HasAttrOnDatapoint("http.route", "/health")'
```

**Benefits:**
- Drops the entire metric if any datapoint matches
- Useful when you want to remove the entire metric series

**Trade-offs:**
- Less granular than Option A
- May drop more data than intended

---

## Corrected Configuration

### Complete Filter Processor Section:

```yaml
processors:
  filter:
    error_mode: ignore
    traces:
      span:
        # OAuth2 healthcheck
        - 'attributes["http.target"] == "/oauth2/auth"'
        - 'attributes["http.route"] == "/oauth2/auth"'
        - 'IsMatch(attributes["http.url"], ".*oauth2.*")'

        # Coroot healthcheck
        - 'IsMatch(attributes["http.target"], ".*/health.*")'
        - 'IsMatch(attributes["http.target"], ".*/healthz.*")'
        - 'IsMatch(attributes["http.target"], ".*/alive.*")'
        - 'IsMatch(attributes["http.target"], ".*trueprofit-apm-http.*")'
        - 'IsMatch(attributes["http.url"], ".*coroot.*")'

        # Kubernetes probes
        - 'attributes["http.target"] == "/livez"'
        - 'attributes["http.target"] == "/readyz"'
        - 'attributes["user_agent.original"] == "kube-probe/*"'

        # NEW: Noise reduction - drop short spans
        - 'duration < 10000000'  # 10ms in nanoseconds

        # NEW: Drop metrics endpoint
        - 'attributes["http.target"] == "/metrics"'

    metrics:
      datapoint:  # FIXED: Changed from 'metric' to 'datapoint'
        # Drop health check metrics
        - 'metric.name == "http_server_duration" and attributes["http.route"] == "/health"'

        # Additional useful filters:
        - 'metric.name == "http_server_duration" and attributes["http.route"] == "/healthz"'
        - 'metric.name == "http_server_duration" and attributes["http.route"] == "/readyz"'
        - 'metric.name == "http_server_duration" and attributes["http.route"] == "/livez"'
```

### Updated Pipeline Configuration:

```yaml
service:
  pipelines:
    traces:
      receivers:
        - otlp
      processors:
        - memory_limiter
        - resourcedetection
        - filter          # Enhanced with new filters
        - resource
        # - transform    # REMOVED - not needed
        - batch
      exporters:
        - starrocks

    metrics:
      receivers:
        - otlp
      processors:
        - memory_limiter
        - resourcedetection
        - filter          # FIXED metric context
        - resource
        - batch
      exporters:
        - starrocks

    logs:
      receivers:
        - otlp
      processors:
        - memory_limiter
        - resourcedetection
        - resource
        - batch
      exporters:
        - starrocks
```

---

## Additional Recommendations

### 1. Add More Metric Filters

Consider adding these filters to reduce noise:

```yaml
filter:
  metrics:
    datapoint:
      # Health checks
      - 'attributes["http.route"] == "/health"'
      - 'attributes["http.route"] == "/healthz"'
      - 'attributes["http.route"] == "/readyz"'
      - 'attributes["http.route"] == "/live"'

      # Common monitoring endpoints
      - 'metric.name == "http_server_duration" and attributes["http.route"] == "/metrics"'
      - 'metric.name == "http_server_duration" and attributes["http.route"] == "/ping"'

      # High-cardinality attributes (optional)
      - 'attributes["http.status_code"] == 200'  # Only keep errors
```

### 2. Performance Tuning

Current batch settings are good, but consider:

```yaml
batch:
  send_batch_size: 2048      # Current: Good
  timeout: 10s               # INCREASED from 5s - allows better batching
  send_batch_max_size: 4096  # Current: Good
```

### 3. Resource Optimization

Your current resource limits are conservative. For production:

```yaml
resources:
  limits:
    cpu: 1000m              # INCREASED from 500m
    memory: 1Gi             # INCREASED from 512Mi
  requests:
    cpu: 200m               # Keep current
    memory: 512Mi           # INCREASED from 256Mi
```

---

## Validation Steps

### 1. Test Configuration Locally

```bash
# Validate configuration
otelcol-contrib --config /path/to/config.yaml --validate

# Dry run
otelcol-contrib --config /path/to/config.yaml --dry-run
```

### 2. Apply Changes

```bash
# Apply the updated configuration
kubectl apply -f otel-collector-config.yaml

# Check pod status
kubectl get pods -n monitoring -l app.kubernetes.io/name=otel-collector

# View logs
kubectl logs -n monitoring -l app.kubernetes.io/name=otel-collector --tail=100 -f
```

### 3. Verify Telemetry Flow

```bash
# Check if receiving data
kubectl port-forward -n monitoring svc/otel-collector 4317:4317

# Send test span
curl -X POST http://localhost:4318/v1/traces -d '{
  "resourceSpans": [{
    "resource": {"attributes": [{"key": "service.name", "value": {"stringValue": "test"}}]},
    "scopeSpans": [{
      "scope": {"name": "test"},
      "spans": [{
        "traceId": "12345678901234567890123456789012",
        "spanId": "1234567890123456",
        "name": "test-span",
        "startTimeUnixNano": 1234567890000000000,
        "endTimeUnixNano": 1234567890001000000
      }]
    }]
  }]
}'
```

---

## Common OTTL Context Reference

### For Traces (Span context):
```yaml
# Valid paths
- name                              # Span name
- duration                          # Span duration in nanoseconds
- attributes["key"]                 # Span attributes
- resource.attributes["key"]        # Resource attributes
- status.code                       # Status code
- parent_span_id                    # Parent span ID
```

### For Metrics (Datapoint context):
```yaml
# Valid paths
- metric.name                       # Metric name
- metric.type                       # Metric type
- attributes["key"]                 # Datapoint attributes
- resource.attributes["key"]        # Resource attributes
- value                             # Datapoint value (for gauge/sum)
- count                             # Count (for histogram)
```

### For Metrics (Metric context):
```yaml
# Valid paths
- name                              # Metric name
- type                              # Metric type
- description                       # Metric description
- resource.attributes["key"]        # Resource attributes ONLY
- HasAttrOnDatapoint("key", "value")  # Check if any datapoint has attribute
```

---

## Summary of Changes

1. **Removed** `transform` processor entirely (lines 117-127 in original)
2. **Changed** `metrics.metric` → `metrics.datapoint` in filter processor
3. **Added** new span filters to the filter processor
4. **Updated** pipeline configuration to remove transform processor

These changes will:
✅ Fix the configuration errors
✅ Improve filtering performance
✅ Reduce complexity
✅ Make the configuration more maintainable

---

## References

- [Filter Processor Documentation](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/processor/filterprocessor/README.md)
- [Transform Processor Documentation](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/processor/transformprocessor/README.md)
- [OTTL Functions Reference](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/pkg/ottl/ottlfuncs/README.md)
- [Metric Context Documentation](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/pkg/ottl/contexts/ottlmetric/README.md)

---

**Generated:** 2026-01-13
**Collector Version:** 1.18.0
**OpenTelemetry Collector Contrib:** Latest
