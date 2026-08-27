# Conceptual data pipeline — Beejan Technologies

This document describes the conceptual design for a data engineering pipeline that unifies customer complaint data from social media, call center logs, SMS, and website forms, and turns it into a single, trustworthy source for reporting.
see below the conceptual design architecture 

![Architecture Diagram](Beejan_Complaint_Pipeline.png)


1. Source identification

Four source types are in scope, each with a different shape and cadence:

Social media — posts and mentions pulled from public APIs, semi-structured JSON, arrives continuously (near real-time).
Call center logs — agent notes and call transcripts exported as files (CSV/text), arrives in batches at the end of an agent's shift or daily.
SMS — structured text messages from a telecom gateway, arrives continuously (near real-time).
Website forms — structured submissions from a web form, arrives as discrete events throughout the day (near real-time to small batches).

Assumption: because two sources are naturally continuous and two are naturally batch, the pipeline is designed to support both modes rather than forcing everything onto one cadence.


2. Ingestion strategy

A single ingestion layer accepts data through three intake methods, chosen per source: continuous event capture for social media and SMS (so a complaint going viral or a spike in SMS complaints is visible within minutes), file pickup for call center log exports, and direct submission capture for website forms. Every ingested record is tagged with its source, an ingestion timestamp, and a unique record ID before it is handed off — this traceability is what lets later stages reconstruct "where did this complaint come from" during troubleshooting.

Assumption: near-real-time (minutes, not milliseconds) is an acceptable definition of "real-time" for this use case; true sub-second processing is not required for complaint handling.

3. Processing and transformation

Two transformation concerns are kept conceptually separate so each can evolve independently:

Cleaning and standardization — deduplicating repeated complaints (e.g. a customer who tweets and also calls), normalizing timestamps and languages, validating that required fields are present, and mapping every source into one canonical complaint schema (customer identifier, channel, timestamp, free-text content, resolved status).
Enrichment and classification — assigning each complaint to a category (network, billing, customer service, other), scoring sentiment, and joining in customer context (e.g. plan type, region) where available, so downstream reports can slice by category or customer segment.

Assumption: complaint categorization is done with an automated classification model rather than manual tagging, since manual tagging cannot keep pace with thousands of daily complaints; a manual review queue handles low-confidence cases.

4. Storage options

Both a data lake and a data warehouse are used, because they serve different needs. The lake holds two zones: a raw/landing zone that stores every record exactly as ingested (for audit and reprocessing) and a curated zone that stores cleaned, enriched records in a columnar format such as Parquet (efficient for large-scale analytical scans). The warehouse holds the same curated data reshaped into aggregated, query-ready tables, optimized for the dashboards and ad-hoc SQL that the reporting team already relies on.

5. Serving

The serving layer exposes the curated data through two paths: pre-built dashboards and scheduled reports for management and the reporting team, and a query/API layer for analysts or other internal systems that need on-demand access (for example, a customer service tool that wants to show a caller's recent complaint history). Access is read-only at this layer — no downstream consumer writes back into the pipeline.

6. Orchestration and monitoring

The pipeline runs on a schedule that matches each source's cadence: continuous processing for the streaming sources, and a scheduled batch run (e.g. hourly) that picks up file-based sources and pushes everything through cleaning, enrichment, and storage together. A monitoring layer watches every stage for two kinds of failure: pipeline failures (a stage stops running or errors out) and data quality failures (e.g. a sudden drop in volume, or a spike in unclassified complaints), and raises alerts to the data engineering team so issues are caught before they reach a report.

7. DataOps

Conceptually, the pipeline is expected to run in a managed, cloud-hosted environment rather than on individual laptops, with separate development and production environments so changes can be tested safely. Pipeline logic is version-controlled, and promotion from development to production follows a review step. Because complaints contain personal customer data, access to raw and curated data is role-restricted, and sensitive fields are masked for any consumer that does not need to see them.

Key assumptions
Near-real-time (minutes) is sufficient; sub-second latency is not a requirement.
An automated classification model can categorize the majority of complaints, with a manual queue for edge cases.
Call center data is available as structured or semi-structured file exports, not raw audio.
Customer identity can be resolved across channels well enough to deduplicate and enrich (e.g. via phone number or account ID).
Data volumes are large enough to justify a lake plus warehouse split, rather than a single store.
Challenges and open questions
Identity resolution — matching the same customer across social media, SMS, calls, and web forms is not always reliable and needs a defined matching strategy.
Classification accuracy — automated categorization will misclassify some complaints; the acceptable error rate and the design of the manual review queue still need to be defined.
Multilingual content — complaints may arrive in multiple languages, which affects both cleaning and classification.
Data volume growth — the design assumes today's volumes; a large spike (e.g. a network outage causing a surge in complaints) needs to be handled gracefully rather than causing pipeline backlog.
Privacy and retention — how long raw complaint data (which may contain personal information) should be retained is not yet defined.
Other notes

This design intentionally keeps raw and curated data separate so that if the classification or cleaning logic changes later, historical raw data can be reprocessed without needing to re-collect it from the original sources. The same reasoning is why cleaning and enrichment are shown as two distinct steps rather than one: standardization rules change rarely, while classification/enrichment logic is expected to be tuned often.
