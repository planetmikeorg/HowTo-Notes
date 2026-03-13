# GitLab CI/CD Pipeline Guide (.gitlab-ci.yml)

A comprehensive guide to creating and managing GitLab CI/CD pipelines using `.gitlab-ci.yml`.

## Table of Contents
- [Pipeline Basics](#pipeline-basics)
- [Configuration Structure](#configuration-structure)
- [Global Keywords](#global-keywords)
- [Job Configuration](#job-configuration)
- [Stages](#stages)
- [Scripts and Commands](#scripts-and-commands)
- [Variables](#variables)
- [Artifacts](#artifacts)
- [Cache](#cache)
- [Dependencies](#dependencies)
- [Rules and Conditions](#rules-and-conditions)
- [Docker Integration](#docker-integration)
- [Services](#services)
- [Environments](#environments)
- [Triggers and Pipelines](#triggers-and-pipelines)
- [Templates and Includes](#templates-and-includes)
- [Advanced Features](#advanced-features)
- [Real-World Examples](#real-world-examples)
- [Best Practices](#best-practices)
- [Troubleshooting](#troubleshooting)

---

## Pipeline Basics

### What is .gitlab-ci.yml?
The `.gitlab-ci.yml` file defines your CI/CD pipeline configuration. It must be placed in the root of your repository.

### Basic Pipeline Structure
```yaml
# Define stages (order matters)
stages:
  - build
  - test
  - deploy

# Define jobs
build-job:
  stage: build
  script:
    - echo "Building the application"
    - make build

test-job:
  stage: test
  script:
    - echo "Running tests"
    - make test

deploy-job:
  stage: deploy
  script:
    - echo "Deploying application"
    - make deploy
```

### Pipeline Execution Flow
1. Pipeline triggered by push, merge request, schedule, or manual action
2. Jobs in the same stage run in parallel
3. Stages execute sequentially
4. If any job fails, the pipeline stops (by default)

---

## Configuration Structure

### File Format
```yaml
# Comments start with #
# Use 2 spaces for indentation (not tabs)

# Global configuration
image: rockylinux:9
variables:
  GLOBAL_VAR: "value"

# Stage definitions
stages:
  - stage1
  - stage2

# Job definitions
job_name:
  stage: stage1
  script:
    - command1
    - command2
```

### YAML Syntax Tips
```yaml
# Multi-line strings
script:
  - |
    echo "Multi-line"
    echo "script block"
  
  - >
    echo "This is folded"
    echo "into one line"

# Anchors and aliases (reuse configuration)
.template: &template_config
  image: rockylinux:9
  cache:
    paths:
      - .npm/

job1:
  <<: *template_config
  script:
    - npm install
```

---

## Global Keywords

### default
Apply settings to all jobs:
```yaml
default:
  image: rockylinux:9
  before_script:
    - echo "This runs before every job"
  after_script:
    - echo "This runs after every job"
  retry: 2
  timeout: 1h
  tags:
    - docker
  cache:
    paths:
      - node_modules/
```

### workflow
Control when pipelines are created:
```yaml
workflow:
  rules:
    # Run pipeline for merge requests
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    # Run pipeline for default branch
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
    # Run pipeline for tags
    - if: '$CI_COMMIT_TAG'
```

### include
Include external YAML files:
```yaml
include:
  # Local file
  - local: '/templates/.gitlab-ci-template.yml'
  
  # Remote file
  - remote: 'https://example.com/ci-config.yml'
  
  # Template from GitLab
  - template: Security/SAST.gitlab-ci.yml
  
  # File from another project
  - project: 'group/project'
    ref: main
    file: '/templates/ci-config.yml'
```

---

## Job Configuration

### Basic Job Structure
```yaml
job_name:
  # Stage assignment
  stage: build
  
  # Docker image
  image: node:18
  
  # Commands to execute
  script:
    - npm install
    - npm run build
  
  # When to run
  only:
    - main
  
  # Job dependencies
  needs: []
  
  # Artifacts
  artifacts:
    paths:
      - dist/
```

### Job Keywords Reference

#### stage
Assign job to a stage:
```yaml
build-app:
  stage: build
  script:
    - make build
```

#### script
Commands to execute (required):
```yaml
test-app:
  script:
    - npm install
    - npm test
    - echo "Tests complete"
```

#### before_script
Commands before main script:
```yaml
deploy-app:
  before_script:
    - echo "Preparing deployment"
    - chmod +x deploy.sh
  script:
    - ./deploy.sh
```

#### after_script
Commands after main script (always runs):
```yaml
build-app:
  script:
    - make build
  after_script:
    - echo "Cleaning up"
    - rm -rf temp/
```

#### image
Specify Docker image:
```yaml
ruby-job:
  image: ruby:3.2
  script:
    - ruby --version

python-job:
  image:
    name: python:3.11
    entrypoint: [""]  # Override entrypoint
  script:
    - python --version
```

#### tags
Runner tags for job selection:
```yaml
docker-job:
  tags:
    - docker
    - linux

kubernetes-job:
  tags:
    - kubernetes
    - production
```

#### allow_failure
Continue pipeline even if job fails:
```yaml
test-experimental:
  script:
    - npm run experimental-tests
  allow_failure: true

# Exit codes that don't fail the job
test-with-warnings:
  script:
    - run-tests.sh
  allow_failure:
    exit_codes:
      - 137
      - 255
```

#### when
Control job execution timing:
```yaml
# Run on success (default)
deploy-prod:
  when: on_success
  script:
    - deploy.sh

# Always run
cleanup:
  when: always
  script:
    - cleanup.sh

# Run on failure
notify-failure:
  when: on_failure
  script:
    - send-alert.sh

# Manual trigger required
deploy-manual:
  when: manual
  script:
    - deploy-prod.sh

# Never run automatically
disabled-job:
  when: never
  script:
    - echo "This won't run"

# Run only if delayed
delayed-job:
  when: delayed
  start_in: 30 minutes
  script:
    - run-maintenance.sh
```

#### retry
Retry failed jobs:
```yaml
test-flaky:
  script:
    - npm test
  retry: 2

# Retry with conditions
test-conditional:
  script:
    - run-tests.sh
  retry:
    max: 2
    when:
      - runner_system_failure
      - stuck_or_timeout_failure
      - script_failure
```

#### timeout
Job timeout:
```yaml
long-job:
  script:
    - long-running-process.sh
  timeout: 3h

short-job:
  script:
    - quick-test.sh
  timeout: 5 minutes
```

---

## Stages

### Defining Stages
```yaml
stages:
  - .pre          # Special stage, runs first
  - build
  - test
  - security
  - deploy
  - .post         # Special stage, runs last
```

### Stage Execution
- Jobs in same stage run in parallel
- Stages run sequentially
- Pipeline stops if a stage fails

```yaml
stages:
  - build
  - test
  - deploy

# These run in parallel
build-frontend:
  stage: build
  script:
    - npm run build

build-backend:
  stage: build
  script:
    - go build

# These run after build completes
unit-tests:
  stage: test
  script:
    - npm test

integration-tests:
  stage: test
  script:
    - pytest
```

---

## Scripts and Commands

### Single Commands
```yaml
simple-job:
  script:
    - echo "Hello World"
```

### Multiple Commands
```yaml
complex-job:
  script:
    - echo "Step 1"
    - make install
    - make test
    - echo "Step 4"
```

### Multi-line Scripts
```yaml
multiline-job:
  script:
    - |
      echo "Starting process"
      for i in {1..5}; do
        echo "Iteration $i"
      done
      echo "Process complete"
```

### Conditional Commands
```yaml
conditional-job:
  script:
    - |
      if [ "$CI_COMMIT_BRANCH" == "main" ]; then
        echo "Deploying to production"
        deploy-prod.sh
      else
        echo "Deploying to staging"
        deploy-staging.sh
      fi
```

### Error Handling
```yaml
error-handling:
  script:
    # Continue on error
    - command_that_might_fail || true
    
    # Exit on any error
    - set -e
    - critical_command
    
    # Custom error handling
    - |
      if ! make test; then
        echo "Tests failed, sending notification"
        notify-team.sh
        exit 1
      fi
```

---

## Variables

### Predefined Variables
GitLab provides many CI/CD variables:
```yaml
show-variables:
  script:
    - echo "Pipeline ID: $CI_PIPELINE_ID"
    - echo "Commit SHA: $CI_COMMIT_SHA"
    - echo "Branch: $CI_COMMIT_BRANCH"
    - echo "Tag: $CI_COMMIT_TAG"
    - echo "Project: $CI_PROJECT_NAME"
    - echo "Runner: $CI_RUNNER_DESCRIPTION"
    - echo "Username: $GITLAB_USER_LOGIN"
```

#### Common Predefined Variables Table

| Variable | Description | Example |
|----------|-------------|---------|
| **Pipeline Variables** | | |
| `CI_PIPELINE_ID` | Unique pipeline ID | `12345` |
| `CI_PIPELINE_IID` | Internal pipeline ID (project scope) | `23` |
| `CI_PIPELINE_SOURCE` | How pipeline was triggered | `push`, `web`, `schedule`, `merge_request_event` |
| `CI_PIPELINE_URL` | Pipeline details URL | `https://gitlab.com/group/project/-/pipelines/12345` |
| `CI_PIPELINE_CREATED_AT` | Pipeline creation timestamp | `2026-03-03T10:00:00Z` |
| **Job Variables** | | |
| `CI_JOB_ID` | Unique job ID | `67890` |
| `CI_JOB_NAME` | Job name | `test-job` |
| `CI_JOB_STAGE` | Job stage | `test` |
| `CI_JOB_URL` | Job details URL | `https://gitlab.com/group/project/-/jobs/67890` |
| `CI_JOB_TOKEN` | Job token for API access | `[secure]` |
| `CI_JOB_STARTED_AT` | Job start timestamp | `2026-03-03T10:05:00Z` |
| **Commit Variables** | | |
| `CI_COMMIT_SHA` | Full commit SHA | `a1b2c3d4e5f6...` |
| `CI_COMMIT_SHORT_SHA` | Short commit SHA (8 chars) | `a1b2c3d4` |
| `CI_COMMIT_BRANCH` | Commit branch name | `main`, `develop` |
| `CI_COMMIT_TAG` | Commit tag (if tagged) | `v1.0.0` |
| `CI_COMMIT_MESSAGE` | Full commit message | `Fix bug in auth` |
| `CI_COMMIT_TITLE` | First line of commit message | `Fix bug in auth` |
| `CI_COMMIT_DESCRIPTION` | Commit message body | `Details...` |
| `CI_COMMIT_REF_NAME` | Branch or tag name | `main` or `v1.0.0` |
| `CI_COMMIT_REF_SLUG` | Lowercased, URL-safe ref | `main`, `feature-branch` |
| `CI_COMMIT_AUTHOR` | Commit author | `John Doe <john@example.com>` |
| `CI_COMMIT_TIMESTAMP` | Commit timestamp | `2026-03-03T09:30:00Z` |
| **Project Variables** | | |
| `CI_PROJECT_ID` | Project ID | `123` |
| `CI_PROJECT_NAME` | Project name | `myproject` |
| `CI_PROJECT_PATH` | Project path with namespace | `group/subgroup/myproject` |
| `CI_PROJECT_PATH_SLUG` | URL-safe project path | `group-subgroup-myproject` |
| `CI_PROJECT_NAMESPACE` | Project namespace | `group/subgroup` |
| `CI_PROJECT_ROOT_NAMESPACE` | Root namespace | `group` |
| `CI_PROJECT_TITLE` | Project title | `My Project` |
| `CI_PROJECT_DIR` | Project directory path | `/builds/group/project` |
| `CI_PROJECT_URL` | Project URL | `https://gitlab.com/group/project` |
| `CI_PROJECT_VISIBILITY` | Project visibility | `private`, `internal`, `public` |
| **Repository Variables** | | |
| `CI_REPOSITORY_URL` | Repository URL | `https://gitlab-ci-token:[secure]@gitlab.com/group/project.git` |
| `CI_DEFAULT_BRANCH` | Default branch | `main` |
| **Runner Variables** | | |
| `CI_RUNNER_ID` | Runner ID | `123` |
| `CI_RUNNER_DESCRIPTION` | Runner description | `docker-runner-1` |
| `CI_RUNNER_TAGS` | Runner tags (comma-separated) | `docker,linux` |
| `CI_RUNNER_EXECUTABLE_ARCH` | Runner architecture | `linux/amd64` |
| **User Variables** | | |
| `GITLAB_USER_ID` | User ID who triggered | `42` |
| `GITLAB_USER_LOGIN` | Username | `johndoe` |
| `GITLAB_USER_NAME` | Full name | `John Doe` |
| `GITLAB_USER_EMAIL` | User email | `john@example.com` |
| **Merge Request Variables** | | |
| `CI_MERGE_REQUEST_ID` | MR internal ID | `5` |
| `CI_MERGE_REQUEST_IID` | MR project-level ID | `5` |
| `CI_MERGE_REQUEST_TITLE` | MR title | `Fix authentication bug` |
| `CI_MERGE_REQUEST_SOURCE_BRANCH_NAME` | Source branch | `feature-auth` |
| `CI_MERGE_REQUEST_TARGET_BRANCH_NAME` | Target branch | `main` |
| `CI_MERGE_REQUEST_DIFF_BASE_SHA` | Base commit SHA | `abc123...` |
| `CI_MERGE_REQUEST_LABELS` | MR labels (comma-separated) | `bug,security` |
| `CI_MERGE_REQUEST_APPROVED` | MR approval status | `true`, `false` |
| **Registry Variables** | | |
| `CI_REGISTRY` | GitLab Container Registry | `registry.gitlab.com` |
| `CI_REGISTRY_IMAGE` | Full registry image path | `registry.gitlab.com/group/project` |
| `CI_REGISTRY_USER` | Registry username | `gitlab-ci-token` |
| `CI_REGISTRY_PASSWORD` | Registry password | `[secure]` |
| **Environment Variables** | | |
| `CI_ENVIRONMENT_NAME` | Environment name | `production` |
| `CI_ENVIRONMENT_SLUG` | URL-safe environment name | `production` |
| `CI_ENVIRONMENT_URL` | Environment URL | `https://example.com` |
| **Kubernetes Variables** | | |
| `KUBE_NAMESPACE` | Kubernetes namespace | `production` |
| `KUBECONFIG` | Kubernetes config file path | `/path/to/kubeconfig` |
| **Schedule Variables** | | |
| `CI_PIPELINE_SCHEDULE` | Pipeline is scheduled | `true`, `false` |
| `CI_SCHEDULE_ID` | Schedule ID | `789` |
| **Git Variables** | | |
| `CI_COMMIT_BEFORE_SHA` | Previous commit SHA | `def456...` |
| `CI_COMMIT_REF_PROTECTED` | Ref is protected | `true`, `false` |
| **Boolean Flags** | | |
| `CI` | Always true in CI | `true` |
| `GITLAB_CI` | Always true in GitLab CI | `true` |
| `CI_SERVER` | Always true | `true` |
| `CI_DEBUG_TRACE` | Enable debug output | `true`, `false` |
| `CI_DISPOSABLE_ENVIRONMENT` | Disposable environment | `true`, `false` |
| **API Variables** | | |
| `CI_API_V4_URL` | GitLab API v4 URL | `https://gitlab.com/api/v4` |
| `CI_SERVER_URL` | GitLab instance URL | `https://gitlab.com` |
| `CI_SERVER_HOST` | GitLab hostname | `gitlab.com` |
| `CI_SERVER_NAME` | GitLab server name | `GitLab` |
| `CI_SERVER_VERSION` | GitLab version | `15.8.0` |
| **Parallel Job Variables** | | |
| `CI_NODE_INDEX` | Current parallel job index | `1`, `2`, `3` (1-based) |
| `CI_NODE_TOTAL` | Total parallel jobs | `5` |
| **Pages Variables** | | |
| `CI_PAGES_DOMAIN` | Pages domain | `gitlab.io` |
| `CI_PAGES_URL` | Pages URL | `https://group.gitlab.io/project` |
| **Dependency Proxy** | | |
| `CI_DEPENDENCY_PROXY_SERVER` | Proxy server | `gitlab.com:443` |
| `CI_DEPENDENCY_PROXY_GROUP_IMAGE_PREFIX` | Proxy prefix | `gitlab.com/group/dependency_proxy/containers` |

**Notes:**
- Variables marked `[secure]` contain sensitive data
- Not all variables are available in all contexts (e.g., MR variables only in MR pipelines)
- Use `printenv | sort` to see all available variables in a job
- Some variables are only available in GitLab Premium/Ultimate tiers

### Custom Variables

#### Global Variables
```yaml
variables:
  DEPLOY_ENV: "production"
  APP_VERSION: "1.0.0"
  DOCKER_DRIVER: overlay2

build:
  script:
    - echo "Building version $APP_VERSION"
```

#### Job Variables
```yaml
test-job:
  variables:
    TEST_ENV: "ci"
    DATABASE_URL: "postgres://localhost/test"
  script:
    - echo "Testing in $TEST_ENV"
    - run-tests.sh
```

#### File Variables
```yaml
variables:
  # Create file from variable content
  DATABASE_CONFIG:
    value: |
      host: localhost
      port: 5432
      database: myapp
    description: "Database configuration"
```

### Variable Expansion
```yaml
variables:
  BASE_URL: "https://example.com"
  API_URL: "${BASE_URL}/api"

job:
  script:
    - echo "API URL is $API_URL"
    - echo "Escaped: $$VARIABLE"  # Literal $
```

### Protected Variables
Set in GitLab UI (Settings > CI/CD > Variables):
- Protected: Only available for protected branches/tags
- Masked: Hidden in job logs
- Environment scope: Available only for specific environments

```yaml
deploy:
  script:
    - echo "Deploying with credentials"
    # $SECRET_KEY is set as protected/masked variable
    - deploy.sh --key $SECRET_KEY
```

---

## Artifacts

### Basic Artifacts
```yaml
build:
  script:
    - make build
  artifacts:
    paths:
      - dist/
      - build/output.log
```

### Artifact Options
```yaml
build-with-options:
  script:
    - npm run build
  artifacts:
    # Files to save
    paths:
      - dist/
      - coverage/
    
    # Exclude patterns
    exclude:
      - dist/**/*.map
      - coverage/tmp/*
    
    # Artifact name
    name: "$CI_COMMIT_REF_NAME-build"
    
    # Expiration
    expire_in: 1 week
    
    # When to upload
    when: on_success  # on_failure, always
    
    # Expose as download
    expose_as: "Build Artifacts"
    
    # Make public
    public: false
```

### Artifact Reports
```yaml
test-job:
  script:
    - npm test
  artifacts:
    reports:
      # JUnit test reports
      junit: test-results.xml
      
      # Code coverage
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml
      
      # Performance metrics
      metrics: metrics.txt
      
      # Security scanning
      sast: gl-sast-report.json
      dependency_scanning: gl-dependency-scanning-report.json
```

### Dotenv Artifacts
Pass variables between jobs:
```yaml
build:
  script:
    - echo "BUILD_VERSION=1.2.3" >> build.env
    - echo "BUILD_DATE=$(date)" >> build.env
  artifacts:
    reports:
      dotenv: build.env

deploy:
  script:
    - echo "Deploying version $BUILD_VERSION"
    - echo "Built on $BUILD_DATE"
  needs:
    - build
```

### Download Artifacts
```yaml
test:
  script:
    - make test
  artifacts:
    paths:
      - test-results/

analyze:
  script:
    # Artifacts from 'test' job are automatically available
    - analyze test-results/
  dependencies:
    - test
```

---

## Cache

### Basic Cache
```yaml
cache:
  paths:
    - node_modules/
    - .npm/

build:
  script:
    - npm ci
    - npm run build
```

### Cache Keys
```yaml
# Global cache with branch-specific key
cache:
  key: "$CI_COMMIT_REF_SLUG"
  paths:
    - node_modules/

# Job-specific cache
test-job:
  cache:
    key: "$CI_COMMIT_REF_SLUG-test"
    paths:
      - vendor/
      - .cache/
```

### Cache Policy
```yaml
build:
  cache:
    key: npm-cache
    paths:
      - node_modules/
    policy: pull-push  # Download and upload (default)
  script:
    - npm ci

test:
  cache:
    key: npm-cache
    paths:
      - node_modules/
    policy: pull  # Only download, don't upload
  script:
    - npm test
```

### Multiple Caches
```yaml
test:
  cache:
    - key: npm-cache
      paths:
        - node_modules/
    
    - key: cypress-cache
      paths:
        - .cache/Cypress/
  script:
    - npm test
    - npm run cypress
```

### Cache with Files
```yaml
# Update cache when files change
cache:
  key:
    files:
      - package-lock.json
      - Gemfile.lock
    prefix: $CI_COMMIT_REF_SLUG
  paths:
    - node_modules/
    - vendor/ruby
```

---

## Dependencies

### Job Dependencies with needs
```yaml
build:
  stage: build
  script:
    - make build
  artifacts:
    paths:
      - build/

test:
  stage: test
  needs: [build]  # Wait for build, skip other jobs
  script:
    - make test

deploy:
  stage: deploy
  needs:
    - job: build
      artifacts: true  # Download artifacts (default)
    - job: test
      artifacts: false  # Don't download artifacts
  script:
    - make deploy
```

### Parallel Job Dependencies
```yaml
stages:
  - build
  - test
  - deploy

build-a:
  stage: build
  script:
    - build-component-a.sh
  artifacts:
    paths:
      - dist-a/

build-b:
  stage: build
  script:
    - build-component-b.sh
  artifacts:
    paths:
      - dist-b/

test-integration:
  stage: test
  needs:
    - build-a
    - build-b
  script:
    - test-integration.sh
```

### Cross-Pipeline Dependencies
```yaml
deploy:
  needs:
    - pipeline: $PARENT_PIPELINE_ID
      job: build
  script:
    - deploy.sh
```

---

## Rules and Conditions

### rules vs only/except
`rules` is the modern way (prefer over `only`/`except`):

```yaml
# Modern approach with rules
job-with-rules:
  script:
    - echo "Hello"
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'

# Legacy approach (avoid)
job-with-only:
  script:
    - echo "Hello"
  only:
    - merge_requests
    - main
```

### Rule Types

#### if Conditions
```yaml
deploy-prod:
  script:
    - deploy.sh
  rules:
    # Run on main branch only
    - if: '$CI_COMMIT_BRANCH == "main"'
    
    # Run on tags matching pattern
    - if: '$CI_COMMIT_TAG =~ /^v\d+\.\d+\.\d+$/'
    
    # Multiple conditions with AND
    - if: '$CI_COMMIT_BRANCH == "main" && $CI_PIPELINE_SOURCE == "push"'
    
    # Check variable existence
    - if: '$DEPLOY_ENABLED'
```

#### changes
Run job when specific files change:
```yaml
test-frontend:
  script:
    - npm test
  rules:
    - changes:
        - "src/**/*.js"
        - "src/**/*.vue"
        - package.json

test-backend:
  script:
    - go test
  rules:
    - changes:
        - "**/*.go"
        - go.mod
```

#### exists
Run job if files exist:
```yaml
deploy-kubernetes:
  script:
    - kubectl apply -f k8s/
  rules:
    - exists:
        - k8s/*.yaml
```

### Rule Clauses

```yaml
complex-rules:
  script:
    - run.sh
  rules:
    # Run on MR, allow manual trigger
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      when: manual
      allow_failure: true
    
    # Run on main, auto-deploy
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: on_success
      variables:
        DEPLOY_ENV: "production"
    
    # Otherwise, don't run
    - when: never
```

### Common Rule Patterns

```yaml
# Run on MR or default branch
.default-rules:
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'

# Deploy only on protected branches
deploy:
  extends: .default-rules
  script:
    - deploy.sh
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_COMMIT_TAG'

# Manual deployment
manual-deploy:
  script:
    - deploy.sh
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
```

---

## Docker Integration

### Using Docker Images
```yaml
# Default image for all jobs
default:
  image: rockylinux:9

# Job-specific image
node-job:
  image: node:18-alpine
  script:
    - npm --version

# Image with registry
private-job:
  image: registry.example.com/myapp:latest
  script:
    - ./run.sh

# Image with authentication
authenticated-job:
  image:
    name: private.registry.com/image:tag
    entrypoint: [""]
  before_script:
    - echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USER" --password-stdin
  script:
    - run-app
```

### Docker-in-Docker (dind)
```yaml
build-docker:
  image: docker:24
  services:
    - docker:24-dind
  variables:
    DOCKER_HOST: tcp://docker:2376
    DOCKER_TLS_CERTDIR: "/certs"
    DOCKER_TLS_VERIFY: 1
    DOCKER_CERT_PATH: "$DOCKER_TLS_CERTDIR/client"
  script:
    - docker info
    - docker build -t myapp:$CI_COMMIT_SHA .
    - docker push myapp:$CI_COMMIT_SHA
```

### Building Docker Images
```yaml
variables:
  DOCKER_REGISTRY: registry.example.com
  IMAGE_TAG: $DOCKER_REGISTRY/$CI_PROJECT_PATH:$CI_COMMIT_SHA

build-image:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $DOCKER_REGISTRY
  script:
    # Build image
    - docker build -t $IMAGE_TAG .
    
    # Tag as latest for main branch
    - |
      if [ "$CI_COMMIT_BRANCH" == "main" ]; then
        docker tag $IMAGE_TAG $DOCKER_REGISTRY/$CI_PROJECT_PATH:latest
        docker push $DOCKER_REGISTRY/$CI_PROJECT_PATH:latest
      fi
    
    # Push image
    - docker push $IMAGE_TAG
```

### Kaniko (rootless builds)
```yaml
build-kaniko:
  stage: build
  image:
    name: gcr.io/kaniko-project/executor:debug
    entrypoint: [""]
  script:
    - mkdir -p /kaniko/.docker
    - echo "{\"auths\":{\"$CI_REGISTRY\":{\"username\":\"$CI_REGISTRY_USER\",\"password\":\"$CI_REGISTRY_PASSWORD\"}}}" > /kaniko/.docker/config.json
    - /kaniko/executor
      --context $CI_PROJECT_DIR
      --dockerfile $CI_PROJECT_DIR/Containerfile
      --destination $CI_REGISTRY_IMAGE:$CI_COMMIT_TAG
```

---

## Services

### Service Containers
```yaml
test-with-postgres:
  image: node:18
  services:
    - postgres:15
  variables:
    POSTGRES_DB: testdb
    POSTGRES_USER: testuser
    POSTGRES_PASSWORD: testpass
    DATABASE_URL: "postgresql://testuser:testpass@postgres:5432/testdb"
  script:
    - npm test

test-with-redis:
  image: python:3.11
  services:
    - redis:7-alpine
  variables:
    REDIS_URL: "redis://redis:6379"
  script:
    - pytest
```

### Multiple Services
```yaml
integration-test:
  image: node:18
  services:
    - name: postgres:15
      alias: database
    - name: redis:7
      alias: cache
    - name: selenium/standalone-chrome:latest
      alias: chrome
  variables:
    DATABASE_URL: "postgresql://postgres:postgres@database:5432/test"
    REDIS_URL: "redis://cache:6379"
    SELENIUM_URL: "http://chrome:4444/wd/hub"
  script:
    - npm run integration-tests
```

### Service Configuration
```yaml
test-mysql:
  services:
    - name: mysql:8
      command:
        - --default-authentication-plugin=mysql_native_password
        - --character-set-server=utf8mb4
      variables:
        MYSQL_ROOT_PASSWORD: root
        MYSQL_DATABASE: testdb
```

---

## Environments

### Basic Environment
```yaml
deploy-staging:
  stage: deploy
  script:
    - deploy.sh staging
  environment:
    name: staging
    url: https://staging.example.com

deploy-production:
  stage: deploy
  script:
    - deploy.sh production
  environment:
    name: production
    url: https://example.com
  when: manual
```

### Dynamic Environments
```yaml
deploy-review:
  stage: deploy
  script:
    - deploy.sh "review-$CI_COMMIT_REF_SLUG"
  environment:
    name: review/$CI_COMMIT_REF_NAME
    url: https://$CI_COMMIT_REF_SLUG.example.com
    on_stop: stop-review
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'

stop-review:
  stage: deploy
  script:
    - cleanup.sh "review-$CI_COMMIT_REF_SLUG"
  environment:
    name: review/$CI_COMMIT_REF_NAME
    action: stop
  when: manual
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
```

### Environment with Deployment
```yaml
deploy-production:
  stage: deploy
  script:
    - kubectl apply -f k8s/
  environment:
    name: production
    url: https://example.com
    kubernetes:
      namespace: production
    deployment_tier: production
  resource_group: production
```

---

## Triggers and Pipelines

### Downstream Pipelines
```yaml
trigger-downstream:
  stage: deploy
  trigger:
    project: group/downstream-project
    branch: main
    strategy: depend  # Wait for completion
  variables:
    UPSTREAM_COMMIT: $CI_COMMIT_SHA
```

### Parent-Child Pipelines
```yaml
# .gitlab-ci.yml
trigger-child:
  trigger:
    include: .gitlab-ci-child.yml
    strategy: depend
  variables:
    PARENT_VAR: "value"

# .gitlab-ci-child.yml
child-job:
  script:
    - echo "Parent var: $PARENT_VAR"
```

### Multi-Project Pipelines
```yaml
trigger-api:
  stage: deploy
  trigger:
    project: backend/api
  variables:
    DEPLOY_VERSION: $CI_COMMIT_TAG

trigger-frontend:
  stage: deploy
  trigger:
    project: frontend/app
  variables:
    API_VERSION: $CI_COMMIT_TAG
```

### Pipeline Schedules
Set in GitLab UI, use in rules:
```yaml
nightly-build:
  script:
    - make full-build
  rules:
    - if: '$CI_PIPELINE_SOURCE == "schedule"'

scheduled-cleanup:
  script:
    - cleanup-old-data.sh
  rules:
    - if: '$CI_PIPELINE_SOURCE == "schedule" && $SCHEDULE_TYPE == "cleanup"'
```

---

## Templates and Includes

### Local Includes
```yaml
# .gitlab-ci.yml
include:
  - local: '/templates/build.yml'
  - local: '/templates/test.yml'
  - local: '/templates/deploy.yml'

# templates/build.yml
.build-template:
  image: node:18
  script:
    - npm ci
    - npm run build
  artifacts:
    paths:
      - dist/
```

### Remote Includes
```yaml
include:
  - remote: 'https://gitlab.com/gitlab-org/gitlab/-/raw/master/lib/gitlab/ci/templates/Jobs/SAST.gitlab-ci.yml'
  - remote: 'https://example.com/ci-templates/docker.yml'
```

### Template Includes
```yaml
include:
  # Security scanning
  - template: Security/SAST.gitlab-ci.yml
  - template: Security/Dependency-Scanning.gitlab-ci.yml
  - template: Security/Container-Scanning.gitlab-ci.yml
  - template: Security/License-Scanning.gitlab-ci.yml
  
  # Code quality
  - template: Code-Quality.gitlab-ci.yml
  
  # Infrastructure as Code
  - template: Terraform.gitlab-ci.yml
```

### Project Includes
```yaml
include:
  - project: 'devops/ci-templates'
    ref: main
    file:
      - '/templates/build.yml'
      - '/templates/deploy.yml'
```

### Extends (Job Templates)
```yaml
.default-job:
  image: rockylinux:9
  before_script:
    - echo "Setting up environment"
  tags:
    - docker
  retry: 2

.node-job:
  extends: .default-job
  image: node:18
  cache:
    paths:
      - node_modules/

build-frontend:
  extends: .node-job
  script:
    - npm ci
    - npm run build

test-frontend:
  extends: .node-job
  script:
    - npm ci
    - npm test
```

### YAML Anchors
```yaml
.docker-config: &docker-config
  image: docker:24
  services:
    - docker:24-dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD

build-api:
  <<: *docker-config
  script:
    - docker build -t api .

build-frontend:
  <<: *docker-config
  script:
    - docker build -t frontend .
```

---

## Advanced Features

### Parallel Jobs
```yaml
# Matrix strategy
test-matrix:
  parallel:
    matrix:
      - RUBY_VERSION: ["3.0", "3.1", "3.2"]
        DATABASE: ["postgres", "mysql"]
  script:
    - bundle install
    - rake test
  image: ruby:$RUBY_VERSION

# Parallel instances
test-split:
  parallel: 5
  script:
    - echo "Running test suite $CI_NODE_INDEX of $CI_NODE_TOTAL"
    - npm test -- --shard=$CI_NODE_INDEX/$CI_NODE_TOTAL
```

### Resource Groups
Prevent parallel execution:
```yaml
deploy-staging:
  stage: deploy
  script:
    - deploy.sh staging
  resource_group: staging-deploy

deploy-production:
  stage: deploy
  script:
    - deploy.sh production
  resource_group: production-deploy
```

### Release Management
```yaml
release-job:
  stage: deploy
  image: registry.gitlab.com/gitlab-org/release-cli:latest
  rules:
    - if: '$CI_COMMIT_TAG'
  script:
    - echo "Creating release"
  release:
    tag_name: '$CI_COMMIT_TAG'
    name: 'Release $CI_COMMIT_TAG'
    description: './CHANGELOG.md'
    assets:
      links:
        - name: 'Binary'
          url: 'https://releases.example.com/app-$CI_COMMIT_TAG.tar.gz'
```

### Pages Deployment
```yaml
pages:
  stage: deploy
  script:
    - npm run build
    - mv dist public
  artifacts:
    paths:
      - public
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
```

### DAG (Directed Acyclic Graph)
```yaml
stages:
  - build
  - test
  - deploy

build-a:
  stage: build
  script:
    - build-component-a

build-b:
  stage: build
  script:
    - build-component-b

test-a:
  stage: test
  needs: [build-a]
  script:
    - test-component-a

test-b:
  stage: test
  needs: [build-b]
  script:
    - test-component-b

deploy:
  stage: deploy
  needs: [test-a, test-b]
  script:
    - deploy-app
```

---

## Real-World Examples

### Node.js Application
```yaml
image: node:18

stages:
  - install
  - lint
  - test
  - build
  - deploy

variables:
  npm_config_cache: "$CI_PROJECT_DIR/.npm"

cache:
  key:
    files:
      - package-lock.json
  paths:
    - .npm/
    - node_modules/

install-dependencies:
  stage: install
  script:
    - npm ci
  artifacts:
    paths:
      - node_modules/
    expire_in: 1 day

lint:
  stage: lint
  needs: [install-dependencies]
  script:
    - npm run lint

test:
  stage: test
  needs: [install-dependencies]
  coverage: '/All files[^|]*\|[^|]*\s+([\d\.]+)/'
  script:
    - npm test
  artifacts:
    reports:
      junit: junit.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml

build:
  stage: build
  needs: [install-dependencies]
  script:
    - npm run build
  artifacts:
    paths:
      - dist/
    expire_in: 1 week

deploy-staging:
  stage: deploy
  needs: [build, test]
  script:
    - npm run deploy -- --env=staging
  environment:
    name: staging
    url: https://staging.example.com
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'

deploy-production:
  stage: deploy
  needs: [build, test]
  script:
    - npm run deploy -- --env=production
  environment:
    name: production
    url: https://example.com
  rules:
    - if: '$CI_COMMIT_TAG'
  when: manual
```

### Python Application with Docker
```yaml
stages:
  - test
  - build
  - deploy

variables:
  PIP_CACHE_DIR: "$CI_PROJECT_DIR/.cache/pip"
  DOCKER_REGISTRY: registry.example.com
  IMAGE_NAME: $DOCKER_REGISTRY/$CI_PROJECT_PATH

cache:
  paths:
    - .cache/pip

.python-base:
  image: python:3.11
  before_script:
    - pip install --upgrade pip
    - pip install -r requirements.txt

test:
  extends: .python-base
  stage: test
  services:
    - postgres:15
  variables:
    POSTGRES_DB: testdb
    POSTGRES_USER: testuser
    POSTGRES_PASSWORD: testpass
    DATABASE_URL: "postgresql://testuser:testpass@postgres:5432/testdb"
  script:
    - pytest --cov=app --cov-report=xml --cov-report=term
    - flake8 app/
    - black --check app/
  coverage: '/TOTAL.*\s+(\d+%)$/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml

build-image:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $DOCKER_REGISTRY
  script:
    - docker build -t $IMAGE_NAME:$CI_COMMIT_SHA .
    - docker tag $IMAGE_NAME:$CI_COMMIT_SHA $IMAGE_NAME:latest
    - docker push $IMAGE_NAME:$CI_COMMIT_SHA
    - docker push $IMAGE_NAME:latest
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'

deploy-kubernetes:
  stage: deploy
  image: bitnami/kubectl:latest
  needs: [build-image]
  script:
    - kubectl config use-context $KUBE_CONTEXT
    - kubectl set image deployment/myapp app=$IMAGE_NAME:$CI_COMMIT_SHA -n production
    - kubectl rollout status deployment/myapp -n production
  environment:
    name: production
    url: https://api.example.com
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  when: manual
```

### Go Application with Multi-Arch Builds
```yaml
stages:
  - test
  - build
  - release

variables:
  PACKAGE: "github.com/example/myapp"
  GO_VERSION: "1.21"

.go-base:
  image: golang:${GO_VERSION}
  before_script:
    - go version
    - mkdir -p $GOPATH/src/$(dirname $PACKAGE)
    - ln -svf $CI_PROJECT_DIR $GOPATH/src/$PACKAGE
    - cd $GOPATH/src/$PACKAGE

test:
  extends: .go-base
  stage: test
  script:
    - go fmt $(go list ./... | grep -v /vendor/)
    - go vet $(go list ./... | grep -v /vendor/)
    - go test -race -coverprofile=coverage.txt -covermode=atomic ./...
  coverage: '/coverage: \d+.\d+% of statements/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.txt

build:
  extends: .go-base
  stage: build
  parallel:
    matrix:
      - GOOS: [linux, darwin, windows]
        GOARCH: [amd64, arm64]
  script:
    - |
      BINARY="myapp-${GOOS}-${GOARCH}"
      if [ "$GOOS" = "windows" ]; then
        BINARY="${BINARY}.exe"
      fi
    - go build -ldflags "-X main.version=$CI_COMMIT_TAG" -o $BINARY
  artifacts:
    paths:
      - myapp-*
    expire_in: 1 week
  rules:
    - if: '$CI_COMMIT_TAG'

release:
  stage: release
  image: registry.gitlab.com/gitlab-org/release-cli:latest
  needs:
    - job: build
      artifacts: true
  script:
    - echo "Creating release $CI_COMMIT_TAG"
  release:
    tag_name: '$CI_COMMIT_TAG'
    description: './CHANGELOG.md'
    assets:
      links:
        - name: 'Linux AMD64'
          url: "${CI_PROJECT_URL}/-/jobs/${CI_JOB_ID}/artifacts/raw/myapp-linux-amd64"
        - name: 'Linux ARM64'
          url: "${CI_PROJECT_URL}/-/jobs/${CI_JOB_ID}/artifacts/raw/myapp-linux-arm64"
        - name: 'macOS AMD64'
          url: "${CI_PROJECT_URL}/-/jobs/${CI_JOB_ID}/artifacts/raw/myapp-darwin-amd64"
        - name: 'macOS ARM64'
          url: "${CI_PROJECT_URL}/-/jobs/${CI_JOB_ID}/artifacts/raw/myapp-darwin-arm64"
        - name: 'Windows AMD64'
          url: "${CI_PROJECT_URL}/-/jobs/${CI_JOB_ID}/artifacts/raw/myapp-windows-amd64.exe"
  rules:
    - if: '$CI_COMMIT_TAG'
```

### Microservices Monorepo
```yaml
workflow:
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'

stages:
  - test
  - build
  - deploy

variables:
  DOCKER_REGISTRY: registry.example.com

# Test only changed services
test-api:
  stage: test
  image: node:18
  script:
    - cd services/api
    - npm ci
    - npm test
  rules:
    - changes:
        - services/api/**/*

test-frontend:
  stage: test
  image: node:18
  script:
    - cd services/frontend
    - npm ci
    - npm test
  rules:
    - changes:
        - services/frontend/**/*

test-worker:
  stage: test
  image: python:3.11
  script:
    - cd services/worker
    - pip install -r requirements.txt
    - pytest
  rules:
    - changes:
        - services/worker/**/*

# Build changed services
.build-template:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $DOCKER_REGISTRY
  script:
    - cd services/$SERVICE_NAME
    - docker build -t $DOCKER_REGISTRY/$CI_PROJECT_PATH/$SERVICE_NAME:$CI_COMMIT_SHA .
    - docker push $DOCKER_REGISTRY/$CI_PROJECT_PATH/$SERVICE_NAME:$CI_COMMIT_SHA

build-api:
  extends: .build-template
  variables:
    SERVICE_NAME: api
  rules:
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
      changes:
        - services/api/**/*

build-frontend:
  extends: .build-template
  variables:
    SERVICE_NAME: frontend
  rules:
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
      changes:
        - services/frontend/**/*

build-worker:
  extends: .build-template
  variables:
    SERVICE_NAME: worker
  rules:
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
      changes:
        - services/worker/**/*

# Deploy all services
deploy-staging:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl config use-context staging
    - |
      for service in api frontend worker; do
        if [ -f "services/$service/k8s/deployment.yaml" ]; then
          kubectl apply -f services/$service/k8s/ -n staging
        fi
      done
  environment:
    name: staging
    url: https://staging.example.com
  rules:
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
```

---

## Best Practices

### Pipeline Efficiency

1. **Use Caching Effectively**
```yaml
cache:
  key:
    files:
      - package-lock.json
  paths:
    - node_modules/
  policy: pull-push
```

2. **Optimize with DAG**
```yaml
# Use needs to skip waiting for entire stage
deploy:
  needs: [build, test]
  script:
    - deploy.sh
```

3. **Minimize Artifacts**
```yaml
artifacts:
  paths:
    - dist/
  expire_in: 1 day
  exclude:
    - dist/**/*.map
```

### Security Best Practices

1. **Protected Variables**
- Store secrets in GitLab CI/CD variables (masked and protected)
- Never hardcode credentials

2. **Image Pinning**
```yaml
# Good: Specific version
image: node:18.19.0-alpine

# Bad: Moving tag
image: node:latest
```

3. **Least Privilege**
```yaml
# Run as non-root user
test:
  image: node:18
  script:
    - |
      groupadd -g 1000 appuser
      useradd -r -u 1000 -g appuser appuser
      su appuser -c "npm test"
```

### Code Organization

1. **Use Templates**
```yaml
.deploy-template:
  script:
    - deploy.sh $ENVIRONMENT
  rules:
    - if: '$CI_COMMIT_BRANCH == $DEPLOY_BRANCH'

deploy-staging:
  extends: .deploy-template
  variables:
    ENVIRONMENT: staging
    DEPLOY_BRANCH: develop

deploy-production:
  extends: .deploy-template
  variables:
    ENVIRONMENT: production
    DEPLOY_BRANCH: main
```

2. **Separate Configuration Files**
```yaml
include:
  - local: '.gitlab/ci/test.yml'
  - local: '.gitlab/ci/build.yml'
  - local: '.gitlab/ci/deploy.yml'
```

3. **Use Workflow Rules**
```yaml
workflow:
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
    - when: never
```

### Performance Tips

1. **Parallel Execution**
```yaml
test:
  parallel: 5
  script:
    - npm test -- --shard=$CI_NODE_INDEX/$CI_NODE_TOTAL
```

2. **Selective Execution**
```yaml
test-backend:
  rules:
    - changes:
        - "backend/**/*"
```

3. **Interruptible Jobs**
```yaml
test:
  interruptible: true
  script:
    - npm test
```

### Documentation

1. **Comment Complex Logic**
```yaml
# Build multi-arch images for production release
# Runs only on version tags (e.g., v1.2.3)
build-multi-arch:
  rules:
    - if: '$CI_COMMIT_TAG =~ /^v\d+\.\d+\.\d+$/'
```

2. **Use Descriptive Names**
```yaml
# Good
deploy-production-kubernetes:
  script:
    - kubectl apply -f k8s/

# Bad
deploy1:
  script:
    - kubectl apply -f k8s/
```

---

## Troubleshooting

### Common Issues

#### Job Stuck in Pending
```yaml
# Check runner tags match
test-job:
  tags:
    - docker  # Ensure runner has this tag
  script:
    - make test
```

#### Cache Not Working
```yaml
# Ensure cache key is consistent
cache:
  key: "$CI_COMMIT_REF_SLUG-$CI_JOB_NAME"
  paths:
    - node_modules/
  policy: pull-push  # Ensure correct policy
```

#### Artifacts Not Available
```yaml
deploy:
  needs:
    - job: build
      artifacts: true  # Explicitly enable
  script:
    - ls dist/  # Should see build artifacts
```

#### Variable Not Expanded
```yaml
# Use double quotes for variable expansion
script:
  - echo "Branch: $CI_COMMIT_BRANCH"  # Works
  - echo 'Branch: $CI_COMMIT_BRANCH'  # Doesn't work
```

### Debugging Techniques

#### Enable Debug Logging
Set variable in GitLab UI or pipeline:
```yaml
variables:
  CI_DEBUG_TRACE: "true"
```

#### Print Environment
```yaml
debug-job:
  script:
    - env | sort
    - printenv
    - echo "Job ID: $CI_JOB_ID"
    - echo "Pipeline ID: $CI_PIPELINE_ID"
```

#### Validate YAML
```bash
# Local validation
gitlab-ci-lint < .gitlab-ci.yml

# Or use GitLab CI Lint page
# Project > CI/CD > Pipelines > CI Lint
```

#### Test Jobs Locally
```bash
# Using gitlab-runner
gitlab-runner exec docker test-job

# Or using Docker
docker run -v $PWD:/app -w /app node:18 sh -c "npm test"
```

### Error Messages

#### `yaml invalid`
- Check indentation (use 2 spaces, not tabs)
- Validate YAML syntax
- Check for missing colons or brackets

#### `This job is stuck`
- No runner available with matching tags
- All runners are busy
- Runner not configured for project

#### `Job failed: exit code 1`
- Script command failed
- Check script output for actual error
- Add error handling to scripts

#### `Insufficient permissions`
- Check runner executor permissions
- Verify Git access for cloning
- Check container image permissions

### Performance Issues

#### Slow Pipeline
```yaml
# Use DAG to parallelize
stages:
  - build
  - test

test-frontend:
  stage: test
  needs: [build-frontend]  # Don't wait for build-backend

test-backend:
  stage: test
  needs: [build-backend]
```

#### Large Artifacts
```yaml
# Reduce artifact size
artifacts:
  paths:
    - dist/
  exclude:
    - "**/*.map"
    - "**/*.log"
  expire_in: 1 day
```

#### Cache Misses
```yaml
# Use file-based cache key
cache:
  key:
    files:
      - package-lock.json  # Update when deps change
  paths:
    - node_modules/
```

### Getting Help

1. **Check Job Logs**: Click on failed job to see detailed output
2. **Use CI Lint**: Validate syntax before commit
3. **Check Runner Logs**: `gitlab-runner --debug run` on runner host
4. **Review Documentation**: https://docs.gitlab.com/ee/ci/
5. **Ask Community**: GitLab Forum or Stack Overflow

---

## Quick Reference

### Common Variables
```yaml
$CI_COMMIT_SHA          # Commit hash
$CI_COMMIT_BRANCH       # Branch name
$CI_COMMIT_TAG          # Tag name
$CI_PIPELINE_ID         # Pipeline ID
$CI_PROJECT_NAME        # Project name
$CI_PROJECT_PATH        # group/project
$CI_REGISTRY            # GitLab Container Registry
$CI_REGISTRY_USER       # Registry username
$CI_REGISTRY_PASSWORD   # Registry password
$CI_DEFAULT_BRANCH      # Default branch (usually main)
$CI_MERGE_REQUEST_ID    # MR number
$GITLAB_USER_LOGIN      # Username who triggered pipeline
```

### Pipeline Sources
```yaml
$CI_PIPELINE_SOURCE == "push"
$CI_PIPELINE_SOURCE == "web"
$CI_PIPELINE_SOURCE == "trigger"
$CI_PIPELINE_SOURCE == "schedule"
$CI_PIPELINE_SOURCE == "api"
$CI_PIPELINE_SOURCE == "merge_request_event"
```

### Job States
- `created`: Job created
- `pending`: Waiting for runner
- `running`: Job executing
- `success`: Job succeeded
- `failed`: Job failed
- `canceled`: Job canceled
- `skipped`: Job skipped
- `manual`: Waiting for manual action

---

## Additional Resources

### Official Documentation
- GitLab CI/CD: https://docs.gitlab.com/ee/ci/
- YAML Reference: https://docs.gitlab.com/ee/ci/yaml/
- Examples: https://docs.gitlab.com/ee/ci/examples/

### Templates Repository
- https://gitlab.com/gitlab-org/gitlab/-/tree/master/lib/gitlab/ci/templates

### Tools
- GitLab Runner: https://docs.gitlab.com/runner/
- CI Lint: Project > CI/CD > Pipelines > CI Lint
- Pipeline Editor: Project > CI/CD > Editor

### Community
- GitLab Forum: https://forum.gitlab.com/
- Discord: https://discord.gg/gitlab
- Stack Overflow: Tag `gitlab-ci`
