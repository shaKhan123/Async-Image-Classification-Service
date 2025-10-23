# Async Image Classification Service

A scalable, asynchronous image classification service built with FastAPI, Celery, and PyTorch. This project demonstrates modern microservices architecture for ML workloads with real-time progress tracking and containerized deployment.

## 🏗️ Architecture Overview

```
Client → FastAPI → Redis Queue → Celery Workers → Redis Results → FastAPI → Client
                                      ↓
                              PyTorch Model Processing
```

### Core Components

- **FastAPI API Server**: REST endpoints for job submission and status tracking
- **Celery Workers**: Background image processing with PyTorch models
- **Redis**: Message broker, result backend, and progress tracking
- **Docker**: Containerized deployment and orchestration

## 🚀 Quick Start

### Prerequisites

- Docker and Docker Compose
- Python 3.11+ (for local development)
- 4GB+ RAM (for PyTorch models)

### Demo Deployment

1. **Clone and Setup**
   ```bash
   git clone <repository-url>
   cd async-image-classifier
   ```

2. **Start Services**
   ```bash
   docker-compose up -d
   ```

3. **Verify Deployment**
   ```bash
   curl http://localhost:8000/health
   ```

4. **Test Classification**
   ```bash
   curl -X POST -F "file=@test_image.jpg" http://localhost:8000/classify
   # Returns: {"job_id": "uuid", "status": "pending"}
   
   curl http://localhost:8000/status/{job_id}
   # Returns: {"job_id": "uuid", "status": "processing", "progress": 45}
   
   curl http://localhost:8000/result/{job_id}
   # Returns: {"job_id": "uuid", "predictions": [...], "processing_time": 1.2}
   ```

## 📚 API Documentation

### Endpoints

#### `POST /classify`
Submit an image for classification.

**Request:**
- Method: `POST`
- Content-Type: `multipart/form-data`
- Body: Image file (JPEG, PNG)
- Max file size: 10MB

**Response:**
```json
{
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "pending"
}
```

#### `GET /status/{job_id}`
Check job processing status and progress.

**Response:**
```json
{
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "processing",
  "progress": 75,
  "updated_at": "2025-10-20T10:30:00Z"
}
```

**Status Values:**
- `pending`: Job queued, waiting for worker
- `processing`: Job being processed by worker
- `completed`: Job finished successfully
- `failed`: Job failed with error

#### `GET /result/{job_id}`
Retrieve classification results.

**Response:**
```json
{
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "predictions": [
    {"class_name": "cat", "confidence": 0.95},
    {"class_name": "dog", "confidence": 0.03},
    {"class_name": "bird", "confidence": 0.02}
  ],
  "processing_time": 1.23,
  "created_at": "2025-10-20T10:30:00Z"
}
```

#### `GET /health`
Service health check.

**Response:**
```json
{
  "status": "healthy",
  "redis_connected": true,
  "workers_available": 2,
  "timestamp": "2025-10-20T10:30:00Z"
}
```

### Interactive API Docs

- Swagger UI: `http://localhost:8000/docs`
- ReDoc: `http://localhost:8000/redoc`

## 🛠️ Technology Stack

### Core Technologies
- **[FastAPI](https://fastapi.tiangolo.com/)** (0.104+): Modern async web framework
- **[Celery](https://docs.celeryq.dev/)** (5.3+): Distributed task queue
- **[Redis](https://redis.io/)** (7.0+): Message broker and result backend
- **[PyTorch](https://pytorch.org/)** (2.0+): Deep learning framework
- **[Docker](https://www.docker.com/)**: Containerization platform

### Supporting Libraries
- **Pillow**: Image processing
- **Pydantic**: Data validation and serialization
- **Uvicorn**: ASGI server
- **aioredis**: Async Redis client

## 🏃‍♂️ Development

### Local Development Setup

1. **Install Dependencies**
   ```bash
   pip install -r requirements.txt
   pip install -r requirements-dev.txt
   ```

2. **Start Redis**
   ```bash
   docker run -d -p 6379:6379 redis:7-alpine
   ```

3. **Start Celery Worker**
   ```bash
   celery -A app.celery_app worker --loglevel=info
   ```

4. **Start FastAPI Server**
   ```bash
   uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
   ```

### Project Structure

```
async-image-classifier/
├── app/
│   ├── __init__.py
│   ├── main.py              # FastAPI application
│   ├── celery_app.py        # Celery configuration
│   ├── models/
│   │   ├── __init__.py
│   │   ├── schemas.py       # Pydantic models
│   │   └── ml_models.py     # PyTorch model loading
│   ├── api/
│   │   ├── __init__.py
│   │   └── endpoints.py     # API route handlers
│   ├── core/
│   │   ├── __init__.py
│   │   ├── config.py        # Application settings
│   │   └── redis_client.py  # Redis connection
│   └── tasks/
│       ├── __init__.py
│       └── classification.py # Celery tasks
├── models/                  # ML model files
├── tests/
├── docker/
│   ├── Dockerfile.api
│   ├── Dockerfile.worker
│   └── docker-compose.yml
├── requirements.txt
├── requirements-dev.txt
└── README.md
```

### Testing

```bash
# Run all tests
pytest

# Run with coverage
pytest --cov=app tests/

# Run specific test category
pytest tests/test_api.py
pytest tests/test_tasks.py
```

### Code Quality

```bash
# Format code
black app/ tests/

# Lint code
flake8 app/ tests/

# Type checking
mypy app/
```

## 🐳 Docker Configuration

### Demo Configuration (docker-compose.yml)

```yaml
version: '3.8'

services:
  api:
    build:
      context: .
      dockerfile: docker/Dockerfile.api
    ports:
      - "8000:8000"
    environment:
      - REDIS_URL=redis://redis:6379
    depends_on:
      - redis
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  worker:
    build:
      context: .
      dockerfile: docker/Dockerfile.worker
    environment:
      - REDIS_URL=redis://redis:6379
    depends_on:
      - redis
    volumes:
      - ./models:/app/models:ro
    deploy:
      replicas: 2

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes

volumes:
  redis_data:
```

### Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `REDIS_URL` | Redis connection URL | `redis://localhost:6379` |
| `MODEL_PATH` | Path to PyTorch model file | `./models/classifier.pth` |
| `MAX_FILE_SIZE` | Maximum upload file size | `10485760` (10MB) |
| `RESULT_TTL` | Result cache TTL (seconds) | `3600` (1 hour) |
| `LOG_LEVEL` | Logging level | `INFO` |

## 📊 Monitoring & Observability

### Demo Monitoring

- **Logs**: Docker logs via `docker-compose logs`
- **Health Checks**: `/health` endpoint
- **Redis Monitoring**: `redis-cli monitor`

### Metrics Available

- Request latency and throughput
- Queue depth and processing time
- Worker utilization
- Model inference time
- Error rates by endpoint

## 🔧 Configuration

### Redis Configuration

```bash
# Demo settings (sufficient for development)
maxmemory 256mb
maxmemory-policy allkeys-lru
save 60 1000
```

### Celery Configuration

```python
# Demo settings
CELERY_BROKER_URL = "redis://redis:6379/0"
CELERY_RESULT_BACKEND = "redis://redis:6379/0"
CELERY_TASK_SERIALIZER = "json"
CELERY_RESULT_SERIALIZER = "json"
CELERY_TASK_TRACK_STARTED = True
CELERY_TASK_TIME_LIMIT = 300  # 5 minutes
CELERY_WORKER_CONCURRENCY = 2
```

## 🚀 Production Deployment

### Production Considerations

> **⚠️ Important**: This demo configuration is not production-ready. For production deployment, consider the following enhancements:

#### Security
- [ ] Implement OAuth2/JWT authentication
- [ ] Add rate limiting and request throttling
- [ ] Input validation and file type restrictions
- [ ] Network segmentation and firewall rules
- [ ] Secrets management (HashiCorp Vault, K8s secrets)
- [ ] HTTPS/TLS encryption
- [ ] Image content scanning for malicious files

#### Scalability
- [ ] Horizontal auto-scaling for API and workers
- [ ] Load balancing (NGINX, HAProxy, cloud LB)
- [ ] Redis Cluster for high availability
- [ ] CDN for static assets and model files
- [ ] Queue-based auto-scaling triggers
- [ ] Multi-region deployment

#### Reliability
- [ ] Database persistence for job metadata
- [ ] Backup and disaster recovery procedures
- [ ] Circuit breakers and retry policies
- [ ] Graceful shutdown handling
- [ ] Dead letter queues for failed jobs

#### Monitoring
- [ ] Structured logging (JSON format)
- [ ] Metrics collection (Prometheus/Grafana)
- [ ] Distributed tracing (Jaeger/Zipkin)
- [ ] Error tracking (Sentry)
- [ ] Model performance monitoring
- [ ] Business metrics dashboards

#### Performance
- [ ] Model optimization (ONNX, TensorRT)
- [ ] GPU utilization for workers
- [ ] Batch processing for multiple images
- [ ] Result caching strategies
- [ ] Connection pooling
- [ ] Async I/O optimization

#### Data Management
- [ ] Result archival to object storage (S3)
- [ ] Data retention and cleanup policies
- [ ] GDPR compliance for uploaded images
- [ ] Model versioning and A/B testing
- [ ] Audit logging for compliance

### Kubernetes Deployment

For production Kubernetes deployment:

```bash
# Example production deployment
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/redis-cluster.yaml
kubectl apply -f k8s/api-deployment.yaml
kubectl apply -f k8s/worker-deployment.yaml
kubectl apply -f k8s/ingress.yaml
```

### CI/CD Pipeline

Recommended CI/CD stages:
1. Code quality checks (linting, typing, security scanning)
2. Unit and integration tests
3. Build and push container images
4. Deploy to staging environment
5. Run E2E tests
6. Deploy to production with blue-green strategy

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit changes (`git commit -m 'Add amazing feature'`)
4. Push to branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

### Development Guidelines

- Write tests for new features
- Follow PEP 8 style guidelines
- Add type hints to all functions
- Update documentation for API changes
- Ensure Docker builds pass

## 📝 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🙋‍♂️ Support

- **Issues**: [GitHub Issues](link-to-issues)
- **Discussions**: [GitHub Discussions](link-to-discussions)
- **Documentation**: [Project Wiki](link-to-wiki)

## 🔗 Related Projects

- [FastAPI](https://github.com/tiangolo/fastapi)
- [Celery](https://github.com/celery/celery)
- [PyTorch](https://github.com/pytorch/pytorch)
- [Redis](https://github.com/redis/redis)

---

**Note**: This is a demonstration project showcasing async ML service architecture. For production use, implement the security, scalability, and monitoring considerations outlined above.