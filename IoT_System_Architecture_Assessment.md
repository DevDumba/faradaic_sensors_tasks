# IoT System Architecture Assessment

## Data Transfer Architecture

### Introduction

This part presents an architectural solution for transferring new data
from the network server's MySQL database to the business application,
with a focus on robustness, simplicity, and scalability.

#### System Context

In the current setup, IoT devices transmit environmental data to a
central network server operating in ALOHA mode. The server stores
incoming data in a MySQL database and occasionally sends configuration
packets back to the devices. After processing, the data is made
available to clients via a business application accessible through web
and mobile platforms. Both the network server and the business
application run on the same Virtual Private Server (VPS). The system is
designed for 10,000 devices, each sending one data packet per hour, with
transmissions evenly distributed over time.

#### Objective

The goal is to design a mechanism that reliably and efficiently
transfers new data entries from the network server's database to the
business application layer, while ensuring maintainability and
scalability in case of future growth.

### Architectural Approach

![](media/image1.png){width="6.5in" height="4.333333333333333in"}

a)  **Database Structure**

**MySQL** hosts two schemas:

- network_data: Contains raw sensor input from IoT devices (sensor_data
  table)

- business_data: Contains structured and pre-processed data used by the
  business application (processed_data table)

Both schemas reside in the same MySQL instance, eliminating the need for
remote connections or data replication mechanisms.

Each schema has a corresponding audit table to log any changes, allowing
for full data lineage and rollback capabilities if needed.

b)  **Data Producer**

- IoT devices transmit data to the network server, which stores records
  in network_data.sensor_data.

- Each row represents a new uplink message, identified via a unique
  auto-incrementing ID or timestamp.

c)  **Data Synchronizer: Spring Boot Cron Job**

- A Spring Boot component scheduled with \@Scheduled(cron = \"\...\")
  runs periodically (e.g., every minute).

- It queries the network_data.sensor_data table for new entries, using a
  tracked last_synced_id.

- After fetching new rows, it transforms and inserts them into
  business_data.processed_data.

d)  **Data Consumer: Business Application**

- The business application reads data from the business_data schema.

- The business application exposes a **simple internal API endpoint**
  (e.g., /ingest-data) to receive new data.

- Since data is already validated, transformed and structured during
  sync, the app can serve users with minimal processing time

e)  **Logging and Monitoring**

- The job logs execution time, row counts, and any exceptions.

### Estimated Throughput Metrics

a)  **Input load**

- **Number of devices**: 10,000

- **Transmission frequency**: 1 packet per hour per device

- **Distribution**: Evenly distributed across time (no spikes)

b)  **Calculated Packet Rate**

- **Per hour**: 10,000 packets

- **Per minute: 10,000 / 60 ≈ 167 packets**

- **Per second: 167 / 60 ≈ 3 packets**

c)  **Packet Size Estimation**

Example packet contains:

![A screen shot of a computer program AI-generated content may be
incorrect.](media/image2.png){width="3.86383530183727in"
height="2.276317804024497in"}

Estimated size: **\~150 bytes** per packet

  -----------------------------------------------------------------------
  **Interval**            **Packets**             **Estimated Data
                                                  Volume**
  ----------------------- ----------------------- -----------------------
  Per second              \~3 packets             \~450 bytes

  Per minute              \~167 packets           \~25 kilobytes

  Per hour                \~10,000 packets        \~1.5 megabytes
  -----------------------------------------------------------------------

d)  **System Impact**

- A cron job that syncs data every minute will process \~167 rows
  (\~25KB), which is negligible for a MySQL query and a Spring Boot
  application.

- No performance optimizations are needed at this scale, and the system
  can easily handle the expected load with room to spare.

### Potential Bottlenecks

a)  **MySQL Read/Write Performance**

Although each batch is small (a few hundred rows), increased data volume
(e.g., 10x more devices or higher frequency) could lead to:

- Slower queries

- Table locking

- Increased disk I/O

Mitigation:

- Use proper indexing

- Optimize queries for batch operations

- Audit tables provide historical tracking and can reduce the need to
  retain all records in main tables

b)  **Cron Job Execution**

If the sync interval is too short, or if the batch size grows, job
execution might **overlap** or run **longer than expected**, delaying
data delivery.

Mitigation:

- Upper mitigation from MySQL Read/Write Performance

- Monitor execution time

- Configure best fixed rate

c)  **Database Growth**

Over time, the sensor_data table in network_data schema may grow
significantly (millions of rows). This could lead to:

- Slower full-table scans

- Query performance degradation

Mitigation:

- Archive old data regularly

- Use time-based partitioning

- Audit tables and archiving strategy help separate live and historical
  data

d)  **Shared VPS Resource Constraints**

Both the network server and business application run on the **same
VPS**, sharing:

- CPU

- RAM

- DISK I/O

High usage from one process (e.g., business app under user load) may
starve the sync process or MySQL.

Mitigation: Monitor resource usage and consider moving to separate VPS

### Scalability Strategy for 100,000 Devices

The proposed architecture can scale to support 100,000 IoT devices with
moderate adjustments, while maintaining simplicity and robustness. Below
are the key areas to consider when scaling the system.

a)  **Increased Data Volume**

With 100,000 devices sending one packet per hour:

- **Hourly**: 100,000 packets (\~15 MB)

- **Per minute**: \~1,667 packets (\~250 KB)

- **Per day**: \~360 MB

This is still a manageable load for a MySQL-backed system, especially
with optimized queries and batch processing.

b)  **MySQL Performance Tuning**

- Ensure proper indexing (id, timestamp, device_id)

- Use efficient queries with pagination

- Perform batch inserts to reduce transaction overhead

- Audit tables remain useful for traceability without bloating main
  tables

c)  **Storage and Archiving**

Anticipate \~30 million rows added monthly to sensor_data.

- Archiving and audit logs offload historical data

d)  **VPS Resource Scaling**

Recommended actions:

- **Separate concerns**: deploy network server and business application
  on different VPS instances or containers

- Monitor CPU, memory, and I/O usage

- Upgrade hardware or move to cloud infrastructure if needed

e)  **Business Application Load**

With more data and more users:

- Implement **in-memory caching** for frequently accessed data

- Use **pagination** in API responses and frontend views

## Downlink Management System

#### 

#### Objective

The goal is to design a mechanism that reliably and efficiently
transfers new data entries from the network server's database to the
business application layer, while ensuring maintainability, scalability,
and auditability.

### System Design -- Downlink Management

To support the delivery of downlink packets from users to IoT devices,
the system introduces a scheduling mechanism that allows downlink
requests to be submitted from the business application and executed by
the network server once the device becomes eligible to receive them
(immediately after an uplink transmission).

a)  **Scheduling via Business Application**

- The client (user) creates a downlink request from the web or mobile
  app (e.g., "Restart Device")

- The request is submitted to the business application backend

- The business app then calls the network server's API to register the
  downlink.

b)  **Downlink Queue Table**

On the network server, table named queued_downlinks stores scheduled
packets.\
Example structure:

- device_id: target device

- payload: binary or base64-encoded command

- expire_at: optional expiry time

- status: pending / sent / failed / expired

c)  **Receiving Uplink Packets from Devices**

- The packet is stored in the sensor_data (in network_data schema).

- Immediately after processing the uplink, the server checks for pending
  downlink in the queued_downlinks table for the same device_id

- If pending downlink exists and the expire_at is still valid:

  - The server sends the downlink to the device during it's short
    receive window.

  - Updates the status of downlink to sent

- If no valid downlink exists, the device does not receive any response.

### API Design for Downlink Scheduling

**Endpoint**: POST /api/downlinks

**Request Headers**:

- Content-Type: application/json

- Authorization: Bearer \<token\>

**Request Body**:

![A screenshot of a computer program AI-generated content may be
incorrect.](media/image3.png){width="3.474750656167979in"
height="1.549714566929134in"}

**Response**

- 201 Created (Success):

![A screen shot of a computer AI-generated content may be
incorrect.](media/image4.png){width="3.497351268591426in"
height="1.4670089676290463in"}

- 400 Bad Request (Validation Error)

- 500 Internal Server Error

**API Behavior and Rules**

- The server validates:

  - If deviceId is known

  - If payload is non-empty and in a valid format

  - If expireAt is in the future

- The server does **not immediately deliver the packet** -- it only
  queues it

- Delivery is attempted **only after a device's next uplink**

- If the expiration time passes without delivery, the packet is marked
  as expired

- All lifecycle changes are inserted to audit table
