# Realtime Commuter Dashboard (GFTS-RT + Microsoft Fabric)

<img width="1451" height="854" alt="Dashboard" src="https://github.com/user-attachments/assets/66529aed-4345-4e3a-9083-85535a0170c3" />

This project demonstrates a complete realtime data pipeline built on Microsoft Fabric using GTFS-Realtime data from Stadtwerke Münster. It ingests live vehicle events via WebSocket, processes them through Azure Event Hub and Fabric Eventstream, stores them in a KQL database (Bronze/Silver/Gold), and visualizes realtime and historical transit performance using a realtime Dashboard.

The goal is to build a realtime Commuter Resilience Engine that helps understand delays, reliability, and operational patterns in public transport.

Originally, the dashboard was planned to be implemented in Power BI and deployed via a deployment pipeline. As my Power BI license expired, I created a Fabric Real-Time Dashboard instead and did not include a deployment pipeline.

# Data Source: Stadtwerke Münster GTFS-RT

The project uses the official realtime feed provided by Stadtwerke Münster.

The feed provides:
- Live vehicle positions
- Delay information (in seconds)
- Trip and stop metadata
- Continuous updates via WebSocket
- No API key required

WebSocket endpoint:
wss://websocket.busradar.conterra.de/

Each message is a FeatureCollection containing vehicle updates with fields such as:

- fahrzeugid (vehicle ID)
- linienid, linientext (line)
- fahrtbezeichner (trip ID)
- sequenz (stop sequence)
- visfahrplanlagezst (actual timestamp)
- ankunftziel, abfahrtstart (scheduled times)
- delay (delay in seconds)
- geometry.coordinates (lon/lat)

# Why a Python Bridge Was Required

Fabric Eventstream cannot directly connect to the Münster WebSocket because:
- The WebSocket does not expose a JSON schema
- Messages are nested FeatureCollections
- No handshake or metadata is provided
- Eventstream cannot infer structure or maintain a stable connection

To solve this, a Python WebSocket bridge was built. It:
- Opens the WebSocket
- Parses and normalizes each event
- Extracts coordinates, IDs, timestamps, delays
- Converts everything into clean JSON
- Sends each event to Azure Event Hub

Azure Event Hub acts as the ingestion buffer and provides a supported connector for Fabric Eventstream. This ensures a stable, schema-consistent stream for Fabric.

# Architecture Overview

1. Python WebSocket Bridge
- Connects to the Münster WebSocket
- Parses FeatureCollection messages
- Normalizes fields
- Sends events to Azure Event Hub

2. Azure Event Hub
- Buffers incoming events
- Provides durable, scalable ingestion
- Acts as the source for Fabric Eventstream

3. Microsoft Fabric Eventstream
- Reads from Event Hub
- Writes raw events into the KQL Bronze table
- Provides realtime preview and monitoring

4. Bronze Layer (Raw)
- Stores unmodified JSON events
- Ensures full fidelity for replay and debugging

5. Silver Layer (Cleaned & Typed)
- Converts types (int, datetime, decimal)
- Converts Unix timestamps
- Filters invalid coordinates
- Removes extreme negative delays
- Produces a clean, analytics-ready table

6. Gold Layer (Materialized View)

Aggregates hourly KPIs:
- event_count
- vehicle_count
- avg_delay_sec
- ontime_events
- delayed_events

Grouped by service_day, hour, line_id, line_text.

7. realtime Dashboard

The dashboard visualizes both realtime and past 24 hours transit performance:
- Live map of all active vehicles
- On-time performance (last 15 minutes)
- Average and maximum delay
- Delay distribution (24 hours)
- Most delayed lines
- Busiest hours
- Active vehicle count
- Last event timestamp

A Fabric Real-Time Dashboard was used built on the KQL database, so updates appear instantly as new events arrive.
