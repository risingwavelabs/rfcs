---
feature: Telemetry
authors:
  - 'ChengxuBian'
start_date: '2023/02/17'
---

# Telemetry for RisingWave

## Summary

The telemetry feature for RisingWave enables collection and analysis of metrics for running database, providing valuable insights into system performance and usage.

## Motivation

The telemetry feature is crucial for monitoring the health and performance of the RisingWave database. By collecting and analyzing metrics, we can gain a deeper understanding of the usage patterns, identify potential issues before they become critical, and optimize the performance of the system.
Design

The telemetry feature involves adding functionality to the RisingWave database to enable the collection of metrics and building a backend service to collect metrics This will include:

- Defining a set of metrics to be collected, such as query latency, number of queries executed, and number of errors encountered.

- Implementing code in the database to collect and record these metrics on a regular basis.

- Providing an API for collect and store the data

## Design

The telemetry feature is implemented by adding code to the database to collect and record metrics. The code is designed to be lightweight and **non-intrusive**, so it does NOT significantly impact the performance of the database.

Metrics are collected at regular intervals(24h), and are stored in a separate telemetry database. The telemetry database can be queried using a separate API to view and analyze the collected metrics.

## Unresolved Questions

1. Which specific metrics should be collected and how frequently should they be collected?
2. What impact will the telemetry feature have on the performance of the database?
3. How can we ensure that the collection of metrics does not compromise the security of the system?

## Alternatives

Alternative designs for the telemetry feature could include using an external monitoring tool to collect metrics or implementing a simpler system for collecting and reporting on database usage. However, these approaches may not provide the same level of detail and insight into the performance of the database as the proposed telemetry feature.

## Future Possibilities

In the future, the telemetry feature could be extended to include additional metrics or integrate with other monitoring and alerting systems. Optimization of the telemetry feature could also be done to minimize its impact on the performance of the database and to ensure that the collected metrics are secure and accessible only to authorized users. Additionally, the telemetry feature could be used to support automation and decision-making processes.
