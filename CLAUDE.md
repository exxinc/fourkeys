# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Four Keys is a Google DORA (DevOps Research and Assessment) project that measures software delivery performance through four key metrics:
- Deployment Frequency
- Lead Time for Changes
- Time to Restore Services
- Change Failure Rate

The system collects events from development environments (GitHub, GitLab, CircleCI, etc.) and displays metrics in a Grafana dashboard.

## Architecture

The system follows an event-driven architecture on Google Cloud Platform:

1. **event-handler/** - Flask service that receives webhooks from various sources, validates them, and publishes to Pub/Sub
2. **bq-workers/** - Collection of parser services (GitHub, GitLab, CircleCI, ArgoCD, PagerDuty, Cloud Build, Tekton) that subscribe to Pub/Sub topics, transform event data, and insert into BigQuery
3. **shared/** - Shared Python module used by bq-workers for BigQuery insertion logic
4. **queries/** - SQL queries for creating BigQuery derived tables (changes, deployments, incidents)
5. **dashboard/** - Grafana dashboard configuration for visualizing the four key metrics
6. **data-generator/** - Python script for generating mock GitHub/GitLab events for testing

### Data Flow
```
Webhook → event-handler → Pub/Sub → bq-workers (parser) → BigQuery → Grafana Dashboard
```

### Key Components
- **sources.py** in event-handler defines AUTHORIZED_SOURCES for webhook validation
- Each bq-worker has its own **main.py** with parser logic specific to that event source
- **shared.py** provides common BigQuery insertion functionality used across all parsers

## Development Commands

### Testing
```bash
# Run all tests
python3 -m nox

# List available test sessions
python3 -m nox -l

# Run a specific test session
python3 -m nox -s "{session_name}"
```

### Linting
```bash
# Run flake8 linter on all code
python3 -m nox -s lint
```

### Building Container Images
```bash
# Build event-handler
gcloud builds submit event-handler --config=event-handler/cloudbuild.yaml --project $PROJECT_ID

# Build dashboard
gcloud builds submit dashboard --config=dashboard/cloudbuild.yaml --project $PROJECT_ID

# Build a specific parser (e.g., GitHub)
gcloud builds submit bq-workers --config=bq-workers/parsers.cloudbuild.yaml --project $PROJECT_ID --substitutions=_SERVICE=github
```

### Generating Mock Data
```bash
# Set required environment variables
export WEBHOOK=$(gcloud run services list --project $PROJECT_ID | grep event-handler | awk '{print $4}')
export SECRET=$(gcloud secrets versions access 1 --secret=event-handler --project $PROJECT_ID)

# Generate mock GitHub data
python3 data-generator/generate_data.py --vc_system=github
```

## Important Development Notes

### Git Workflow
**CRITICAL**: Do NOT use "Squash Merging" when merging to trunk. This breaks the link between trunk commits and branch commits, making it impossible to measure "Lead Time for Changes" correctly.

### Python Environment
- Uses Python 3.10
- Dependencies are managed per-service with individual requirements.txt files
- Test dependencies are in requirements-test.txt at the root level

### Adding New Event Sources
1. Add the new source to `AUTHORIZED_SOURCES` in `event-handler/sources.py`
2. Create a new parser service in `bq-workers/` using the `new-source-template/`
3. Implement the parsing logic in the new service's `main.py`
4. Update BigQuery queries in `queries/` to classify events from the new source

### Modifying Event Classification
To reclassify what constitutes a "change", "deployment", or "incident":
1. Edit the SQL files in `queries/` directory for the relevant view
2. Run `terraform apply` from the setup directory to update the BigQuery views

### BigQuery Schema
- **events_raw**: Raw events with source, event_type, id, metadata (JSON), time_created, signature, msg_id
- **changes**: Derived table with change_id, time_created, change_type
- **deployments**: Derived table with deploy_id, changes (array), time_created
- **incidents**: Derived table with incident_id, changes (array), time_created, time_resolved

## Testing Philosophy

- All services have corresponding test files (e.g., `main.py` → `main_test.py`)
- Tests use pytest framework
- nox runs tests across different directories and handles dependency installation
- Flake8 linting enforces Google import order style with specific ignore rules
