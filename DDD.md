# Detailed Design Document (DDD)

## Project: Earnings-V8 Embedding Pipeline

---

## 1. Overview
This document provides a detailed design for the Earnings-V8 embedding pipeline, which processes PDF and Excel files, generates embeddings using AI Core, and stores them in a HANA database. The system is designed to run both locally and on SAP BTP Cloud Foundry.

---

## 2. Architecture

### 2.1 High-Level Components
- **API Server** (`server.py`): Flask-based REST API for file upload, embedding generation, and status endpoints.
- **Embedding Pipeline** (`embedding_storer.py`): Orchestrates file validation, embedding generation, and storage.
- **File Processors** (`pdf_processor.py`, `excel_processor.py`): Extracts and processes content from PDF and Excel files.
- **Database Layer** (`db_connection.py`): Manages HANA DB connections, schema, and vector store initialization.
- **Credential Management** (`destination_srv.py`): Handles extraction of credentials from environment or Cloud Foundry services.
- **Logging** (`logger_setup.py`): Centralized logging for all modules.

---

## 3. Data Flow

1. **User/API uploads files** via `/api/upload` endpoint.
2. **Metadata** is stored in the HANA DB (`store_metadata_in_hana`).
3. **User/API triggers embedding generation** via `/api/generate-embeddings`.
4. **Files are downloaded** and categorized (transcript, non-transcript, images).
5. **`process_and_store_embeddings`** is called for each file or directory:
    - Validates directory and files.
    - Categorizes files (PDF/Excel).
    - Computes file hashes and checks for changes.
    - Initializes embedding model (AI Core).
    - Processes PDFs and Excels for embeddings.
    - Stores embeddings in HANA DB tables.
    - Removes duplicates.
6. **Status is updated** and returned to the user.

---

## 4. Key Modules & Functions

### 4.1 `server.py`
- **Flask App**: Handles API endpoints for upload, embedding generation, health check, and status.
- **Key Functions**:
    - `upload_file()`: Handles file uploads and metadata storage.
    - `generate_embeddings()`: Orchestrates the embedding pipeline.
    - `health_check()`, `get_status()`: Monitoring endpoints.
- **Inputs**: HTTP requests, files, API payloads.
- **Outputs**: API responses, logs.

### 4.2 `embedding_storer.py`
- **Pipeline Orchestration**: Main logic for processing and storing embeddings.
- **Key Functions**:
    - `process_and_store_embeddings(directory_path, force_overwrite_files, model_name)`
    - `store_embeddings(vector_store, texts, embeddings, metadatas)`
    - `remove_duplicates(table_name)`
- **Inputs**: Directory path, model name, overwrite flags.
- **Outputs**: Embeddings in DB, logs.

### 4.3 `pdf_processor.py` / `excel_processor.py`
- **File Processing**: Extracts content and generates embeddings for PDFs and Excels.
- **Key Functions**:
    - `process_pdf_with_embeddings(pdf_path, model_name)`
    - `process_all_excel(directory_path, model_name)`
- **Inputs**: File paths, model name.
- **Outputs**: Embedding objects, logs.

### 4.4 `db_connection.py`
- **DB Connection Pool**: Manages HANA DB connections and sets schema.
- **Key Functions**:
    - `get_db_connection()`, `release_db_connection(conn)`
    - `load_vector_stores()`
- **Schema Handling**: Sets schema using `DEF_SCHEMA` from `env_config.py`.

### 4.5 `destination_srv.py`
- **Credential Extraction**: Loads credentials from `VCAP_SERVICES` or environment.
- **Key Functions**:
    - `get_destination_service_credentials(vcap_services)`
    - `extract_hana_credentials(config)`
    - `extract_aicore_credentials(config)`

---

## 5. Database Design

- **Schema**: Set dynamically via `DEF_SCHEMA` (from env or destination config).
- **Tables** (see `env_config.py`):
    - `transcript`: PDF transcript embeddings
    - `non_transcript`: PDF non-transcript embeddings
    - `excel_non_transcript`: Excel embeddings
    - `FILE_METADATA`: File upload metadata
- **Columns**: Typically include `VEC_TEXT`, `VEC_VECTOR`, `VEC_META` (JSON with file info, hash, page, etc.)

---

## 6. Error Handling & Logging
- All major steps log start, success, and error messages.
- Early returns and exceptions are logged with context.
- API returns clear error messages for user-facing issues.

---

## 7. Configuration & Environment
- **env_config.py**: Central place for table names, model config, and `DEF_SCHEMA`.
- **.env**: Used for local development to provide `VCAP_SERVICES` and other secrets.
- **Cloud Foundry**: Injects `VCAP_SERVICES` automatically.

---

## 8. Extensibility
- New file types: Add new processor modules and update categorization logic.
- New embedding models: Update model initialization in `embedding_storer.py` and `db_connection.py`.
- Additional endpoints: Add to `server.py` as needed.

---

## 9. Security
- XSUAA authentication is enforced on protected endpoints.
- Credentials are never hardcoded; always loaded from environment or service bindings.
- File uploads are validated for type and size.

---

## 10. Deployment Notes
- **Local**: Requires valid `.env` with `VCAP_SERVICES` and access to HANA/AIC endpoints.
- **Cloud Foundry**: All credentials and endpoints are injected; file storage is ephemeral.
- **Logs**: Written to `logs/earnings_analysis.log` (local or container filesystem).

---

## 11. Sequence Diagram (Textual)
1. User uploads file → `/api/upload` → File saved, metadata stored in HANA
2. User triggers embedding → `/api/generate-embeddings`
3. Server downloads files, categorizes, and calls `process_and_store_embeddings`
4. Embeddings generated and stored in HANA
5. Duplicates removed
6. Status returned to user

---

## 12. File Map
- `server.py`: API endpoints, orchestration
- `embedding_storer.py`: Embedding pipeline
- `pdf_processor.py`, `excel_processor.py`: File processing
- `db_connection.py`: DB connection and schema
- `destination_srv.py`: Credential management
- `env_config.py`: Config and schema
- `logger_setup.py`: Logging
- `requirements.txt`: Dependencies
- `SDD.md`, `DDD.md`: Documentation

---

## 13. Inputs & Outputs
- **Inputs**: Files (PDF, Excel, etc.), API requests, environment variables
- **Outputs**: Embeddings in HANA, logs, API responses

---
