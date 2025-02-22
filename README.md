# Data-Engineering-Test

**Exercise Title:** Real-Time Event Analytics with Aggregation, Schema Evolution, and Data Quality

**Overview:**

This exercise simulates a real-world scenario where you need to ingest, process, aggregate, and visualize real-time event data using Kafka, Kafka Streams, ClickHouse, and Superset. You will also need to handle schema evolution and data quality issues. The focus is on building a robust and performant analytical pipeline.

**Goal:**

*   Demonstrate proficiency in setting up and configuring Kafka, Kafka Streams, ClickHouse, and Superset.
*   Implement a pipeline to ingest, transform, and aggregate data from Kafka into ClickHouse using Kafka Streams.
*   Create ClickHouse Materialized Views for pre-aggregation and query optimization.
*   Build a Superset dashboard to visualize both real-time and pre-aggregated data from ClickHouse.
*   Handle schema evolution gracefully.
*   Implement data quality checks and a dead-letter queue.

**Instructions:**

1.  **Setup & Dependencies:**

    *   **Environment:** Choose your environment (Docker Compose, local, cloud). Provide instructions.
    *   **ClickHouse:** Install and configure ClickHouse (Docker image or direct install).
    *   **Kafka:** Install and configure Kafka (single-node is sufficient).
    *   **Kafka Streams (or ksqlDB):** Install and configure a Kafka Streams application runtime (e.g., using a Java IDE or building a Docker image).  ksqlDB is a simplified alternative if the candidate is comfortable with SQL-like syntax for stream processing.
    *   **Superset:** Install and configure Superset. Ensure connectivity to ClickHouse.

2.  **Data Generation:**

    *   **Provided Data Generator (`data_generator.py` - Enhanced):** Use the following Python script (already enhanced with schema evolution and bad data generation):

        ```python
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
        ```

    *   **Task:** Understand the data generator. Note the schema evolution (after 30 seconds) and potential for bad data.

3.  **Kafka Configuration:**

    *   Create Kafka topics: `my_events`, `enriched_events`, `country_event_averages`, `dlq` (dead letter queue).

4.  **Kafka Streams Application:**

    *   **Data Transformation:**
        *   Consume from `my_events`.
        *   Calculate a rolling average of event counts per `country` over a 5-minute window.
        *   Handle schema evolution (adding `device_type`).  Ensure older records without `device_type` are handled gracefully.
        *   Implement data quality checks:
            *   Invalid `user_id` (< 0) and invalid `country` codes ("ZZ", "XX").
            *   Route invalid records to the `dlq` topic. Log errors.
        *   Produce transformed, cleaned, and aggregated data to `enriched_events` and `country_event_averages`.
        *   **Note:** the aggregation to the topic `country_event_averages` should output json records with the following fields `country`, `window_start` (DateTime), `window_end` (DateTime), and `average_events` (Float64).

5.  **ClickHouse Configuration:**

    *   **Database & Tables:**
        *   Create a database: `events_db`.
        *   Create tables:
            *   `events`: Consumes from `enriched_events` (transformed data). Handle potentially `Nullable` `device_type`.
            *   `country_event_averages_raw`: Consumes from `country_event_averages`.
            *   `dlq_table`: Consumes from `dlq`.
        *   Use `JSONEachRow` format for Kafka tables. Consider `kafka_skip_broken_messages = 1`.

    *   **Materialized View for DAU:**

        *   Create a table `daily_active_users_final`:

            ```sql
            CREATE TABLE daily_active_users_final (
                event_date Date,
                country String,
                dau UInt32
            ) ENGINE = MergeTree()
            ORDER BY (event_date, country);
            ```

        *   Create a Materialized View `daily_active_users_mv` consuming from the `events` table:

            ```sql
            CREATE MATERIALIZED VIEW daily_active_users_mv
            TO daily_active_users_final
            AS SELECT
                toDate(timestamp) AS event_date,
                country,
                count(DISTINCT user_id) AS dau
            FROM events
            GROUP BY event_date, country;
            ```

6.  **Data Ingestion:**

    *   Run the data generator (`data_generator.py`).
    *   Start the Kafka Streams application.
    *   Verify data ingestion into ClickHouse tables (`events`, `country_event_averages_raw`, `dlq_table`, and `daily_active_users_final`).

7.  **Superset Dashboard:**

    *   Connect Superset to ClickHouse.
    *   Create datasets for: `events`, `country_event_averages_raw`, `daily_active_users_final`.
    *   Create a dashboard with:
        *   Line chart: Rolling average of event counts per `country` over time (from `country_event_averages_raw`).
        *   Line chart: DAU per `country` over time (from `daily_active_users_final`).
        *   Pie chart: Distribution of event types (from `events`).
        *   Table: Top 5 most visited page URLs (from `events`).
        *   Table or chart showing DLQ records (from `dlq_table`) to visualize data quality issues.

8.  **Deliverables:**

    *   **Code:**
        *   Share a Github Repo with README on how to run the project to vikash@makeit.sg
        *   ClickHouse table creation scripts (SQL).
        *   Kafka Streams application code (or ksqlDB scripts).
        *   Superset configuration details.
        *   Any setup scripts.
    *   **Documentation:**
        *   Instructions for setup and running.
        *   Explanation of design choices: data types, materialized views, Kafka Streams logic, schema evolution strategy, data quality handling.
        *   Screenshots of the Superset dashboard.
        *   Issues encountered and resolutions.
        *   Explanation of trade-offs between Kafka Streams real-time aggregation and ClickHouse Materialized Views. Justify the chosen approach.

**Evaluation Criteria:**

*   **Correctness:** End-to-end functionality. Data flows correctly, aggregations are accurate, schema evolution is handled, data quality checks work.
*   **Completeness:** All instructions followed. Documentation is thorough.
*   **Code Quality:** Well-structured, readable, maintainable code.
*   **Performance:** Efficient data ingestion, optimized queries, proper use of materialized views.
*   **Understanding:** Clear understanding of the technologies and their integration.
*   **Troubleshooting:** Effective problem-solving.
*   **Aggregation:** Accurate and efficient real-time and pre-aggregation. Justified choice of aggregation methods.
*   **Schema Evolution:** Graceful handling of schema changes without data loss or query errors.
*   **Data Quality:** Effective identification and handling of invalid data.
