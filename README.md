# Secure EHR Insight: Clinical Validator

A privacy-conscious clinical-record retrieval prototype that combines PostgreSQL and pgvector search with a FastAPI service, a Streamlit interface, clinical PII redaction, and NeMo Guardrails.

> **Important:** This repository is a software demonstration, not a medical device. It does not provide diagnoses or treatment recommendations. Do not use its output to make clinical decisions. Use only data you are authorized to process, and never publish patient data, credentials, or private keys.

## How It Works

1. Clinical encounter records are loaded into PostgreSQL.
2. A clinical embedding model creates 768-dimensional vectors for records.
3. The API finds relevant records for the selected patient using pgvector similarity search.
4. Retrieved text passes through Presidio-based PII redaction and NeMo Guardrails before the answer is returned.
5. A Streamlit chat interface sends requests to the FastAPI service.

## Project Layout

```text
scripts/                  Database setup, ingestion, embeddings, and checks
src/api/                  FastAPI application
src/database/             PostgreSQL schema
src/guardrails/           NeMo Guardrails configuration and rules
src/pii_redaction/        Presidio-based clinical text redaction
src/UI/                   Streamlit application
requirements.txt          Python dependencies
```

The clinical dataset is not included. The ingestion script expects an authorized local file at `data/MIMIC_IV_Trasncript.csv`.

## Prerequisites

- Python 3.12
- PostgreSQL 14 with the pgvector extension
- An authorized copy of the dataset at the path above
- A compatible LLM API key for the provider configured in `src/guardrails/config.yml`
- `uv` is recommended for installing Python packages

The embedding model is downloaded the first time embeddings are generated. Model files and database contents are not part of this repository.

## 1. Set Up PostgreSQL

The commands below target Ubuntu 24.04. Install PostgreSQL 14 from the PostgreSQL APT repository and install pgvector:

```bash
sudo apt update
sudo apt install -y curl ca-certificates
sudo install -d /usr/share/postgresql-common/pgdg
sudo curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc --fail \
	https://www.postgresql.org/media/keys/ACCC4CF8.asc
echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] https://apt.postgresql.org/pub/repos/apt $(. /etc/os-release && echo "$VERSION_CODENAME")-pgdg main" \
	| sudo tee /etc/apt/sources.list.d/pgdg.list
sudo apt update
sudo apt install -y postgresql-14 postgresql-contrib-14 postgresql-14-pgvector
```

Create a database and a dedicated user. Choose your own strong password; do not reuse or commit credentials from local setup notes.

```bash
sudo -u postgres psql
```

At the `psql` prompt:

```sql
CREATE DATABASE ehr_db;
CREATE USER fde_admin WITH PASSWORD 'REPLACE_WITH_A_STRONG_PASSWORD';
ALTER ROLE fde_admin SET client_encoding TO 'utf8';
ALTER ROLE fde_admin SET default_transaction_isolation TO 'read committed';
ALTER ROLE fde_admin SET timezone TO 'UTC';
GRANT ALL PRIVILEGES ON DATABASE ehr_db TO fde_admin;
\connect ehr_db
GRANT ALL ON SCHEMA public TO fde_admin;
\quit
```

Enable pgvector in the database:

```bash
sudo -u postgres psql -d ehr_db -c "CREATE EXTENSION IF NOT EXISTS vector;"
```

If connecting to PostgreSQL from another machine, configure PostgreSQL and its firewall for your specific trusted client IPs. Avoid opening database access to `0.0.0.0/0`.

## 2. Configure the Environment

Create a `.env` file in the project root. Keep it local and use your own values:

```env
DB_HOST=localhost
DB_PORT=5432
DB_NAME=ehr_db
DB_USER=fde_admin
DB_PASSWORD=REPLACE_WITH_YOUR_DATABASE_PASSWORD

OPENAI_API_KEY=REPLACE_WITH_YOUR_PROVIDER_API_KEY
```

The configured Guardrails model uses an OpenAI-compatible API. Set the API key expected by your provider and configuration. Never commit `.env` or paste real credentials into documentation, issues, or logs.

## 3. Install Dependencies

From the repository root, create and activate a virtual environment, then install the pinned dependencies:

```bash
uv venv
```

On Linux or macOS:

```bash
source .venv/bin/activate
uv pip install -r requirements.txt
```

On Windows PowerShell:

```powershell
.venv\Scripts\Activate.ps1
uv pip install -r requirements.txt
```

## 4. Load and Prepare the Data

Place the authorized dataset at `data/MIMIC_IV_Trasncript.csv`, then run the pipeline in order:

```bash
python scripts/01_ingest_baseline_data.py
python scripts/02_verify_ingestion.py
python scripts/03_apply_vector_schema.py
python scripts/04_generate_embeddings.py
python scripts/05_test_vector_search.py
python scripts/06_test_guardrails.py
```

The embedding script currently processes up to 1,000 records per run. Run it again if more records need embeddings.

## 5. Run the Services

In one terminal, start the API:

```bash
uvicorn src.api.main:app --reload --host 0.0.0.0 --port 8000
```

In a second terminal, start the Streamlit UI:

```bash
streamlit run src/UI/app.py
```

Open Streamlit at `http://localhost:8501`. The API's interactive documentation is at `http://localhost:8000/docs`.

The UI selects a patient with embedded records and sends clinical questions to the API. Example retrieval questions:

- What medications were prescribed to this patient upon discharge?
- Can you summarize the last few reports for this patient?
- What liver-related diagnoses are noted in the patient's file?
- Was the patient admitted urgently or routinely?
- Can you plot this patient's heart rate over time?

Requests for a new diagnosis or a medication change should be refused by the configured guardrails. A system that blocks a request is not a substitute for clinical review.

## AWS EC2 and Docker Deployment

The supplied deployment workflow targets an Ubuntu 24.04 EC2 instance. A `t3.large` with 30-40 GB of storage is suggested for the model workload; actual requirements depend on model and usage. In the instance security group, allow SSH (port 22) only from your IP and expose Streamlit (port 8501) only to the audience that needs it. Keep PostgreSQL private. FastAPI uses port 8000 internally and does not need a public inbound rule.

Connect to the instance:

```bash
ssh -i /path/to/your-key.pem ubuntu@YOUR_EC2_PUBLIC_IP
```

Install system utilities:

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y ca-certificates curl git nano
```

Install Docker Engine by following the [official Ubuntu instructions](https://docs.docker.com/engine/install/ubuntu/). Enable and verify it:

```bash
sudo systemctl enable --now docker
sudo usermod -aG docker "$USER"
newgrp docker
docker --version
docker run --rm hello-world
```

Clone the repository:

```bash
git clone https://github.com/yashaggarwal10/Secure-EHR-Insight-Clinical-Validator.git
cd Secure-EHR-Insight-Clinical-Validator
```

Create a server-side `.env` with the database and model-provider values described above. Do not upload your SSH key or `.env` to GitHub.

The notes use a single container for both services. Create a `.dockerignore` in the project root so credentials, local data, and developer files are not copied into the image:

```text
.git
.gitignore
.env
*.env
__pycache__
*.pyc
*.pyo
.venv
venv
env
data
```

Create `start.sh` to launch FastAPI and then Streamlit. The uppercase `UI` in the Streamlit path is intentional because Linux paths are case-sensitive:

```sh
#!/bin/sh
set -e

uvicorn src.api.main:app --host 0.0.0.0 --port 8000 &
API_PID=$!

echo "Waiting for FastAPI to become ready..."
while ! curl -fsS http://127.0.0.1:8000/openapi.json >/dev/null 2>&1; do
	if ! kill -0 "$API_PID" 2>/dev/null; then
		echo "FastAPI failed to start."
		wait "$API_PID"
		exit 1
	fi
	sleep 5
done

exec streamlit run src/UI/app.py \
	--server.address=0.0.0.0 \
	--server.port=8501 \
	--server.headless=true
```

Make the script executable:

```bash
chmod +x start.sh
```

Create a `Dockerfile` in the project root:

```dockerfile
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1
ENV PIP_NO_CACHE_DIR=1

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
		build-essential \
		libpq-dev \
		curl \
		&& rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --upgrade pip && pip install --no-cache-dir -r requirements.txt

COPY . .
RUN chmod +x /app/start.sh

EXPOSE 8000 8501
CMD ["/app/start.sh"]
```

Build and run the container. Only Streamlit is published to the host; FastAPI remains available inside the container:

```bash
docker build --pull -t secure-ehr-insight:latest .
docker run -d \
	--name secure-ehr-insight \
	--restart unless-stopped \
	--env-file .env \
	-p 8501:8501 \
	secure-ehr-insight:latest
```

Check the container and service health:

```bash
docker ps
docker logs -f secure-ehr-insight
curl http://localhost:8501/_stcore/health
docker exec secure-ehr-insight curl -f http://127.0.0.1:8000/openapi.json
```

Browse to `http://YOUR_EC2_PUBLIC_IP:8501` only when the security group permits your intended clients. For production, put the UI behind HTTPS and authentication, restrict network access, and use managed secrets and database services.

To retrieve the instance's public IP, run `curl -s https://checkip.amazonaws.com` on EC2. If PostgreSQL did not restart with the instance, check or restart it with `sudo systemctl status postgresql` and `sudo systemctl restart postgresql`.

## Security and Limitations

- Treat clinical data as sensitive. This repository intentionally excludes the dataset and local credentials.
- PII redaction and guardrails reduce risk but do not guarantee that sensitive information will never be exposed or that model output is correct.
- The API currently includes a mock clinical-query endpoint with fabricated patient details; do not treat it as a real patient record.
- The application is a prototype. Validate access controls, privacy, logging, model behavior, and deployment security before any real-world use.
