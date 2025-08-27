# The Twelve-Factor App

**Adam Wiggins and the Heroku Team**  
*Originally published 2011, continuously updated*

**Original Source:** [Local HTML](sources/twelve-factor-app.html) | [Online](https://12factor.net)

## Overview

The Twelve-Factor App is a methodology for building software-as-a-service (SaaS) applications that are portable, scalable, and maintainable. Born from Heroku's experience running thousands of applications, these principles have become the de facto standard for cloud-native development. They address the gap between development and production, enabling continuous deployment and horizontal scaling.

## Context and Motivation

### Problems Addressed
- **Configuration drift**: Hardcoded values, environment-specific code
- **Dependency hell**: "Works on my machine" syndrome
- **Scaling bottlenecks**: Stateful services, vertical scaling limitations
- **Deployment friction**: Manual processes, long release cycles
- **Operational opacity**: Poor observability, debugging difficulties

### Goals
- **Declarative automation**: Minimize time and cost for new developers
- **Clean contract**: Maximum portability between execution environments
- **Cloud platform deployment**: Without servers or systems administration
- **Continuous deployment**: Minimize divergence between development and production
- **Horizontal scaling**: Without significant changes to tooling or architecture

## The Twelve Factors

### I. Codebase
**One codebase tracked in revision control, many deploys**

Principles:
- Single repository per app (monorepo or polyrepo both valid at organization level)
- One-to-one correlation between codebase and app
- Multiple deploys (staging, production, developer environments) from same codebase
- Different versions may be deployed, but same codebase

Anti-patterns:
```
❌ Multiple apps sharing a codebase (should be libraries)
❌ Multiple codebases for one app (should be one codebase)
❌ Copy-paste code between projects (should be shared libraries)
```

Best practices:
```
✅ Git, Mercurial, or other distributed VCS
✅ Clear branching strategy (GitFlow, GitHub Flow, etc.)
✅ Tagged releases for production deploys
```

### II. Dependencies
**Explicitly declare and isolate dependencies**

Principles:
- Never rely on system-wide packages
- Declare all dependencies explicitly via manifest
- Use dependency isolation tools during execution
- Include system tools if needed (e.g., curl, ImageMagick)

Implementation by language:
```python
# Python: requirements.txt or Pipfile
flask==2.0.1
postgresql==13.3
redis==6.2.4

# Isolation: virtualenv, venv, pipenv
```

```javascript
// Node.js: package.json
{
  "dependencies": {
    "express": "^4.17.1",
    "pg": "^8.6.0",
    "redis": "^3.1.2"
  }
}
// Isolation: npm/yarn install creates node_modules
```

```ruby
# Ruby: Gemfile
source 'https://rubygems.org'
gem 'rails', '~> 6.1.4'
gem 'pg', '~> 1.2'
gem 'redis', '~> 4.3'

# Isolation: bundle exec
```

### III. Config
**Store config in the environment**

Config varies between deploys, code doesn't.

Config includes:
- Database URLs and credentials
- API keys for external services
- Per-deploy values (canonical hostname)
- Feature flags

Principles:
- Strict separation of config from code
- Config never checked into version control
- Environment variables for language/OS agnostic solution
- Granular variables over grouped configurations

Good example:
```python
import os

DATABASE_URL = os.environ['DATABASE_URL']
STRIPE_API_KEY = os.environ['STRIPE_API_KEY']
FEATURE_NEW_UI = os.environ.get('FEATURE_NEW_UI', 'false') == 'true'
```

Bad example:
```python
# ❌ Environment-specific files
if environment == 'production':
    config = load('config/production.yml')
elif environment == 'staging':
    config = load('config/staging.yml')
```

### IV. Backing Services
**Treat backing services as attached resources**

Backing service examples:
- Databases (PostgreSQL, MySQL)
- Message queues (RabbitMQ, AWS SQS)
- Caching systems (Redis, Memcached)
- Email services (SendGrid, AWS SES)
- File storage (S3, CloudFiles)

Principles:
- No code changes when swapping providers
- Resources attached/detached via config
- Local and third-party services treated identically

Implementation:
```python
# Database swap via config change only
DATABASE_URL=postgres://user:pass@localhost/myapp_dev  # Local PostgreSQL
DATABASE_URL=mysql://user:pass@rds.aws.com/myapp      # AWS RDS MySQL

# Code remains unchanged
db = connect(os.environ['DATABASE_URL'])
```

### V. Build, Release, Run
**Strictly separate build and run stages**

Three stages:
1. **Build**: Transform repo into executable bundle
   - Compile code
   - Bundle dependencies
   - Minify assets
   - Output: build artifact

2. **Release**: Combine build with config
   - Take build artifact
   - Add current config
   - Output: release (immutable, versioned)

3. **Run**: Launch processes from release
   - Execute release in environment
   - Should be simple as possible
   - No code changes possible

Example pipeline:
```bash
# Build stage
git clone https://github.com/myapp/myapp.git
npm install
npm run build
docker build -t myapp:${COMMIT_SHA} .

# Release stage
docker tag myapp:${COMMIT_SHA} myapp:release-${VERSION}
kubectl set image deployment/myapp myapp=myapp:release-${VERSION}

# Run stage (handled by orchestrator)
kubectl rollout status deployment/myapp
```

### VI. Processes
**Execute the app as one or more stateless processes**

Principles:
- Processes are stateless and share-nothing
- Persistent data in stateful backing services
- Memory/filesystem used as brief, single-transaction cache
- No sticky sessions (session affinity)

Good patterns:
```python
# ✅ Store session in Redis
session_data = redis.get(session_id)

# ✅ Store uploaded files in S3
s3.upload_file(file, bucket, key)
```

Bad patterns:
```python
# ❌ In-memory session storage
sessions[session_id] = user_data

# ❌ Local filesystem for permanent storage
with open(f'/uploads/{filename}', 'wb') as f:
    f.write(uploaded_file)
```

### VII. Port Binding
**Export services via port binding**

Principles:
- App is completely self-contained
- Exports HTTP as service by binding to port
- No runtime injection of webserver
- In production, routing layer handles requests

Implementation:
```python
# Python Flask
if __name__ == '__main__':
    port = int(os.environ.get('PORT', 5000))
    app.run(host='0.0.0.0', port=port)
```

```javascript
// Node.js Express
const port = process.env.PORT || 3000;
app.listen(port, () => {
    console.log(`Server running on port ${port}`);
});
```

Enables:
- Local development (http://localhost:5000)
- Service composition (microservices)
- Platform flexibility

### VIII. Concurrency
**Scale out via the process model**

Principles:
- Processes are first-class citizens
- Workload diversity via process types
- Horizontal scaling by adding processes
- Never daemonize or write PID files

Process types:
- **Web**: Handle HTTP requests
- **Worker**: Process background jobs
- **Clock**: Schedule periodic tasks

Procfile example:
```yaml
web: gunicorn app:application
worker: celery -A app worker
clock: python scheduler.py
```

Scaling:
```bash
# Scale horizontally
heroku ps:scale web=10 worker=5 clock=1

# Or with Kubernetes
kubectl scale deployment web --replicas=10
kubectl scale deployment worker --replicas=5
```

### IX. Disposability
**Maximize robustness with fast startup and graceful shutdown**

Principles:
- Processes can be started/stopped at moment's notice
- Minimize startup time (seconds, not minutes)
- Shut down gracefully on SIGTERM
- Robust against sudden death

Graceful shutdown:
```python
import signal
import sys

def graceful_shutdown(signum, frame):
    print("Received shutdown signal")
    # Finish current request
    # Close database connections
    # Stop accepting new work
    sys.exit(0)

signal.signal(signal.SIGTERM, graceful_shutdown)
```

Robust job processing:
```python
# Make jobs idempotent and reentrant
def process_payment(payment_id):
    payment = get_payment(payment_id)
    if payment.status == 'completed':
        return  # Already processed, safe to retry
    
    # Process payment...
    payment.status = 'completed'
    payment.save()
```

### X. Dev/Prod Parity
**Keep development, staging, and production as similar as possible**

Traditional gaps and twelve-factor solutions:

| Gap | Traditional | Twelve-Factor |
|-----|-------------|---------------|
| **Time** | Weeks between deploys | Hours or minutes |
| **Personnel** | Developers write, ops deploy | Developers deploy |
| **Tools** | Different stack (SQLite vs PostgreSQL) | Same stack everywhere |

Implementation:
```yaml
# docker-compose.yml for development
version: '3'
services:
  web:
    build: .
    environment:
      DATABASE_URL: postgres://user:pass@db:5432/myapp
  db:
    image: postgres:13
    
# Same PostgreSQL in production
DATABASE_URL: postgres://user:pass@rds.amazonaws.com:5432/myapp
```

Benefits:
- Reduced bugs from environment differences
- Faster onboarding
- Continuous deployment enabled

### XI. Logs
**Treat logs as event streams**

Principles:
- App never concerns itself with routing or storage
- Write all logs to stdout
- Execution environment handles capture, routing, and storage
- No log files or log rotation in app

Application code:
```python
import sys
import json
from datetime import datetime

def log(level, message, **kwargs):
    entry = {
        'timestamp': datetime.utcnow().isoformat(),
        'level': level,
        'message': message,
        **kwargs
    }
    print(json.dumps(entry), file=sys.stdout)

# Usage
log('INFO', 'User logged in', user_id=123, ip='192.168.1.1')
```

Platform handles routing:
```bash
# Local development
python app.py

# Production with log aggregation
python app.py | tee >(aws logs put-log-events --log-group-name myapp)

# Or container orchestration handles it
kubectl logs -f deployment/myapp
```

Use cases for log stream:
- Finding specific events
- Graphing trends (requests per minute)
- Active alerting (error rate threshold)

### XII. Admin Processes
**Run admin/management tasks as one-off processes**

Examples:
- Database migrations
- Console for inspection (REPL)
- One-time scripts

Principles:
- Run in identical environment as regular processes
- Run against same release
- Ship admin code with application code

Implementation:
```bash
# Database migration
heroku run rails db:migrate

# Interactive console
heroku run rails console

# One-off script
heroku run python scripts/fix_user_data.py

# With containers
kubectl run -it --rm debug --image=myapp:latest --restart=Never -- python
```

Keep admin code in sync:
```python
# management/commands/cleanup_old_data.py
from app.models import Record
from datetime import datetime, timedelta

def cleanup_old_records():
    cutoff = datetime.now() - timedelta(days=90)
    old_records = Record.objects.filter(created_at__lt=cutoff)
    count = old_records.count()
    old_records.delete()
    print(f"Deleted {count} old records")

if __name__ == '__main__':
    cleanup_old_records()
```

## Beyond the Original Twelve

The community has proposed additional factors:

### API First
Design API before implementation, enabling:
- Parallel development (frontend/backend)
- Service composition
- Third-party integrations

### Telemetry
Built-in monitoring and metrics:
- Application Performance Monitoring (APM)
- Distributed tracing
- Business metrics

### Security
Security as first-class concern:
- Secrets management (beyond environment variables)
- Encryption at rest and in transit
- Regular dependency updates

## Implementation Strategies

### Containerization
Docker naturally aligns with twelve-factor:
```dockerfile
# Dockerfile
FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["gunicorn", "--bind", "0.0.0.0:$PORT", "app:application"]
```

### Orchestration
Kubernetes implements twelve-factor patterns:
```yaml
# Kubernetes Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3  # Factor VIII: Concurrency
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest  # Factor V: Build, Release, Run
        ports:
        - containerPort: 8080  # Factor VII: Port binding
        env:  # Factor III: Config
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: database-url
```

### Platform-as-a-Service
PaaS providers built around twelve-factor:
- **Heroku**: Original inspiration, native support
- **Cloud Foundry**: Enterprise PaaS
- **Google App Engine**: Serverless approach
- **AWS Elastic Beanstalk**: AWS's PaaS offering

## Common Challenges and Solutions

### Challenge: Binary Dependencies
Some apps need system libraries (ImageMagick, FFmpeg).

Solution:
```dockerfile
# Include in container
FROM python:3.9
RUN apt-get update && apt-get install -y \
    imagemagick \
    ffmpeg
```

### Challenge: Background Jobs
Long-running tasks don't fit request/response.

Solution:
```python
# Separate worker process type
# web.py
def upload_video(request):
    job_id = queue.enqueue('process_video', request.files['video'])
    return {'job_id': job_id}

# worker.py
def process_video(video_file):
    # Long-running task
    transcoded = transcode(video_file)
    upload_to_s3(transcoded)
```

### Challenge: Scheduled Tasks
Cron-like functionality needed.

Solution:
```python
# Clock process
import schedule
import time

schedule.every().hour.do(cleanup_temp_files)
schedule.every().day.at("02:00").do(send_daily_reports)

while True:
    schedule.run_pending()
    time.sleep(60)
```

## Criticisms and Limitations

### Valid Criticisms
1. **Stateful Services**: Some apps genuinely need state (databases, caches)
2. **Environment Variable Limitations**: Complex configs, secrets rotation
3. **Monolith Bias**: Microservices require adaptation
4. **Binary Assets**: Large files don't fit the model
5. **Development Overhead**: Can be overkill for simple apps

### When Twelve-Factor Doesn't Fit
- Desktop applications
- Mobile apps
- Embedded systems
- High-performance computing
- Legacy system migrations

## Modern Evolution

### Serverless and Twelve-Factor
Serverless platforms enforce many factors:
- **Stateless**: Functions are ephemeral
- **Concurrency**: Automatic scaling
- **Disposability**: Fast startup critical

### Service Mesh
Istio, Linkerd handle cross-cutting concerns:
- **Port Binding**: Automatic sidecar injection
- **Backing Services**: Service discovery
- **Logs**: Distributed tracing

### GitOps
Extends twelve-factor with:
- Declarative infrastructure
- Version-controlled operations
- Automated reconciliation

## Practical Checklist

Essential twelve-factor compliance:

```markdown
- [ ] Single codebase in version control
- [ ] Dependencies explicitly declared
- [ ] Configuration in environment variables
- [ ] Backing services configurable via URLs
- [ ] Separate build and run stages
- [ ] Stateless processes
- [ ] Services exposed via port binding
- [ ] Horizontally scalable processes
- [ ] Fast startup and graceful shutdown
- [ ] Dev/prod environment parity
- [ ] Logs written to stdout
- [ ] Admin tasks as one-off processes
```

## Impact and Legacy

### Industry Adoption
- **Cloud-Native Computing Foundation**: Twelve-factor influences CNCF projects
- **Microservices**: Natural fit for twelve-factor
- **DevOps**: Bridges development and operations
- **Continuous Deployment**: Enables rapid iteration

### Tools Built on Twelve-Factor
- **Buildpacks**: Heroku, Cloud Foundry, Paketo
- **Procfile**: Process declaration standard
- **dotenv**: Environment variable management
- **Foreman/Honcho**: Local development process managers

## Key Takeaways

1. **Declarative Over Imperative**: Describe what, not how
2. **Explicit Over Implicit**: No hidden dependencies or configs
3. **Stateless Over Stateful**: Push state to edges
4. **Horizontal Over Vertical**: Scale out, not up
5. **Automation Over Manual**: Machines do operations

## Conclusion

The Twelve-Factor App methodology transformed how we build and deploy web applications. While not every factor applies to every situation, the principles provide a solid foundation for cloud-native development. The methodology's emphasis on automation, explicitness, and operational excellence continues to influence modern platforms and practices.

As applications evolve toward serverless, edge computing, and AI-driven systems, the core insights remain valuable: separate config from code, treat infrastructure as disposable, and design for horizontal scaling. The twelve factors aren't rules but guidelines—apply them thoughtfully based on your specific requirements and constraints.