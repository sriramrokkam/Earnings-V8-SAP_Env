# System Design Document (SDD)

## Project: Earnings-V8 Embedding Pipeline

---

## 1. High-Level Process Flow

### 1.1 Start the Server
- **File:** `server.py`
- **Function:** `main()` or Flask/FastAPI app entrypoint
- **Input:** HTTP request (API call to start embedding generation)
- **Output:** API response (success/failure, logs)

### 1.2 Load AI Credentials
- **File:** `embedding_storer.py`, `destination_srv.py`
- **Functions:**
  - `get_destination_service_credentials()`
  - `generate_token()`
  - `fetch_destination_details()`
  - `extract_aicore_credentials()`
- **Input:** Environment variable `VCAP_SERVICES`
- **Output:** AI Core credentials (dict)

### 1.3 Receive Embedding Generation Request
- **File:** `server.py`
- **Function:** API endpoint (e.g., `/generate_embeddings`)
- **Input:** Directory path, model name, force overwrite list (from API payload)
- **Output:** Triggers embedding process, returns status

### 1.4 Process and Store Embeddings
- **File:** `embedding_storer.py`
- **Function:** `process_and_store_embeddings(directory_path, force_overwrite_files, model_name)`
- **Input:** Directory path, force overwrite files, model name
- **Output:** Embeddings stored in DB, logs, status

#### 1.4.1 Validate Directory and Initialize Variables
- **File:** `embedding_storer.py`
- **Function:** `process_and_store_embeddings`
- **Input:** Directory path
- **Output:** List of files, error if not found

#### 1.4.2 Categorize Files (PDF/Excel)
- **File:** `embedding_storer.py`
- **Function:** `process_and_store_embeddings`
- **Input:** List of files
- **Output:** Sets of PDF and Excel files to process

#### 1.4.3 Compute File Hashes and Detect Changes
- **File:** `embedding_storer.py`
- **Function:** `compute_file_hash`, `get_existing_file_info_from_db`
- **Input:** File paths
- **Output:** Dict of file hashes, set of changed/new files

#### 1.4.4 Initialize Embedding Model
- **File:** `embedding_storer.py`
- **Function:** `init_embedding_model`
- **Input:** Model name, AI Core credentials
- **Output:** Embedding model instance

#### 1.4.5 Process PDF Files
- **File:** `embedding_storer.py`, `pdf_processor.py`
- **Function:** `process_pdf_task` (calls `process_pdf_with_embeddings`)
- **Input:** PDF file paths, model
- **Output:** Transcript and non-transcript embeddings (list of (doc, embedding) tuples)

#### 1.4.6 Process Excel Files
- **File:** `embedding_storer.py`, `excel_processor.py`
- **Function:** `process_excel_task` (calls `process_all_excel`)
- **Input:** Excel file paths, model
- **Output:** Excel embeddings (list of (doc, embedding) tuples)

#### 1.4.7 Store Embeddings in Database
- **File:** `embedding_storer.py`
- **Function:** `store_embeddings`, `delete_embeddings_for_file`
- **Input:** Embeddings, metadata, DB connection
- **Output:** Embeddings inserted into HANA tables

#### 1.4.8 Remove Duplicates
- **File:** `embedding_storer.py`
- **Function:** `remove_duplicates`
- **Input:** Table names
- **Output:** Deduplicated DB tables

#### 1.4.9 Logging and Error Handling
- **File:** `embedding_storer.py`, `server.py`
- **Function:** All major steps
- **Input:** All process state
- **Output:** Detailed logs for each step, error messages if any

---

## 2. Database Tables
- **transcript**: Stores transcript PDF embeddings
- **non_transcript**: Stores non-transcript PDF embeddings
- **excel_non_transcript**: Stores Excel embeddings
- **File:** Table names are defined in `env_config.py` as `TABLE_NAMES`

---

## 3. Key Functions and Their I/O

### 3.1 `process_and_store_embeddings`
- **Input:** `directory_path`, `force_overwrite_files`, `model_name`
- **Output:** None (side effect: DB insertions, logs)

### 3.2 `process_pdf_with_embeddings`
- **Input:** PDF file path, model name
- **Output:** List of embedding objects (or None)

### 3.3 `process_all_excel`
- **Input:** Directory path, model name
- **Output:** List of (doc, embedding) tuples

### 3.4 `store_embeddings`
- **Input:** Vector store instance, texts, embeddings, metadatas
- **Output:** None (side effect: DB insertions)

### 3.5 `remove_duplicates`
- **Input:** Table name
- **Output:** Number of duplicates removed

---

## 4. Error Handling
- All major steps log errors and return early if a critical failure occurs (e.g., missing directory, DB connection failure, model init failure).
- Errors are logged with context to help trace why the process may not reach the final step in `server.py`.

---

## 5. End-to-End Example
1. User/API calls `/generate_embeddings` in `server.py` with a directory path.
2. Credentials are loaded and validated.
3. `process_and_store_embeddings` is called.
4. Files are validated, categorized, and checked for changes.
5. Embedding model is initialized.
6. PDFs and Excels are processed for embeddings.
7. Embeddings are stored in the appropriate HANA tables.
8. Duplicates are removed.
9. Logs are written for each step and errors are reported if any.
10. API returns success/failure to the user.

---

## 6. File Map
- `server.py`: API entrypoint, triggers embedding process
- `embedding_storer.py`: Main pipeline logic, DB operations
- `pdf_processor.py`: PDF embedding extraction
- `excel_processor.py`: Excel embedding extraction
- `env_config.py`: Table names, model config
- `destination_srv.py`: AI Core credential management
- `db_connection.py`: DB connection utilities
- `logger_setup.py`: Logger configuration

---

## 7. Inputs and Outputs Summary
- **Inputs:** Directory path, model name, force overwrite list, environment variables (credentials)
- **Outputs:** Embeddings in DB, logs, API response

---

## 8. Logging
- Each step logs start, success, and error messages with context for traceability.
- Logs are written to the configured logger (see `logger_setup.py`).

---

## 9. Extensibility
- New file types or embedding models can be added by extending the processor modules and updating the pipeline logic in `embedding_storer.py`.

---

## 10. Contact
- For questions, see the code comments or contact the project maintainer.
