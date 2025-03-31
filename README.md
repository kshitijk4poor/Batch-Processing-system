# Batch Processing System Specification

## Overview
A configurable batch processing system that retrieves MongoDB documents with "pending" status, processes them through OpenAI's batch API, and updates the documents with results according to configurable templates. The system offers flexible document schema configuration through Python code.

## Requirements

### Functional Requirements
1. Query MongoDB for documents with "ai_status" = "pending" (field name configurable)
2. Process these documents in configurable batches using OpenAI's batch API
3. Update document status during processing (status field and values configurable)
4. Append API responses to a configurable response field with configurable structure
5. Update final document status to configurable completion values
6. Implement comprehensive logging of all operations
7. Send notifications for batch completion or failure
8. Support configurable retry attempts with exponential backoff for failures

### Non-Functional Requirements
1. Handle thousands of documents efficiently
2. Implement rate limiting and backoff strategies
3. Provide clear documentation and usage examples
4. Ensure thread safety for concurrent operations

## Architecture

### Core Components
1. **Config Manager**: Handles configuration from Python modules and environment variables
2. **Template Manager**: Manages prompt and response templates
3. **Database Connector**: MongoDB interface with configurable schema mapping
4. **Batch Processor**: Core component that manages OpenAI batch API interactions
5. **Status Manager**: Tracks and updates document statuses
6. **Notification System**: Sends alerts for batch completions and failures
7. **Logger**: Records detailed system operations

### Data Flow
1. Retrieve pending documents from MongoDB using configurable query
2. Apply prompt template to generate user messages
3. Generate JSONL batches for OpenAI processing
4. Submit batches to OpenAI API
5. Poll for batch completion
6. Process results using response template
7. Update documents with responses and status

## Data Schemas

### Schema Configuration (Python Module)
```python
from pydantic import BaseModel, Field
from typing import Optional, Dict, Any, List

class StatusConfig(BaseModel):
    """Define status field and values for tracking document processing."""
    field_name: str = "ai_status"
    pending_value: str = "pending"
    in_progress_value: str = "in_progress"
    completed_value: str = "completed"
    failed_value: str = "failed"

class SchemaConfig(BaseModel):
    """Define MongoDB document schema configuration."""
    status: StatusConfig = Field(default_factory=StatusConfig)
    response_field: str = "event_response"
    response_date_field: str = "created_date"
    response_content_field: str = "talking_points"
    document_id_field: str = "_id"
    processing_info_field: str = "processing_info"
    
    # Query template for finding documents to process
    query_template: Dict[str, Any] = Field(default_factory=lambda: {"ai_status": "pending"})
    
    # Update template for marking documents as in progress
    update_template: Dict[str, Any] = Field(default_factory=lambda: {
        "$set": {"ai_status": "in_progress"}
    })
    
    # Template for adding a new response to the document
    response_template: Dict[str, Any] = Field(default_factory=lambda: {
        "$push": {
            "event_response": {
                "talking_points": "{{response_content}}",
                "created_date": "{{current_date}}"
            }
        },
        "$set": {"ai_status": "completed"}
    })

class DocumentFields(BaseModel):
    """Define fields to extract from documents for prompts."""
    field_map: Dict[str, str] = Field(default_factory=dict)
    
    # Optional preprocessing functions for fields
    preprocess_functions: Dict[str, str] = Field(default_factory=dict)
```

### Configuration Module Structure (config.py)
```python
import os
from pydantic import BaseModel, Field
from typing import Optional, Dict, Any, List
from dotenv import load_dotenv

# Load environment variables
load_dotenv()

class DatabaseConfig(BaseModel):
    """MongoDB database configuration."""
    connection_string: str = Field(default_factory=lambda: os.getenv("MONGODB_URI", ""))
    database_name: str = Field(default_factory=lambda: os.getenv("MONGODB_DB", ""))
    collection_name: str = Field(default_factory=lambda: os.getenv("MONGODB_COLLECTION", ""))

class OpenAIConfig(BaseModel):
    """OpenAI API configuration."""
    api_key: str = Field(default_factory=lambda: os.getenv("OPENAI_API_KEY", ""))
    model: str = Field(default_factory=lambda: os.getenv("OPENAI_MODEL", "gpt-4o-mini"))
    temperature: float = Field(default_factory=lambda: float(os.getenv("OPENAI_TEMP", "0.1")))
    max_tokens: Optional[int] = Field(default_factory=lambda: int(os.getenv("OPENAI_MAX_TOKENS", "500")))
    response_format: Optional[dict] = None

class BatchConfig(BaseModel):
    """Batch processing configuration."""
    batch_size: int = Field(default_factory=lambda: int(os.getenv("BATCH_SIZE", "100")))
    completion_window: str = Field(default_factory=lambda: os.getenv("COMPLETION_WINDOW", "24h"))
    poll_interval: int = Field(default_factory=lambda: int(os.getenv("POLL_INTERVAL", "300")))
    endpoint: str = Field(default="/v1/chat/completions")
    max_retries: int = Field(default_factory=lambda: int(os.getenv("MAX_RETRIES", "3")))

class PromptConfig(BaseModel):
    """Prompt configuration."""
    system_prompt: str = Field(default_factory=lambda: os.getenv("SYSTEM_PROMPT", ""))
    user_prompt_template: str = Field(default_factory=lambda: os.getenv("USER_PROMPT_TEMPLATE", ""))

class NotificationConfig(BaseModel):
    """Notification configuration."""
    webhook_url: Optional[str] = Field(default_factory=lambda: os.getenv("NOTIFICATION_WEBHOOK", ""))
    email: Optional[str] = Field(default_factory=lambda: os.getenv("NOTIFICATION_EMAIL", ""))

# Import schema configuration (example)
from schema_config import SchemaConfig, DocumentFields

class AppConfig(BaseModel):
    """Main application configuration."""
    database: DatabaseConfig = Field(default_factory=DatabaseConfig)
    openai: OpenAIConfig = Field(default_factory=OpenAIConfig)
    batch: BatchConfig = Field(default_factory=BatchConfig)
    prompt: PromptConfig = Field(default_factory=PromptConfig)
    notification: NotificationConfig = Field(default_factory=NotificationConfig)
    schema: SchemaConfig = Field(default_factory=SchemaConfig)
    document_fields: DocumentFields = Field(default_factory=DocumentFields)
    
    @classmethod
    def from_env(cls) -> "AppConfig":
        """Load configuration from environment variables."""
        return cls()
```

## Schema Configuration Example
```python
# schema_config.py - Example for medical documents
from pydantic import BaseModel, Field
from typing import Dict, Any

class StatusConfig(BaseModel):
    field_name: str = "ai_status"
    pending_value: str = "pending"
    in_progress_value: str = "in_progress"
    completed_value: str = "completed"
    failed_value: str = "failed"

class SchemaConfig(BaseModel):
    status: StatusConfig = Field(default_factory=StatusConfig)
    response_field: str = "event_response"
    response_date_field: str = "created_date"
    response_content_field: str = "talking_points"
    document_id_field: str = "document_id"
    
    # Query to find documents to process
    query_template: Dict[str, Any] = Field(default_factory=lambda: {
        "ai_status": "pending"
    })
    
    # Update to mark as in progress
    update_template: Dict[str, Any] = Field(default_factory=lambda: {
        "$set": {"ai_status": "in_progress"}
    })
    
    # Template for adding a response
    response_template: Dict[str, Any] = Field(default_factory=lambda: {
        "$push": {
            "event_response": {
                "talking_points": "{{response_content}}",
                "created_date": "{{current_date}}"
            }
        },
        "$set": {"ai_status": "completed"}
    })

class DocumentFields(BaseModel):
    # Map document fields to template variables
    field_map: Dict[str, str] = Field(default_factory=lambda: {
        "patient_name": "patient_details.first_name",
        "patient_age": "patient_details.patient_age",
        "doctor_name": "medical_details.doctor_name",
        "document_id": "document_id"
    })
```

## JSONL Batch Format
```json
{
  "custom_id": "doc_id_123",
  "method": "POST",
  "url": "/v1/chat/completions",
  "body": {
    "model": "configured_model",
    "temperature": 0.1,
    "max_tokens": 500,
    "messages": [
      {
        "role": "system",
        "content": "configured_system_prompt"
      },
      {
        "role": "user",
        "content": "templated_content_from_document"
      }
    ]
  }
}
```

## Template System

### Prompt Template
Uses a Jinja2-like syntax to insert document fields into prompts:
```
Your patient {{patient_name}} has visited on {{last_visit_date}}. 
Their medical details show {{doctor_name}} has recommended treatment.
```

### Response Processing
Python functions to transform API responses before database updates:
```python
def process_response(response, document):
    """Transform API response to match required document structure."""
    # Extract content from OpenAI response
    content = response["choices"][0]["message"]["content"]
    
    # Parse content as needed (e.g., from JSON)
    import json
    try:
        content = json.loads(content)
    except:
        pass
    
    # Create the response object based on schema configuration
    return {
        "_id": document["document_id"],
        "patient_type": "ip",  # Default or from document
        "talking_points": content
    }
```

## Error Handling Strategy
1. Implement structured error types:
   - `ConnectionError`: Database and API connection issues
   - `ValidationError`: Data validation failures
   - `ProcessingError`: Batch processing failures
   - `TimeoutError`: Polling timeout issues
   - `TemplateError`: Template rendering or parsing errors

2. Error recovery:
   - Documents with processing errors return to "pending" state (if under retry limit)
   - Documents exceeding retry limit marked as "failed"
   - Implement exponential backoff for rate limit errors

3. Document error tracking:
   - Store error details in configurable error field
   - Include timestamp and attempt count

## Logging
1. Implement structured logging with:
   - Timestamp
   - Operation type (query, process, update)
   - Batch ID
   - Document count
   - Status
   - Error details (when applicable)

2. Log levels:
   - INFO: General operation status
   - ERROR: Operation failures
   - DEBUG: Detailed processing information

## Notification System
1. Send webhook notifications for:
   - Batch completion
   - Batch failure
   - Retry attempts

2. Notification payload:
   - Batch ID
   - Status
   - Document count (total/succeeded/failed)
   - Timestamp
   - Error summary (if applicable)

## Implementation Plan

### Phase 1: Core Infrastructure
1. Set up configuration management
2. Implement MongoDB connector
3. Create template system for prompts and responses

### Phase 2: Batch Processing
1. Implement OpenAI batch API client
2. Create status polling and management
3. Develop result processing and storage

### Phase 3: Resilience & Monitoring
1. Add retry mechanisms
2. Implement logging
3. Add notification system

### Phase 4: Testing & Documentation
1. Create unit and integration tests
2. Prepare documentation
3. Develop usage examples

## Testing Strategy

### Unit Tests
1. Test configuration loading
2. Test MongoDB operations
3. Test template rendering and parsing
4. Test status updates
5. Test error handling

### Integration Tests
1. Test end-to-end flow with mock OpenAI API
2. Test error recovery
3. Test template application with real documents

### Performance Tests
1. Test with varying batch sizes
2. Measure processing throughput
3. Test with simulated API delays

## Usage Example
```python
from batch_processor import BatchProcessor
from my_custom_schema import medical_schema_config

# Initialize with default configuration and custom schema
processor = BatchProcessor(schema_config=medical_schema_config)

# Process pending documents
result = processor.process_pending_documents()

# Check processing stats
print(f"Processed: {result.total_processed}")
print(f"Succeeded: {result.succeeded}")
print(f"Failed: {result.failed}")

# Example with custom response processor
def custom_response_processor(response, document):
    """Custom function to process responses."""
    content = response["choices"][0]["message"]["content"]
    return {
        "_id": document["document_id"],
        "patient_type": document.get("patient_details", {}).get("patient_type", "unknown"),
        "talking_points": {
            "Benefits": content.get("benefits", []),
            "Condition": content.get("condition", ""),
            "IfDelayed": content.get("if_delayed", []),
            "Recommendation": content.get("recommendation", "")
        }
    }

processor.set_response_processor(custom_response_processor)
processor.process_pending_documents()
```

## File Structure
```
batch_processing/
├── config.py              # Configuration management
├── schema_config.py       # Default schema configuration
├── db.py                  # Database connector
├── processor.py           # Main batch processor
├── templates.py           # Template system
├── models.py              # Data models
├── notifier.py            # Notification system
├── logger.py              # Logging utilities
├── errors.py              # Error handling
├── utils.py               # Utility functions
├── tests/                 # Test suite
│   ├── test_processor.py
│   ├── test_templates.py
│   └── test_db.py
├── examples/              # Usage examples
│   ├── basic_processing.py
│   ├── custom_schema.py
│   └── custom_processor.py
├── schema_examples/       # Example schema configurations
│   ├── medical_schema.py
│   └── customer_service_schema.py
└── README.md              # Documentation
```

## Implementation Timeline

### Week 1: Core Components
- Day 1-2: Configuration system and schema setup
- Day 3-4: MongoDB connector implementation
- Day 5: Template system implementation

### Week 2: Processing Pipeline
- Day 1-2: Batch creation and JSONL generation
- Day 3-4: OpenAI API integration and polling
- Day 5: Response processing and document updates

### Week 3: Resilience & Quality
- Day 1-2: Error handling and retries
- Day 3: Logging system
- Day 4: Notification system
- Day 5: Unit tests

### Week 4: Testing & Documentation
- Day 1-2: Integration tests
- Day 3: Performance tests
- Day 4: Documentation
- Day 5: Example implementations and usage guides

## Dependencies
- Python 3.8+
- OpenAI Python SDK 1.6.0+
- PyMongo 4.3.0+
- Pydantic 2.0.0+
- Jinja2 (or similar templating)
- Requests
- python-dotenv

## Deployment Considerations
1. Run as a scheduled job for periodic processing
2. Implement locking to prevent duplicate processing
3. Consider containerization for deployment
4. Set up monitoring for system health
5. Store templates in separate files for easy management

This comprehensive specification provides a complete roadmap for implementing the batch processing system with configurable document schemas, prompts, and response templates. A developer can use this specification to implement a robust, configurable, and maintainable system that meets all the requirements discussed. 