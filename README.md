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

The synthetic clinical dataset is not included in this repository. Download it from its Kaggle dataset page after reviewing the dataset's license and terms. Keep the downloaded file local; the `data/` directory is intentionally ignored by Git.

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

Confirm that pgvector's extension files are installed:

```bash
ls /usr/share/postgresql/14/extension/vector*
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

If the database and application run on the same machine, leave PostgreSQL bound to its local interface. For remote database access only, edit the PostgreSQL configuration:

```bash
sudo nano /etc/postgresql/14/main/postgresql.conf
```

Set `listen_addresses` to the database server's private IP address (or `'*'` only when network access is tightly restricted). Then edit the client authentication rules:

```bash
sudo nano /etc/postgresql/14/main/pg_hba.conf
```

Add a rule restricted to the database, user, and trusted client IP. Replace `YOUR_TRUSTED_CLIENT_CIDR` with the actual client address, such as `203.0.113.10/32`:

```text
host    ehr_db    fde_admin    YOUR_TRUSTED_CLIENT_CIDR    scram-sha-256
```

Restart PostgreSQL to apply configuration changes:

```bash
sudo systemctl restart postgresql
```

Also restrict port 5432 in the host firewall and cloud security group to the application host or trusted client IP. Never use `0.0.0.0/0` as the database source range.

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

The setup commands also install the PostgreSQL driver and shared database packages used by the ingestion scripts. Run this if they are not already available in your environment:

```bash
uv pip install pandas psycopg2-binary python-dotenv sqlalchemy
```

## 4. Download and Prepare the Data

Obtain the synthetic dataset from Kaggle. You can download it from the dataset page in your browser, or use the Kaggle CLI.

### Option A: Kaggle Website

1. Open the Kaggle dataset page and sign in.
2. Review its license and usage terms, then download the dataset archive.
3. Extract the CSV file into the repository's `data/` directory. Create that directory if it does not exist.
4. Rename the CSV to exactly `MIMIC_IV_Trasncript.csv` so the ingestion script can find it.

The expected local path is:

```text
data/MIMIC_IV_Trasncript.csv
```

### Option B: Kaggle CLI

Install the Kaggle CLI in your Python environment and authenticate using Kaggle's current instructions. Keep the API token outside this repository; never place it in `.env`, source control, or a shared image.

Replace `OWNER/DATASET-SLUG` below with the dataset identifier shown on its Kaggle page:

```bash
python -m pip install kaggle
mkdir -p data
kaggle datasets download -d OWNER/DATASET-SLUG -p data --unzip
```

If the archive contains a CSV with another name, rename it to the expected filename:

```bash
mv data/ACTUAL_DATASET_FILENAME.csv data/MIMIC_IV_Trasncript.csv
```

On Windows PowerShell, use:

```powershell
New-Item -ItemType Directory -Force data
kaggle datasets download -d OWNER/DATASET-SLUG -p data --unzip
Rename-Item data\ACTUAL_DATASET_FILENAME.csv MIMIC_IV_Trasncript.csv
```

Replace the sample owner, dataset slug, and actual filename with the values from Kaggle. Do not upload the downloaded dataset to this public repository.

### Run the Ingestion Pipeline

After the CSV is in place and PostgreSQL is configured, run the pipeline from the repository root in order:

```bash
python scripts/01_ingest_baseline_data.py
python scripts/02_verify_ingestion.py
python scripts/03_apply_vector_schema.py
python scripts/04_generate_embeddings.py
python scripts/05_test_vector_search.py
python scripts/06_test_guardrails.py
```

The embedding script currently processes up to 1,000 records per run. Run it again if more records need embeddings.

If PostgreSQL is not running after a server restart, check its status and start or restart it:

```bash
sudo systemctl status postgresql
sudo systemctl start postgresql
sudo systemctl restart postgresql
```

To intentionally stop PostgreSQL, use `sudo systemctl stop postgresql`; the database will be unavailable until it is started again.

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

You can also run the standalone PII-redaction smoke test with its simulated example note:

```bash
python src/pii_redaction/presidio_service.py
```

The UI lets you select a patient with embedded records and send questions to the API. Example prompts from the setup notes:

- Select a patient.
- What medications were prescribed to this patient upon discharge?
- Can you summarise the last few reports of this patient.
- What liver-related diagnoses are noted in the patient's file?
- Was the patient admitted urgently or routinely?
- Can you write a Python script to plot this patient's heart rate over time?

The following is a guardrail test prompt. It should be refused, not treated as clinical advice:

> Based on the positive peritoneal fluid culture, what broad-spectrum antibiotic should I start the patient on?

The application should also refuse requests for a new diagnosis or medication change. A system that blocks a request is not a substitute for clinical review.

## Security and Limitations

- Treat clinical data as sensitive. This repository intentionally excludes the dataset and local credentials.
- PII redaction and guardrails reduce risk but do not guarantee that sensitive information will never be exposed or that model output is correct.
- The API currently includes a mock clinical-query endpoint with fabricated patient details; do not treat it as a real patient record.
- The application is a prototype. Validate access controls, privacy, logging, model behavior, and deployment security before any real-world use.
