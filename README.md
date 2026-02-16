
# ML Systems Architecture – Distributed Training & Model Lifecycle

This document summarizes the architecture and system design patterns used in a cloud-native ML platform. It outlines model training workflows, streaming inference, retraining automation, distributed ingestion strategies, and production reliability considerations.

---

## Focus Areas

- Event-driven architecture
- Asynchronous model training
- Continuous streaming inference
- Model artifact versioning
- Distributed ingestion & retraining workflows
- Production reliability patterns

---

## System Overview

The architecture follows a distributed microservices model:

- Stateless FastAPI service layer
- Background task processing using Celery
- Redis Pub/Sub for inter-service signaling
- BigQuery for large-scale analytical storage
- Cloud object storage for model artifact versioning
- Scheduled retraining workflows
- Secure secret management via GCP Secret Manager
- Dev/Prod environment isolation via service accounts and configuration

---

## Model Training Flow (Asynchronous, Event-Driven)

1. A user configures a decision-tree model via the Angular frontend and submits the form.
2. A FastAPI endpoint receives the request and immediately starts a Celery background task.
3. The API returns a success response while maintaining a WebSocket connection to stream task progress (no polling required).
4. The Celery task:
   - Fetches training data from BigQuery  
   - Loads data into a Pandas DataFrame  
   - Cleans duplicates and null values  
   - Executes `model.fit()` for training  
5. The trained model is serialized (e.g., `.h5`) and uploaded to cloud object storage.
6. Upon completion, Celery publishes a Redis message indicating training success.
7. FastAPI subscribes to the message and notifies the frontend via WebSocket that training is complete.

### Design Characteristics

- Non-blocking API responses
- Horizontal scalability
- Real-time UI feedback
- No client-side polling

---

## Streaming Inference & Action Loop (Asynchronous)

Predictions are generated continuously as new data arrives (not triggered by a user request):

1. A streaming data source is consumed by a long-running worker/service.
2. The worker loads the latest model artifact from cloud storage.
3. For each incoming data event/window, the service computes features and runs `model.predict()` to generate a signal.
4. Predictions feed downstream logic that produces actions and side effects (e.g., publish events, update state, emit notifications).
5. Side effects are executed asynchronously via queue/task mechanisms to decouple inference from external I/O and improve reliability.

### Design Characteristics

- Continuous 24/7 inference
- Decoupled execution of side effects
- Fault-tolerant, horizontally scalable processing
- Clear separation of training → artifact storage → inference → execution

---

## Scheduled Retraining Workflow

A weekly scheduler triggers automated retraining:

1. Scheduler invokes an API endpoint.
2. The API retrieves the latest model artifacts from cloud storage.
3. Updated data is pulled from BigQuery (maintained via scheduled ingestion jobs).
4. Models are retrained and re-serialized.
5. New versions are uploaded using folder-based or metadata-based versioning.
6. Lifecycle rules automatically clean up deprecated artifacts.

This separation of training, artifact storage, inference, and cleanup enables independent scaling and safer model evolution.

---

## Distributed Ingestion Optimization

Large-scale ingestion into BigQuery initially caused memory pressure and long execution times.

Optimization strategy:

- Partition bulk workloads into parallel Cloud Run Job executions
- Achieved ~10x faster processing
- Reduced memory utilization by ~90%
- Benchmarked Parquet vs CSV ingestion to reduce compute cost

---

## Reliability & Production Design

The architecture incorporates:

- Retry logic for transient failures
- Dead-letter queue patterns for failed events
- Structured logging for observability
- Stateless service design for horizontal scaling
- Environment-based configuration isolation
- Secure secret management (no credentials stored in source)

---

## Technology Stack

Python, FastAPI, Django, Celery, Redis  
BigQuery, Cloud Run Services, Cloud Run Jobs  
Cloud Storage (model artifacts)  
Docker, Service Accounts, Secret Manager  
TensorFlow/Keras, SciPy  

---
