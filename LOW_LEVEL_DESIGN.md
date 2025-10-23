# Async Image Classification Service - Low Level Design

## 1. Introduction

### 1.1 Purpose
This Low Level Design (LLD) document provides detailed implementation specifications for the Async Image Classification Service, including class diagrams, database schemas, API specifications, and implementation details.

### 1.2 Scope
- Detailed class and module designs
- Database schema and data models
- API endpoint specifications
- Error handling and validation logic
- Configuration management
- Testing strategies

## 2. Module Design

### 2.1 Project Structure
```
async-image-classifier/
├── app/
│   ├── __init__.py
│   ├── main.py                 # FastAPI application entry point
│   ├── config.py               # Configuration management
│   ├── celery_app.py          # Celery application setup
│   ├── api/
│   │   ├── __init__.py
│   │   ├── dependencies.py     # Dependency injection
│   │   ├── endpoints/
│   │   │   ├── __init__.py
│   │   │   ├── classification.py
│   │   │   ├── status.py
│   │   │   ├── health.py
│   │   │   └── admin.py
│   │   └── middleware/
│   │       ├── __init__.py
│   │       ├── auth.py
│   │       ├── cors.py
│   │       ├── rate_limiting.py
│   │       └── error_handling.py
│   ├── core/
│   │   ├── __init__.py
│   │   ├── exceptions.py       # Custom exceptions
│   │   ├── security.py         # Authentication utilities
│   │   ├── redis_client.py     # Redis connection management
│   │   └── logging.py          # Logging configuration
│   ├── models/
│   │   ├── __init__.py
│   │   ├── requests.py         # Pydantic request models
│   │   ├── responses.py        # Pydantic response models
│   │   ├── enums.py            # Enumerations and constants
│   │   └── database.py         # Database models (if needed)
│   ├── services/
│   │   ├── __init__.py
│   │   ├── image_service.py    # Image processing utilities
│   │   ├── job_service.py      # Job management service
│   │   └── model_service.py    # ML model management
│   ├── tasks/
│   │   ├── __init__.py
│   │   ├── classification.py   # Celery classification tasks
│   │   └── cleanup.py          # Cleanup and maintenance tasks
│   └── utils/
│       ├── __init__.py
│       ├── validators.py       # Input validation utilities
│       ├── helpers.py          # General utility functions
│       └── constants.py        # Application constants
├── tests/
│   ├── __init__.py
│   ├── conftest.py            # Pytest configuration
│   ├── test_api/
│   ├── test_tasks/
│   ├── test_services/
│   └── test_utils/
├── docker/
│   ├── Dockerfile.api
│   ├── Dockerfile.worker
│   └── docker-compose.yml
├── scripts/
│   ├── start-api.sh
│   ├── start-worker.sh
│   └── setup-dev.sh
├── requirements/
│   ├── base.txt
│   ├── dev.txt
│   └── prod.txt
├── .env.example
├── .gitignore
├── pyproject.toml
└── README.md
```

## 3. Data Models and Schemas

### 3.1 Pydantic Models

#### 3.1.1 Request Models
```pseudocode
// Request model for image classification
ClassificationRequest:
    priority: optional Priority enum (default: NORMAL)
    model_version: optional ModelVersion enum (default: LATEST)  
    callback_url: optional URL string (must match URL pattern)
    metadata: optional key-value dictionary (default: empty)
    
    validation_rules:
        callback_url must be valid https URL if provided
        metadata cannot exceed 1KB in size
    
    example:
        priority: "normal"
        model_version: "v1.0"
        callback_url: "https://example.com/webhook"
        metadata: {user_id: "12345", session_id: "abc"}

// Image upload validation schema
ImageUpload:
    file: uploaded_file (required)
    
    validation_function validate_image_file(file):
        allowed_types = [image/jpeg, image/png, image/webp]
        max_size = 10_MB
        
        if file.content_type not in allowed_types:
            raise ValidationError("Unsupported file type")
        if file.size > max_size:
            raise ValidationError("File size exceeds limit")
        return file

// Job status query parameters
JobStatusRequest:
    include_progress: boolean (default: true)
    include_metadata: boolean (default: false)
```

#### 3.1.2 Response Models
```pseudocode
// Individual prediction result schema
Prediction:
    class_name: string (required, description: "Predicted class name")
    confidence: float (required, range: 0.0 to 1.0, description: "Confidence score")
    class_id: integer (required, description: "Class identifier")
    
// Bounding box coordinates schema
BoundingBox:
    x: float (required, minimum: 0.0)
    y: float (required, minimum: 0.0)  
    width: float (required, greater than: 0.0)
    height: float (required, greater than: 0.0)

// Response for classification request
ClassificationResponse:
    job_id: string (required, description: "Unique job identifier")
    status: JobStatus enum (required, description: "Current job status")
    created_at: timestamp (required, description: "Job creation timestamp")
    estimated_completion: optional timestamp (description: "Estimated completion time")
    
    example:
        job_id: "550e8400-e29b-41d4-a716-446655440000"
        status: "pending"
        created_at: "2025-10-23T10:30:00Z"
        estimated_completion: "2025-10-23T10:31:00Z"

// Response for job status query
JobStatusResponse:
    job_id: string
    status: JobStatus enum
    progress: integer (range: 0 to 100)
    stage: optional string
    updated_at: timestamp  
    error_message: optional string
    metadata: optional key-value dictionary

// Final classification results
ClassificationResult:
    job_id: string
    predictions: array of Prediction objects
    processing_time: float (description: "Processing time in seconds")
    model_version: string
    image_metadata: key-value dictionary
    created_at: timestamp
    
// Health check response
HealthResponse:
    status: string (description: "Overall health status")
    timestamp: timestamp
    services: key-value dictionary (description: "Individual service statuses")
    version: string
    uptime: float

// Error response schema
ErrorResponse:
    error: string
    detail: string  
    timestamp: timestamp
    request_id: optional string
```

#### 3.1.3 Enumerations
```pseudocode
// Job status enumeration
JobStatus enumeration:
    PENDING = "pending"
    PROCESSING = "processing" 
    COMPLETED = "completed"
    FAILED = "failed"
    CANCELLED = "cancelled"

// Priority levels enumeration
Priority enumeration:
    LOW = "low"
    NORMAL = "normal"
    HIGH = "high"
    URGENT = "urgent"

// Model version enumeration
ModelVersion enumeration:
    LATEST = "latest"
    V1_0 = "v1.0"
    V1_1 = "v1.1"
    EXPERIMENTAL = "experimental"

// Processing stage enumeration
ProcessingStage enumeration:
    INITIALIZING = "initializing"
    PREPROCESSING = "preprocessing"
    INFERENCE = "inference" 
    POSTPROCESSING = "postprocessing"
    COMPLETED = "completed"

// Error code enumeration
ErrorCode enumeration:
    INVALID_IMAGE = "INVALID_IMAGE"
    MODEL_ERROR = "MODEL_ERROR"
    TIMEOUT = "TIMEOUT"
    INTERNAL_ERROR = "INTERNAL_ERROR"
    RATE_LIMITED = "RATE_LIMITED"
    UNAUTHORIZED = "UNAUTHORIZED"
```

### 3.2 Redis Data Schema

#### 3.2.1 Key Patterns and TTL
```pseudocode
// Redis key patterns and configurations
RedisKeys configuration:
    // Job-related key patterns
    JOB_STATUS = "job:{job_id}:status"           // TTL: 3600 seconds
    JOB_RESULT = "job:{job_id}:result"           // TTL: 86400 seconds  
    JOB_IMAGE = "job:{job_id}:image"             // TTL: 1800 seconds
    JOB_METADATA = "job:{job_id}:metadata"       // TTL: 3600 seconds
    
    // Rate limiting key patterns
    RATE_LIMIT_IP = "rate_limit:ip:{ip}:{window}"           // TTL: 60 seconds
    RATE_LIMIT_USER = "rate_limit:user:{user_id}:{window}"  // TTL: 60 seconds
    
    // Session key patterns
    SESSION = "session:{session_id}"              // TTL: 1800 seconds
    
    // System key patterns
    HEALTH_CHECK = "health:check"                 // TTL: 30 seconds
    WORKER_STATUS = "worker:{worker_id}:status"   // TTL: 60 seconds
    
    // Celery key patterns (managed by Celery)
    CELERY_QUEUE = "celery"
    CELERY_RESULT = "celery-task-meta-{task_id}"

// Job status data structure definition
JobStatusData structure:
    properties:
        job_id: string
        status: string
        progress: integer (default: 0)
        stage: optional string
        error: optional string  
        metadata: key-value dictionary (default: empty)
        updated_at: timestamp (auto-generated)
    
    function to_dict():
        return all_properties_as_dictionary
    
    function from_dict(data_dictionary):
        return create_instance_from_dictionary(data_dictionary)
```

## 4. API Endpoint Specifications

### 4.1 Classification Endpoints

#### 4.1.1 POST /classify
```pseudocode
// Classification endpoint specification
endpoint POST /api/v1/classify:
    
    input_parameters:
        file: uploaded_image_file (required)
        priority: string (optional, default: "normal")
        model_version: string (optional, default: "latest")
        callback_url: string (optional)
        
    authentication: required (Bearer token)
    
    processing_logic:
        // Generate unique job identifier
        job_id = generate_uuid()
        
        // Validate image file
        call image_service.validate_image(file)
        
        // Read and store image data temporarily
        image_data = read_file_content(file)
        call image_service.store_temporary_image(job_id, image_data)
        
        // Create classification job record
        job = call job_service.create_job(
            job_id, priority, model_version, callback_url, image_metadata
        )
        
        // Submit task to Celery queue
        task = submit_async_task(
            task_name: "classify_image_task",
            arguments: [job_id, model_version],
            queue: "classification_" + priority
        )
        
        // Update job with task reference
        call job_service.update_job_task_id(job_id, task.id)
        
        return ClassificationResponse(
            job_id, status="pending", created_at, estimated_completion
        )
        
    error_handling:
        ImageValidationError -> HTTP 400 "Bad Request"
        ServiceUnavailableError -> HTTP 503 "Service Unavailable" 
        Generic Exception -> HTTP 500 "Internal Server Error"

// Batch classification endpoint
endpoint POST /api/v1/classify/batch:
    
    input_parameters:
        files: array_of_uploaded_files (required, max: 10)
        priority: string (optional)
        model_version: string (optional)
        
    processing_logic:
        validate batch_size <= maximum_allowed
        results = empty_array
        
        for each file in files:
            result = process_single_classification(file, priority, model_version)
            add result to results
            
        return results
```

#### 4.1.2 GET /status/{job_id}
```pseudocode
// Job status endpoint specification
endpoint GET /api/v1/status/{job_id}:
    
    path_parameters:
        job_id: string (required, job identifier)
        
    query_parameters:
        include_metadata: boolean (optional, default: false)
        
    authentication: required (Bearer token)
    
    processing_logic:
        // Retrieve job status from service
        job_status = call job_service.get_job_status(job_id)
        
        if job_status is null:
            return HTTP_404_ERROR("Job not found")
        
        // Prepare response data
        response_data = {
            job_id: job_id,
            status: job_status.status,
            progress: job_status.progress,
            stage: job_status.stage,
            updated_at: job_status.updated_at,
            error_message: job_status.error
        }
        
        if include_metadata is true:
            response_data.metadata = job_status.metadata
            
        return JobStatusResponse(response_data)
        
    error_handling:
        Job not found -> HTTP 404 "Job not found"
        Generic Exception -> HTTP 500 "Internal server error"

// WebSocket endpoint for real-time status updates
websocket_endpoint /api/v1/status/{job_id}/ws:
    
    path_parameters:
        job_id: string (required)
        
    processing_logic:
        establish_websocket_connection()
        
        while connection_is_active:
            status = call job_service.get_job_status(job_id)
            
            if status exists:
                send_json_message(status.to_dict())
                
                // Close connection if job is finished
                if status.status in [COMPLETED, FAILED]:
                    break
                    
            wait_for_1_second()
            
        close_websocket_connection()
        
    error_handling:
        WebSocket disconnect -> log_and_cleanup()
        Generic Exception -> close_connection_with_error_code()
```

#### 4.1.3 GET /result/{job_id}
```pseudocode
// Classification result endpoint specification
endpoint GET /api/v1/result/{job_id}:
    
    path_parameters:
        job_id: string (required, job identifier)
        
    authentication: required (Bearer token)
    
    processing_logic:
        // Check job status first
        job_status = call job_service.get_job_status(job_id)
        
        if job_status is null:
            return HTTP_404_ERROR("Job not found")
        
        if job_status.status equals FAILED:
            return HTTP_409_ERROR("Job failed: " + job_status.error)
        
        if job_status.status not equals COMPLETED:
            return HTTP_409_ERROR("Job not completed. Current status: " + job_status.status)
        
        // Retrieve results
        result = call job_service.get_job_result(job_id)
        
        if result is null:
            return HTTP_404_ERROR("Results not found")
        
        return result
        
    error_handling:
        Job not found -> HTTP 404 "Job not found"
        Job failed -> HTTP 409 "Job failed with error message"
        Job not completed -> HTTP 409 "Job not completed"
        Results not found -> HTTP 404 "Results not found"
        Generic Exception -> HTTP 500 "Internal server error"
```

### 4.2 Health and Admin Endpoints

#### 4.2.1 GET /health
```pseudocode
// Health check endpoint specification
endpoint GET /api/v1/health:
    
    dependencies:
        redis_client: Redis connection
        celery_app: Celery application instance
    
    processing_logic:
        services_status = empty_dictionary
        overall_status = "healthy"
        
        // Check Redis connectivity
        try:
            call redis_client.ping()
            services_status["redis"] = "healthy"
        catch Exception as redis_error:
            services_status["redis"] = "unhealthy: " + redis_error.message
            overall_status = "degraded"
        
        // Check Celery workers
        try:
            active_workers = call celery_app.control.inspect_active_workers()
            if active_workers has_workers:
                services_status["celery_workers"] = "healthy (" + worker_count + " workers)"
            else:
                services_status["celery_workers"] = "no active workers"
                overall_status = "degraded"
        catch Exception as celery_error:
            services_status["celery_workers"] = "unhealthy: " + celery_error.message
            overall_status = "unhealthy"
        
        // Check model availability
        try:
            model_status = call check_model_health()
            services_status["models"] = model_status
        catch Exception as model_error:
            services_status["models"] = "unhealthy: " + model_error.message
            overall_status = "degraded"
        
        return HealthResponse(
            status: overall_status,
            timestamp: current_timestamp(),
            services: services_status,
            version: application_version,
            uptime: calculate_uptime()
        )
```

## 5. Service Layer Implementation

### 5.1 Job Service
```pseudocode
// Service for managing classification jobs
class JobService:
    
    constructor(redis_client):
        this.redis = redis_client
        
    function create_job(job_id, priority, model_version, callback_url, image_metadata):
        job_data = {
            job_id: job_id,
            status: JobStatus.PENDING,
            progress: 0,
            priority: priority,
            model_version: model_version,
            callback_url: callback_url,
            image_metadata: image_metadata or empty_dict,
            created_at: current_timestamp(),
            updated_at: current_timestamp()
        }
        
        // Calculate estimated completion time
        estimated_time = calculate_estimated_completion(priority)
        job_data.estimated_completion = estimated_time
        
        // Store job data in Redis with TTL
        job_key = "job:" + job_id + ":status"
        call redis.hash_set(job_key, job_data)
        call redis.set_expiry(job_key, 3600)  // 1 hour TTL
        
        return JobStatusResponse(job_data)
    
    function get_job_status(job_id):
        job_key = "job:" + job_id + ":status"
        job_data = call redis.hash_get_all(job_key)
        
        if job_data is empty:
            return null
        
        return JobStatusResponse(job_data)
    
    function update_job_status(job_id, status, progress, stage, error):
        job_key = "job:" + job_id + ":status"
        
        update_data = {
            status: status,
            updated_at: current_timestamp()
        }
        
        if progress is not null:
            update_data.progress = progress
        if stage is not null:
            update_data.stage = stage
        if error is not null:
            update_data.error = error
            
        call redis.hash_set(job_key, update_data)
    
    function store_job_result(job_id, result):
        result_key = "job:" + job_id + ":result"
        result_data = serialize_to_json(result)
        
        call redis.set(result_key, result_data, ttl=86400)  // 24 hour TTL
        
        // Update job status to completed
        call update_job_status(job_id, JobStatus.COMPLETED, progress=100)
    
    function get_job_result(job_id):
        result_key = "job:" + job_id + ":result"
        result_data = call redis.get(result_key)
        
        if result_data is null:
            return null
        
        return deserialize_from_json(result_data, ClassificationResult)
    
    private function calculate_estimated_completion(priority):
        base_time = 30  // 30 seconds base processing time
        
        priority_multipliers = {
            Priority.LOW: 2.0,
            Priority.NORMAL: 1.0,
            Priority.HIGH: 0.5,
            Priority.URGENT: 0.25
        }
        
        multiplier = priority_multipliers[priority] or 1.0
        estimated_seconds = base_time * multiplier
        
        return current_timestamp() + estimated_seconds
```

### 5.2 Image Service
```pseudocode
// Service for image processing and validation
class ImageService:
    
    constants:
        ALLOWED_FORMATS = [JPEG, PNG, WEBP]
        MAX_FILE_SIZE = 10_MB
        MAX_DIMENSIONS = (4096, 4096)
        MIN_DIMENSIONS = (32, 32)
    
    constructor(redis_client):
        this.redis = redis_client
    
    function validate_image(uploaded_file):
        // Check file size
        if uploaded_file.size > MAX_FILE_SIZE:
            throw ImageValidationError("File size exceeds limit")
        
        // Read file content
        content = read_file_content(uploaded_file)
        
        if content.length > MAX_FILE_SIZE:
            throw ImageValidationError("File size exceeds limit")
        
        // Validate image format and dimensions
        try:
            image = open_image_from_bytes(content)
            
            // Check format
            if image.format not in ALLOWED_FORMATS:
                throw ImageValidationError("Unsupported format: " + image.format)
            
            // Check dimensions
            width, height = image.size
            if width < MIN_DIMENSIONS.width or height < MIN_DIMENSIONS.height:
                throw ImageValidationError("Image too small: " + width + "x" + height)
            
            if width > MAX_DIMENSIONS.width or height > MAX_DIMENSIONS.height:
                throw ImageValidationError("Image too large: " + width + "x" + height)
            
            return {
                format: image.format,
                mode: image.mode,
                size: image.size,
                file_size: content.length
            }
                
        catch Exception as validation_error:
            throw ImageValidationError("Invalid image file: " + validation_error.message)
    
    function store_temporary_image(job_id, image_data):
        image_key = "job:" + job_id + ":image"
        
        // Encode image data as base64 for Redis storage
        encoded_data = encode_base64(image_data)
        
        // Store with 30 minute TTL
        call redis.set(image_key, encoded_data, ttl=1800)
    
    function get_temporary_image(job_id):
        image_key = "job:" + job_id + ":image"
        encoded_data = call redis.get(image_key)
        
        if encoded_data is null:
            throw ImageValidationError("Image data not found for job " + job_id)
        
        return decode_base64(encoded_data)
    
    function preprocess_image(image_data, target_size=(224, 224)):
        image = open_image_from_bytes(image_data)
        
        // Convert to RGB if necessary
        if image.mode != RGB:
            image = convert_to_rgb(image)
        
        // Resize image using high-quality resampling
        image = resize_image(image, target_size, LANCZOS_RESAMPLING)
        
        // Save processed image as JPEG
        output_buffer = create_byte_buffer()
        save_image_to_buffer(image, output_buffer, format=JPEG, quality=95)
        
        return output_buffer.get_bytes()
```

## 6. Celery Task Implementation

### 6.1 Classification Task
```pseudocode
// Main image classification Celery task
function classify_image_task(job_id, model_version="latest"):
    start_time = current_time()
    
    // Initialize services
    image_service = create_image_service(redis_client)
    job_service = create_job_service(redis_client)
    model_service = create_model_service()
    
    try:
        // Update job status to processing
        call job_service.update_job_status(
            job_id, JobStatus.PROCESSING, progress=0, stage=INITIALIZING
        )
        
        // Step 1: Load image data (Progress: 0-20%)
        update_celery_task_state(progress=10, stage=INITIALIZING)
        image_data = call image_service.get_temporary_image(job_id)
        
        // Step 2: Preprocess image (Progress: 20-40%)
        update_celery_task_state(progress=25, stage=PREPROCESSING)
        processed_image = call image_service.preprocess_image(image_data)
        image_tensor = convert_image_to_tensor(processed_image)
        
        call job_service.update_job_status(
            job_id, JobStatus.PROCESSING, progress=40, stage=PREPROCESSING
        )
        
        // Step 3: Load model (Progress: 40-50%)
        update_celery_task_state(progress=45, stage="loading_model")
        model = call model_service.get_model(model_version)
        
        // Step 4: Run inference (Progress: 50-80%)
        update_celery_task_state(progress=60, stage=INFERENCE)
        disable_gradients:
            outputs = call model.forward(image_tensor.unsqueeze(0))
            predictions = process_model_outputs(outputs, model_version)
        
        call job_service.update_job_status(
            job_id, JobStatus.PROCESSING, progress=80, stage=INFERENCE
        )
        
        // Step 5: Post-process results (Progress: 80-95%)
        update_celery_task_state(progress=85, stage=POSTPROCESSING)
        formatted_results = format_predictions(predictions)
        processing_time = current_time() - start_time
        
        // Step 6: Store results (Progress: 95-100%)
        update_celery_task_state(progress=95, stage="storing_results")
        
        result = create_classification_result(
            job_id, formatted_results, processing_time, 
            model_version, get_image_metadata(image_data)
        )
        
        call job_service.store_job_result(job_id, result)
        
        // Complete
        update_celery_task_state(progress=100, stage=COMPLETED)
        
        // Cleanup temporary image
        cleanup_temporary_data(job_id)
        
        return result.to_dict()
        
    catch Exception as error:
        // Handle task failure
        error_message = error.message
        
        call job_service.update_job_status(
            job_id, JobStatus.FAILED, error=error_message
        )
        
        // Retry logic with exponential backoff
        if current_retry_count < max_retries:
            retry_countdown = 2 ^ current_retry_count
            schedule_retry(countdown=retry_countdown, exception=error)
        
        throw error

// Helper functions
function convert_image_to_tensor(image_data):
    image = open_image_from_bytes(image_data)
    
    // Apply normalization transforms
    tensor = convert_to_tensor(image)
    normalized_tensor = normalize(tensor, 
        mean=[0.485, 0.456, 0.406], 
        std=[0.229, 0.224, 0.225]
    )
    
    return normalized_tensor

function process_model_outputs(outputs, model_version):
    probabilities = apply_softmax(outputs, dimension=1)
    top_predictions = get_top_k(probabilities, k=5)
    
    class_names = get_class_names(model_version)
    
    predictions = empty_array
    for i, (probability, class_index) in enumerate(top_predictions):
        prediction = {
            class_id: class_index,
            class_name: class_names[class_index],
            confidence: probability,
            rank: i + 1
        }
        add prediction to predictions
    
    return predictions

function format_predictions(predictions):
    formatted_results = empty_array
    for prediction in predictions:
        formatted_prediction = create_prediction_object(
            class_name=prediction.class_name,
            confidence=prediction.confidence,
            class_id=prediction.class_id
        )
        add formatted_prediction to formatted_results
    
    return formatted_results

function get_image_metadata(image_data):
    image = open_image_from_bytes(image_data)
    return {
        format: image.format,
        mode: image.mode,
        size: image.size,
        file_size: image_data.length
    }

function cleanup_temporary_data(job_id):
    image_key = "job:" + job_id + ":image"
    call redis_client.delete(image_key)

function get_class_names(model_version):
    // Load class names from configuration file
    return [
        "cat", "dog", "bird", "car", "bicycle", "airplane", "boat", "traffic_light",
        "fire_hydrant", "stop_sign", "parking_meter", "bench", "elephant", "bear",
        "zebra", "giraffe", "backpack", "umbrella", "handbag", "tie"
        // ... more classes based on model
    ]
```

## 7. Configuration Management

### 7.1 Settings
```pseudocode
// Application configuration structure
class Settings:
    
    // Application settings
    APP_NAME: string (default: "Async Image Classifier")
    VERSION: string (default: "1.0.0")
    DEBUG: boolean (environment: "DEBUG", default: false)
    
    // API settings
    API_HOST: string (environment: "API_HOST", default: "0.0.0.0")
    API_PORT: integer (environment: "API_PORT", default: 8000)
    API_PREFIX: string (default: "/api/v1")
    
    // Redis settings
    REDIS_URL: string (environment: "REDIS_URL", default: "redis://localhost:6379")
    REDIS_PASSWORD: optional string (environment: "REDIS_PASSWORD")
    REDIS_DB: integer (environment: "REDIS_DB", default: 0)
    REDIS_MAX_CONNECTIONS: integer (environment: "REDIS_MAX_CONNECTIONS", default: 20)
    
    // Celery settings
    CELERY_BROKER_URL: string (environment: "CELERY_BROKER_URL", default: "redis://localhost:6379/0")
    CELERY_RESULT_BACKEND: string (environment: "CELERY_RESULT_BACKEND", default: "redis://localhost:6379/0")
    CELERY_TASK_SERIALIZER: string (default: "json")
    CELERY_RESULT_SERIALIZER: string (default: "json")
    CELERY_ACCEPT_CONTENT: array (default: ["json"])
    CELERY_TIMEZONE: string (default: "UTC")
    CELERY_ENABLE_UTC: boolean (default: true)
    
    // Model settings
    MODEL_PATH: string (environment: "MODEL_PATH", default: "./models")
    DEFAULT_MODEL_VERSION: string (environment: "DEFAULT_MODEL_VERSION", default: "v1.0")
    MODEL_CACHE_SIZE: integer (environment: "MODEL_CACHE_SIZE", default: 3)
    
    // File upload settings
    MAX_FILE_SIZE: integer (environment: "MAX_FILE_SIZE", default: 10_MB)
    ALLOWED_EXTENSIONS: array (default: ["jpg", "jpeg", "png", "webp"])
    UPLOAD_TEMP_DIR: string (environment: "UPLOAD_TEMP_DIR", default: "/tmp/uploads")
    
    // Security settings
    SECRET_KEY: string (environment: "SECRET_KEY", required: true)
    ACCESS_TOKEN_EXPIRE_MINUTES: integer (environment: "ACCESS_TOKEN_EXPIRE_MINUTES", default: 30)
    API_KEY_HEADER: string (default: "X-API-Key")
    ALLOWED_HOSTS: array (environment: "ALLOWED_HOSTS", default: ["*"])
    
    // Rate limiting settings
    RATE_LIMIT_PER_MINUTE: integer (environment: "RATE_LIMIT_PER_MINUTE", default: 60)
    RATE_LIMIT_BURST: integer (environment: "RATE_LIMIT_BURST", default: 10)
    
    // Logging settings
    LOG_LEVEL: string (environment: "LOG_LEVEL", default: "INFO")
    LOG_FORMAT: string (default: "%(asctime)s - %(name)s - %(levelname)s - %(message)s")
    
    // Monitoring settings
    METRICS_ENABLED: boolean (environment: "METRICS_ENABLED", default: true)
    HEALTH_CHECK_INTERVAL: integer (environment: "HEALTH_CHECK_INTERVAL", default: 30)
    
    configuration:
        environment_file: ".env"
        case_sensitive: true

// Global settings instance
settings = create_settings_instance()
```

## 8. Error Handling and Validation

### 8.1 Custom Exceptions
```pseudocode
// Base exception for API errors
class BaseAPIException extends Exception:
    constructor(message, status_code=500, error_code=null):
        this.message = message
        this.status_code = status_code
        this.error_code = error_code
        call super(message)

// Image validation exception
class ImageValidationError extends BaseAPIException:
    constructor(message):
        call super(message, status_code=400, error_code="INVALID_IMAGE")

// Job not found exception
class JobNotFoundError extends BaseAPIException:
    constructor(job_id):
        message = "Job " + job_id + " not found"
        call super(message, status_code=404, error_code="JOB_NOT_FOUND")

// Model operation exception
class ModelError extends BaseAPIException:
    constructor(message):
        call super(message, status_code=500, error_code="MODEL_ERROR")

// Rate limiting exception
class RateLimitExceededError extends BaseAPIException:
    constructor(message="Rate limit exceeded"):
        call super(message, status_code=429, error_code="RATE_LIMITED")

// Service unavailable exception
class ServiceUnavailableError extends BaseAPIException:
    constructor(message="Service temporarily unavailable"):
        call super(message, status_code=503, error_code="SERVICE_UNAVAILABLE")
```

### 8.2 Error Handler Middleware
```pseudocode
// Global error handling middleware
class ErrorHandlingMiddleware extends BaseHTTPMiddleware:
    
    function dispatch(request, call_next):
        try:
            response = call call_next(request)
            return response
            
        catch BaseAPIException as api_exception:
            log_warning("API Exception: " + api_exception.message)
            return create_json_response(
                status_code: api_exception.status_code,
                content: {
                    error: api_exception.error_code or "API_ERROR",
                    detail: api_exception.message,
                    timestamp: current_timestamp(),
                    path: request.url
                }
            )
            
        catch HTTPException as http_exception:
            log_warning("HTTP Exception: " + http_exception.detail)
            return create_json_response(
                status_code: http_exception.status_code,
                content: {
                    error: "HTTP_ERROR",
                    detail: http_exception.detail,
                    timestamp: current_timestamp(),
                    path: request.url
                }
            )
            
        catch RequestValidationError as validation_exception:
            log_warning("Validation Error: " + validation_exception.message)
            return create_json_response(
                status_code: 422,
                content: {
                    error: "VALIDATION_ERROR",
                    detail: "Request validation failed",
                    errors: validation_exception.errors(),
                    timestamp: current_timestamp(),
                    path: request.url
                }
            )
            
        catch Exception as generic_exception:
            log_error("Unexpected error: " + generic_exception.message)
            log_error(get_stack_trace())
            
            return create_json_response(
                status_code: 500,
                content: {
                    error: "INTERNAL_ERROR",
                    detail: "An internal server error occurred",
                    timestamp: current_timestamp(),
                    path: request.url
                }
            )
```

## 9. Testing Strategy

### 9.1 Test Configuration
```pseudocode
// Pytest configuration and fixtures
test_configuration:
    
    fixture event_loop (scope: session):
        // Create event loop for async test session
        loop = create_new_event_loop()
        yield loop
        close_loop(loop)

    fixture client():
        // Create test client for FastAPI application
        return create_test_client(app)

    fixture redis_client (async):
        // Create Redis client for testing
        client = create_redis_client()
        yield client
        call client.flush_database()  // Clean up after each test
        call client.close()

    fixture sample_image():
        // Create sample image for testing
        image = create_rgb_image(size=(224, 224), color="red")
        image_bytes = create_byte_buffer()
        save_image_to_buffer(image, image_bytes, format="JPEG")
        return image_bytes.get_value()
```

### 9.2 API Tests
```pseudocode
// API endpoint tests
class TestClassificationAPI:
    
    test_classify_image_success(client, sample_image):
        // Test successful image classification
        files = {"file": ("test.jpg", create_byte_buffer(sample_image), "image/jpeg")}
        
        mock_task = create_mock()
        mock_task.apply_async.return_value.id = "test-task-id"
        
        with patch("app.tasks.classification.classify_image_task", mock_task):
            response = call client.post("/api/v1/classify", files=files)
            
            assert response.status_code equals 200
            data = response.json()
            assert "job_id" exists in data
            assert data["status"] equals "pending"
    
    test_classify_image_invalid_file_type(client):
        // Test classification with invalid file type
        files = {"file": ("test.txt", create_byte_buffer("not an image"), "text/plain")}
        
        response = call client.post("/api/v1/classify", files=files)
        
        assert response.status_code equals 400
        assert "Unsupported file type" in response.json()["detail"]
    
    test_classify_image_file_too_large(client):
        // Test classification with file too large
        large_file = create_bytes(11_MB)  // 11MB file
        files = {"file": ("large.jpg", create_byte_buffer(large_file), "image/jpeg")}
        
        response = call client.post("/api/v1/classify", files=files)
        
        assert response.status_code equals 400
        assert "exceeds limit" in response.json()["detail"]
    
    test_get_job_status_success(client):
        // Test successful job status retrieval
        mock_status = create_mock_object()
        mock_status.job_id = "test-job-id"
        mock_status.status = "processing"
        mock_status.progress = 50
        
        with patch("app.services.job_service.JobService.get_job_status") as mock_get_status:
            mock_get_status.return_value = mock_status
            
            response = call client.get("/api/v1/status/test-job-id")
            
            assert response.status_code equals 200
            data = response.json()
            assert data["job_id"] equals "test-job-id"
            assert data["status"] equals "processing"
            assert data["progress"] equals 50
    
    test_get_job_status_not_found(client):
        // Test job status for non-existent job
        with patch("app.services.job_service.JobService.get_job_status") as mock_get_status:
            mock_get_status.return_value = null
            
            response = call client.get("/api/v1/status/non-existent-job")
            
            assert response.status_code equals 404
            assert "Job not found" in response.json()["detail"]

    test_get_result_success(client):
        // Test successful result retrieval
        mock_job_status = create_mock_object()
        mock_job_status.status = "completed"
        
        mock_result = create_mock_classification_result()
        
        with patch("app.services.job_service.JobService.get_job_status") as mock_status:
            with patch("app.services.job_service.JobService.get_job_result") as mock_result_call:
                mock_status.return_value = mock_job_status
                mock_result_call.return_value = mock_result
                
                response = call client.get("/api/v1/result/test-job-id")
                
                assert response.status_code equals 200
                data = response.json()
                assert "predictions" exists in data
                assert "processing_time" exists in data

    test_get_result_job_not_completed(client):
        // Test result retrieval for incomplete job
        mock_job_status = create_mock_object()
        mock_job_status.status = "processing"
        
        with patch("app.services.job_service.JobService.get_job_status") as mock_status:
            mock_status.return_value = mock_job_status
            
            response = call client.get("/api/v1/result/test-job-id")
            
            assert response.status_code equals 409
            assert "Job not completed" in response.json()["detail"]
```

This Low Level Design provides comprehensive implementation details for the Async Image Classification Service, covering all aspects from data models to testing strategies. The design is structured to ensure maintainable, scalable, and testable code that aligns with the high-level architecture.