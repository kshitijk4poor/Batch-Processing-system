## Batch Processing API Specification

### 1. Overview

This document specifies an API endpoint and background processing system designed to handle batch processing of MongoDB documents using the OpenAI Batch API. The system accepts a JSONL file containing requests, validates it, submits it to OpenAI, polls for completion, validates the results against a user-provided schema, and updates the original MongoDB documents with the outcomes.

### 2. Core Requirements

*   Accept batch processing requests via an HTTP API endpoint.
*   Validate input JSONL files against OpenAI Batch API format and ensure model consistency.
*   Validate a user-provided JSON Schema for output processing.
*   Interact with OpenAI API to upload files and manage batch jobs.
*   Track the status of submitted batch jobs persistently.
*   Periodically poll OpenAI for batch job status updates.
*   Upon completion, retrieve results, validate AI response content against the user-provided schema.
*   Update source MongoDB documents with validated results or failure status.
*   Implement robust logging for observability.
*   Handle transient errors gracefully with retries.

### 3. API Endpoint

*   **Path:** `POST /process-batch`
*   **Authentication:** None. The endpoint is publicly accessible.
*   **Request Format:** `multipart/form-data` containing the following fields:
    *   `jsonl_file`: (Required) The `.jsonl` file where each line represents a single OpenAI API request (e.g., for `/v1/chat/completions`). Must adhere to OpenAI batch input format.
    *   `output_schema_json`: (Required) A string containing a valid JSON Schema definition. This schema will be used to validate the `content` part of the AI's response before updating the target MongoDB document.
    *   `mongodb_uri`: (Required) The MongoDB connection string (URI) for the database containing the documents to be processed and updated.
    *   `collection_name`: (Required) The name of the collection within the specified MongoDB database.
*   **Success Response:**
    *   **Status Code:** `202 Accepted`
    *   **Body:**
        ```json
        {
          "job_id": "<internal_job_id>" // The MongoDB ObjectId of the created record in the batch_jobs collection
        }
        ```
*   **Error Response (Validation):**
    *   **Status Code:** `400 Bad Request`
    *   **Body:**
        ```json
        {
          "error": "Validation Failed",
          "details": [
            {
              "type": "<error_type>", // e.g., "jsonl_format_error", "model_mismatch", "schema_validation_error", "db_connection_error", "custom_id_not_found"
              "line": <line_number>, // Optional: Line number in JSONL file if applicable
              "message": "<Specific error message>",
              "context": { /* Optional: Further details, e.g., Pydantic validation output */ }
            }
            // ... potentially more error objects ...
          ]
        }
        ```

### 4. Input Validation (Performed by API Endpoint)

1.  **JSONL File:**
    *   Each line must be a valid JSON object.
    *   Each JSON object must contain the keys: `custom_id` (string), `method` (string, must be "POST"), `url` (string, e.g., "/v1/chat/completions"), `body` (object).
    *   The `body` object must conform to the requirements of the specified OpenAI `url`.
    *   The `model` specified within the `body` object must be the same for all lines in the file. The model is determined from the first line.
2.  **Output Schema:**
    *   The `output_schema_json` string must parse as valid JSON and represent a valid JSON Schema structure.
3.  **MongoDB:**
    *   The provided `mongodb_uri` must be valid and allow connection.
    *   The specified `collection_name` must exist (or be creatable, TBD - assume exists for now).
    *   Each `custom_id` within the JSONL file must correspond to the `_id` of an existing document within the specified MongoDB collection.

### 5. Backend Processing Workflow

1.  **API Request Handling (`POST /process-batch`):**
    *   Receive `multipart/form-data`.
    *   Perform all validations outlined in Section 4. If any validation fails, return `400 Bad Request` immediately.
    *   Upload the validated `jsonl_file` to OpenAI Files API with `purpose="batch"`. Obtain the `input_file_id`.
    *   Create an OpenAI Batch job using the `input_file_id`, the endpoint derived from the JSONL `url`, and `completion_window="24h"`. Obtain the `openai_batch_id`.
    *   Create a new document in the `batch_jobs` MongoDB collection (see Section 6) storing `openai_batch_id`, `input_file_id`, `output_schema_json`, `mongodb_uri`, `collection_name`, `model`, initial status (`submitted`), and timestamps. Obtain the internal `_id` (`job_id`).
    *   Connect to the target MongoDB using `mongodb_uri`. For each `custom_id` in the input JSONL, update the corresponding document in `collection_name`, setting its status field (e.g., `ai_status`) to an "in progress" value (e.g., `in_progress`).
    *   Return `202 Accepted` with the internal `job_id`.

2.  **Polling and Result Processing (Separate Scheduled Script):**
    *   A separate script/service runs periodically (triggered by e.g., Google Cloud Scheduler).
    *   Query the `batch_jobs` collection for documents where `status` is not `completed` or `failed`.
    *   For each active job:
        *   Attempt to retrieve the batch status from OpenAI using `openai_batch_id` (`client.batches.retrieve(...)`). Implement retries (up to 3 attempts with exponential backoff) for transient network/API errors. If retries fail, log the error and proceed to the next job (relying on the next scheduled run).
        *   Update the `openai_status` and `updated_at` fields in the `batch_jobs` document.
        *   **If `openai_status` is `completed`:**
            *   Update `batch_jobs` status to `processing`.
            *   Retrieve `output_file_id` and `error_file_id` from the OpenAI batch object and store them in the `batch_jobs` document.
            *   Download the content of the `output_file_id` from OpenAI Files API (with retries).
            *   If `error_file_id` exists, download its content (with retries).
            *   Retrieve `output_schema_json`, `mongodb_uri`, `collection_name` from the `batch_jobs` document.
            *   Connect to the target MongoDB.
            *   Parse the output file JSONL. For each result line:
                *   Identify the `custom_id`.
                *   Find the corresponding document in the target collection.
                *   Check if the line contains a `response` or an `error`.
                *   **If `response`:**
                    *   Extract the primary AI response content (e.g., `response.body.choices[0].message.content`).
                    *   Validate this extracted content against the `output_schema_json`.
                    *   **If valid:** Perform a MongoDB update on the target document: `$push` an object `{ "event_response": <validated_content>, "updated": <current_timestamp> }` into the `event_response` array field, and `$set` the status field (e.g., `ai_status`) to `completed`. Log success (INFO).
                    *   **If invalid:** Perform a MongoDB update on the target document: `$set` the status field (e.g., `ai_status`) to `failed`. *Do not* modify the `event_response` field. Log the validation failure (WARN) including `custom_id` and validation error details.
                *   **If `error`:** Perform a MongoDB update on the target document: `$set` the status field (e.g., `ai_status`) to `failed`. Log the OpenAI error details (WARN) including `custom_id`.
            *   Log any content retrieved from the `error_file_id` (WARN).
            *   Update the `batch_jobs` document status to `completed`.
        *   **If `openai_status` is `failed` or `expired`:**
            *   Update the `batch_jobs` document status to `failed`.
            *   (Optional/Recommended) Consider iterating through associated `custom_id`s (perhaps stored temporarily or queried) and updating their status in the target collection to `failed`. Log the event (WARN).
        *   **If `openai_status` is `cancelling` or `cancelled`:**
            *   Update the `batch_jobs` document status accordingly. Log the event (INFO).
        *   **Other statuses (`validating`, `in_progress`, `finalizing`):** Simply update `openai_status` in `batch_jobs` and wait for the next polling cycle.

### 6. Data Schemas

*   **`batch_jobs` MongoDB Collection Schema:**
    *   `_id`: `ObjectId` (Primary Key, the internal `job_id`)
    *   `openai_batch_id`: `String` (Indexed)
    *   `input_file_id`: `String`
    *   `output_file_id`: `String` (Nullable)
    *   `error_file_id`: `String` (Nullable)
    *   `status`: `String` (e.g., `submitted`, `processing`, `completed`, `failed`) (Indexed)
    *   `openai_status`: `String` (Nullable, mirrors OpenAI status)
    *   `output_schema_json`: `String`
    *   `mongodb_uri`: `String`
    *   `collection_name`: `String`
    *   `model`: `String`
    *   `created_at`: `Timestamp`
    *   `updated_at`: `Timestamp`

*   **Target MongoDB Document Update Structure:**
    *   An **array** field named `event_response` will be added/updated.
    *   Objects pushed into this array will have the structure:
        ```json
        {
          "event_response": <validated_content_matching_output_schema>,
          "updated": <timestamp_of_processing>
        }
        ```
    *   A status field (e.g., `ai_status`: `String`) will be updated (e.g., to `in_progress`, `completed`, `failed`).

### 7. Configuration

*   `OPENAI_API_KEY`: Provided via environment variable. Read by both API endpoint and polling script.
*   `MONGODB_URI`: Provided per-request via `multipart/form-data`.
*   `collection_name`: Provided per-request via `multipart/form-data`.
*   (Polling Script) `POLL_INTERVAL`: Configurable interval for the scheduler (e.g., every 5 minutes).
*   (Polling Script) `RETRY_MAX_ATTEMPTS`: `3`
*   (Target Documents) `STATUS_FIELD_NAME`: Configurable name for the status field (e.g., `ai_status`).
*   (Target Documents) `STATUS_VALUES`: Configurable values for `in_progress`, `completed`, `failed`.

### 8. Logging

*   **Destination:** Standard Output (stdout).
*   **Format:** Structured JSON.
*   **Mandatory Fields per Log Entry:** `timestamp`, `level` (`INFO`, `WARN`, `ERROR`, `DEBUG`), `event` (string code), `message` (string).
*   **Optional/Contextual Fields:** `job_id`, `openai_batch_id`, `custom_id`, `error_details`, `validation_errors`, `traceback`.
*   **Key Events to Log:** API request received/responded, validation success/failure, file upload, OpenAI batch creation, polling script start/end, job polling status check, result/error file download, result processing start/end, output validation success/failure, document update success/failure, transient errors, retries, persistent errors after retries.

### 9. Error Handling

*   **API Input Validation:** Return `400 Bad Request` with structured JSON details.
*   **Transient Errors (Polling Script):** Retry operations (OpenAI API calls, DB interactions) up to 3 times with exponential backoff. If still failing, log the error and skip processing that specific job for the current run.
*   **OpenAI Batch Item Errors:** Reflected in the error file or error object in the results file. Mark the corresponding target document status as `failed`. Log the error details.
*   **Output Schema Validation Failure:** Mark the corresponding target document status as `failed`. Do not update the `event_response` field. Log the validation error details.
*   **Critical Polling Script Errors:** Log detailed error including traceback. The script should attempt to continue processing other jobs if possible.

### 10. Testing Strategy

*   **Unit Tests:**
    *   Test JSONL line parsing and validation logic.
    *   Test JSON Schema validation utility.
    *   Test Pydantic model generation from JSON Schema string.
    *   Test API request validation logic (mock external services).
    *   Test polling script's job selection query.
    *   Test result processing logic (parsing, validation, generating DB updates) with mock data.
    *   Test retry mechanism logic.
*   **Integration Tests:**
    *   **API:** Send various valid/invalid requests (`multipart/form-data`) to the running API endpoint. Mock OpenAI API calls. Verify responses (`202`, `400`). Verify correct document creation in `batch_jobs` collection. Verify initial status updates in target collection (using a test DB).
    *   **Polling Script:** Set up `batch_jobs` collection with various job states. Mock OpenAI API to return different statuses (`completed`, `failed`, `in_progress`, etc.) and mock result/error file content. Run the polling script. Verify status updates in `batch_jobs`. Verify correct processing of results, including schema validation and updates/status changes in the target collection (test DB). Test error handling and retry logic by mocking transient API failures.
*   **End-to-End Tests (Optional):** If feasible, run the entire flow against a test OpenAI environment and a dedicated test MongoDB instance with a small, controlled batch input. Verify the final state matches expectations.
*   **Performance Tests:** Simulate high load on the API endpoint. Simulate a large number of active jobs for the polling script. Measure response times and processing throughput. Monitor resource usage.

---
