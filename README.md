# Data-Engineering-Test
## Real-Time Event Analytics with Aggregation, Schema Evolution, and Data Quality

Overview:

This exercise simulates a real-world scenario where you need to ingest, process, aggregate, and visualize real-time event data using Kafka, Kafka Streams, ClickHouse, and Superset. You will also need to handle schema evolution and data quality issues. The focus is on building a robust and performant analytical pipeline.

Goal:

Demonstrate proficiency in setting up and configuring Kafka, Kafka Streams, ClickHouse, and Superset.

Implement a pipeline to ingest, transform, and aggregate data from Kafka into ClickHouse using Kafka Streams.

Create ClickHouse Materialized Views for pre-aggregation and query optimization.

Build a Superset dashboard to visualize both real-time and pre-aggregated data from ClickHouse.

Handle schema evolution gracefully.

Implement data quality checks and a dead-letter queue.

Instructions:

Setup & Dependencies:

Environment: Choose your environment (Docker Compose, local, cloud). Provide instructions.

ClickHouse: Install and configure ClickHouse (Docker image or direct install).

Kafka: Install and configure Kafka (single-node is sufficient).

Kafka Streams (or ksqlDB): Install and configure a Kafka Streams application runtime (e.g., using a Java IDE or building a Docker image). ksqlDB is a simplified alternative if the candidate is comfortable with SQL-like syntax for stream processing.

Superset: Install and configure Superset. Ensure connectivity to ClickHouse.

Data Generation:

Provided Data Generator (data_generator.py - Enhanced): Use the following Python script (already enhanced with schema evolution and bad data generation):

import json
import time
import random
import uuid
import argparse
from kafka import KafkaProducer

def generate_event(include_device_type=False, generate_bad_data=False):
    event_type = random.choice(["page_view", "button_click"])
    user_id = random.randint(1, 1000)
    if generate_bad_data and random.random() < 0.05:  # 5% chance of bad data
        user_id = -1 * user_id  # Invalid user ID
    page_url = f"/page/{random.randint(1, 100)}"
    button_id = str(uuid.uuid4())
    timestamp = int(time.time())
    country = random.choice(["US", "CA", "UK", "DE", "FR", "JP", "XX"]) # XX is invalid
    if generate_bad_data and random.random() < 0.05:
      country = "ZZ" # very invalid.

    event = {
        "event_type": event_type,
        "user_id": user_id,
        "page_url": page_url,
        "button_id": button_id,
        "timestamp": timestamp,
        "country": country
    }

    if include_device_type:
        event["device_type"] = random.choice(["mobile", "desktop", "tablet"])

    return json.dumps(event).encode('utf-8')


if __name__ == '__main__':
    parser = argparse.ArgumentParser(description='Generate and send events to Kafka.')
    parser.add_argument('--topic', type=str, default='my_events', help='Kafka topic to send events to.')
    parser.add_argument('--rps', type=int, default=10, help='Records per second to generate.')
    parser.add_argument('--duration', type=int, default=60, help='Duration (seconds) to run the generator.')
    parser.add_argument('--kafka_broker', type=str, default='localhost:9092', help='Address of kafka broker')

    args = parser.parse_args()

    producer = KafkaProducer(bootstrap_servers=[args.kafka_broker])

    start_time = time.time()
    events_sent = 0
    include_device_type = False
    try:
        while time.time() - start_time < args.duration:
            # Simulate schema evolution after 30 seconds
            if time.time() - start_time > 30:
                include_device_type = True

            for _ in range(args.rps):
                event = generate_event(include_device_type, generate_bad_data=True)
                producer.send(args.topic, event)
                events_sent += 1
            time.sleep(1)
        print(f"Sent {events_sent} events to topic {args.topic} in {args.duration} seconds.")

    except KeyboardInterrupt:
        print("Stopped by user.")
    finally:
        producer.close()
content_copy
download
Use code with caution.
Python

Task: Understand the data generator. Note the schema evolution (after 30 seconds) and potential for bad data.

Kafka Configuration:

Create Kafka topics: my_events, enriched_events, country_event_averages, dlq (dead letter queue).

Kafka Streams Application:

Data Transformation:

Consume from my_events.

Calculate a rolling average of event counts per country over a 5-minute window.

Handle schema evolution (adding device_type). Ensure older records without device_type are handled gracefully.

Implement data quality checks:

Invalid user_id (< 0) and invalid country codes ("ZZ", "XX").

Route invalid records to the dlq topic. Log errors.

Produce transformed, cleaned, and aggregated data to enriched_events and country_event_averages.

Note: the aggregation to the topic country_event_averages should output json records with the following fields country, window_start (DateTime), window_end (DateTime), and average_events (Float64).

ClickHouse Configuration:

Database & Tables:

Create a database: events_db.

Create tables:

events: Consumes from enriched_events (transformed data). Handle potentially Nullable device_type.

country_event_averages_raw: Consumes from country_event_averages.

dlq_table: Consumes from dlq.

Use JSONEachRow format for Kafka tables. Consider kafka_skip_broken_messages = 1.

Materialized View for DAU:

Create a table daily_active_users_final:

CREATE TABLE daily_active_users_final (
    event_date Date,
    country String,
    dau UInt32
) ENGINE = MergeTree()
ORDER BY (event_date, country);
content_copy
download
Use code with caution.
SQL

Create a Materialized View daily_active_users_mv consuming from the events table:

CREATE MATERIALIZED VIEW daily_active_users_mv
TO daily_active_users_final
AS SELECT
    toDate(timestamp) AS event_date,
    country,
    count(DISTINCT user_id) AS dau
FROM events
GROUP BY event_date, country;
content_copy
download
Use code with caution.
SQL

Data Ingestion:

Run the data generator (data_generator.py).

Start the Kafka Streams application.

Verify data ingestion into ClickHouse tables (events, country_event_averages_raw, dlq_table, and daily_active_users_final).

Superset Dashboard:

Connect Superset to ClickHouse.

Create datasets for: events, country_event_averages_raw, daily_active_users_final.

Create a dashboard with:

Line chart: Rolling average of event counts per country over time (from country_event_averages_raw).

Line chart: DAU per country over time (from daily_active_users_final).

Pie chart: Distribution of event types (from events).

Table: Top 5 most visited page URLs (from events).

Table or chart showing DLQ records (from dlq_table) to visualize data quality issues.

Deliverables:

Code:

docker-compose.yml (if used).

ClickHouse table creation scripts (SQL).

Kafka Streams application code (or ksqlDB scripts).

Superset configuration details.

Any setup scripts.

Documentation:

Instructions for setup and running.

Explanation of design choices: data types, materialized views, Kafka Streams logic, schema evolution strategy, data quality handling.

Screenshots of the Superset dashboard.

Issues encountered and resolutions.

Explanation of trade-offs between Kafka Streams real-time aggregation and ClickHouse Materialized Views. Justify the chosen approach.

Evaluation Criteria:

Correctness: End-to-end functionality. Data flows correctly, aggregations are accurate, schema evolution is handled, data quality checks work.

Completeness: All instructions followed. Documentation is thorough.

Code Quality: Well-structured, readable, maintainable code.

Performance: Efficient data ingestion, optimized queries, proper use of materialized views.

Understanding: Clear understanding of the technologies and their integration.

Troubleshooting: Effective problem-solving.

Aggregation: Accurate and efficient real-time and pre-aggregation. Justified choice of aggregation methods.

Schema Evolution: Graceful handling of schema changes without data loss or query errors.

Data Quality: Effective identification and handling of invalid data.

Example Kafka Streams Code Snippet (Conceptual):

// Example in Java using Kafka Streams DSL (replace with ksqlDB if preferred).
// This is a HIGHLY simplified example and needs significant error handling and detail
KStream<String, String> eventsStream = builder.stream("my_events");

KStream<String, JsonNode> parsedEvents = eventsStream.mapValues(json -> {
    ObjectMapper mapper = new ObjectMapper();
    try {
        return mapper.readTree(json);
    } catch (Exception e) {
        // Handle parsing errors (send to DLQ)
        return null;
    }
});

// Filter out null events (parsing errors)
KStream<String, JsonNode> validEvents = parsedEvents.filter((key, event) -> event != null);

//Data Quality
validEvents.filter((key, event) -> {
         //Bad Data Example
          if (event.get("user_id").asInt() < 0){
                //Send the record to DLQ
                return false;
              }
               return true;
        }).to("enriched_events");

// Create a 5-minute tumbling window
TimeWindows tumblingWindow = TimeWindows.of(Duration.ofMinutes(5));

KTable<Windowed<String>, Long> countryEventCounts = validEvents
    .groupBy((key, event) -> event.get("country").asText())
    .windowedBy(tumblingWindow)
    .count();

countryEventCounts.toStream()
    .map((windowedCountry, count) -> {
        String country = windowedCountry.key();
        long windowStart = windowedCountry.window().start();
        long windowEnd = windowedCountry.window().end();
        double averageEvents = (double) count; // Since it's already a count

        //Create JSON output string
        String output = String.format("{\"country\":\"%s\", \"window_start\":\"%s\", \"window_end\":\"%s\", \"average_events\":%f}",
                                       country, new Date(windowStart), new Date(windowEnd), averageEvents);
        return new KeyValue<>(country, output);
    })
    .to("country_event_averages", Produced.with(Serdes.String(), Serdes.String()));
content_copy
download
Use code with caution.
Java

Important Considerations:

Time Estimate: This is a substantial exercise. Estimate 16-32 hours depending on experience.

Partial Solutions: Be flexible. Assess the quality of completed components.

Documentation: Emphasize clear, concise, and well-reasoned documentation.

Provide Pointers: Offer links to Kafka Streams, ksqlDB, ClickHouse, and Superset documentation.

ksqlDB Alternative: If the candidate is familiar with SQL-like syntax, suggest ksqlDB as a potentially simpler alternative to writing a full Kafka Streams application. Provide examples and guidance.

Specific Versions: It is good to set a specific version of the frameworks you expect the candidate to work with.

By following these steps, you'll have an in-depth take-home exercise that thoroughly assesses a candidate's ability to design, implement, and troubleshoot a complete real-time analytical data pipeline. Good luck!
