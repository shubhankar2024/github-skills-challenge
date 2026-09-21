# AIOps Service Monitoring Pipeline

## 1. AIOps Scenario

This project demonstrates a small AIOps workflow for monitoring a payment service. The pipeline reads service telemetry, detects unusual behavior, publishes anomaly events to an in-memory topic, and consumes those events for downstream handling.

## 2. Operational Data

The input data is stored in [`data/service_data.json`](data/service_data.json). It contains 10 one-minute observations for `payment-service` from `10:00` through `10:09` on September 20, 2026. Each record includes:

- Timestamp and service name
- Response time in milliseconds
- CPU and memory utilization percentages
- Log level and message

## 3. Observations

Most records show stable behavior: response time is between 120 and 150 ms, CPU is between 42% and 50%, and memory is between 51% and 57%. The normal records contain `INFO` messages indicating successful payment processing.

Two consecutive records are abnormal. At `10:05`, response time rises to 610 ms and the log reports a payment timeout. At `10:06`, response time rises to 640 ms, CPU reaches 94%, memory reaches 91%, and the log reports a database connection timeout.

## 4. Anomaly-Detection Findings

The detector uses these thresholds:

- Response time greater than 500 ms
- CPU utilization greater than 80%
- Memory utilization greater than 80%
- A log level of `ERROR`

It identifies two anomaly events:

1. `10:05`: high response time and an error log.
2. `10:06`: high response time, high CPU utilization, high memory utilization, and an error log.

## 5. Event-Processing Flow

1. `run_pipeline` loads the JSON records.
2. `AnomalyDetector` evaluates each record against the thresholds.
3. Detected events are sent through `EventProducer` to the `service-events` in-memory topic.
4. `EventConsumer` reads from that same topic.
5. The pipeline returns the processed-record count, detected anomalies, and consumed events.

## 6. Final Workflow Result

The final execution processed 10 records, detected 2 anomalies, and consumed 2 events. The consumed events correspond to the two timeout-related records at `10:05` and `10:06`.

## 7. Issues Identified and Corrected

- Added `src/__init__.py` so `src` can be imported as a Python package.
- Changed sibling imports to package-relative imports, including the consumer import that caused `ModuleNotFoundError`.
- Corrected the detector’s `ERROR` comparison so error logs are recognized.
- Connected the consumer to the producer’s topic. Previously, events were published to `service-events` but the consumer listened to a separate empty topic, causing zero consumed events.

## 8. Limitation and Possible Improvement

This demonstration uses fixed thresholds and an in-memory topic. It does not learn normal behavior, persist events, handle retries, or support multiple services at production scale. A possible improvement would be to use historical telemetry for adaptive baselines and a durable broker such as Kafka, with alert deduplication and retry handling.

## 9. Reproduction Steps

From the repository root:

```bash
pip install -r requirements.txt
python -m pytest -q
python -m src.aiops_pipeline
```

The test suite verifies the detector, producer, and consumer. The final command runs the demonstration using `data/service_data.json` and prints the workflow summary.

