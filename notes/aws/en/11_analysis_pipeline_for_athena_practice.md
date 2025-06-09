# Real-time Data Query Practice with Apache Kafka, Apache Hudi, and AWS Athena

[English](./11_analysis_pipeline_for_athena_practice.md) | [繁體中文](../zh-tw/11_analysis_pipeline_for_athena_practice.md) | [日本語](../ja/11_analysis_pipeline_for_athena_practice.md) | [Back to Index](../README.md)

---

## Case Background

In modern microservice architectures, a common requirement is to synchronize data into a data lake whenever each microservice writes to its database, enabling subsequent analysts to query using AWS Athena. This case study introduces how to integrate Apache Kafka, Apache Hudi, and AWS Athena to implement an efficient, scalable real-time data query pipeline.

## Architecture Overview

1. **Microservice Data Writing**
   - Each microservice writes data to its respective database (e.g., Oracle, DynamoDB).
2. **Change Data Capture (CDC)**
   - Capture data change events through CDC tools (e.g., Oracle GoldenGate, Debezium, DynamoDB Streams) and push them to Apache Kafka.
3. **Apache Kafka Event Stream**
   - Kafka serves as an event broker, receiving and distributing all data change events.
4. **Data Processing and Apache Hudi Writing**
   - Use AWS Lambda or Kafka Connect to process Kafka events, transform and write to Apache Hudi (stored in AWS S3).
5. **AWS Athena Query**
   - Athena queries Hudi data on S3, supporting SQL analysis and reporting.

## Detailed Steps

### 1. Change Data Capture (CDC)
- **Oracle**: Use Oracle GoldenGate or Debezium to capture data changes and push to Kafka.
- **DynamoDB**: Enable DynamoDB Streams to capture table change events.

### 2. Kafka Broker Layer Design
- Deploy Kafka cluster to receive events from CDC tools.
- Design corresponding Kafka Topics for each data source.

### 3. Data Processing and Apache Hudi Writing
- **AWS Lambda Processing**: Lambda listens to Kafka Topics, triggers event conversion to Hudi format and writes to S3.
- **Kafka Connect Processing**: Deploy Hudi Sink Connector to directly write Kafka Topic data to Hudi tables, suitable for high-throughput requirements.

### 4. AWS Athena Query for Hudi Data
- Register Hudi table structure in AWS Glue Data Catalog.
- Use Athena to query Hudi tables on S3, supporting SQL analysis.

## Summary and Practical Recommendations

This architecture effectively implements real-time capture, processing, storage, and querying of data changes, suitable for scenarios requiring efficient data processing and analysis. Recommendations:
- Choose CDC tools based on database type and budget
- Design Kafka Topics considering data sources and partitioning strategies
- Hudi table design should select appropriate table types (Copy-on-write or Merge-on-read) based on query patterns
- Before Athena queries, optimize Glue Catalog and data partitioning

---

This case study can serve as a reference architecture for enterprises implementing real-time data lake queries.
