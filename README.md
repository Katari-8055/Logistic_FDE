# ColdChain AI 

An enterprise-style operations assistant for investigating cold-chain incidents,
route disruption, and shipment risk. The project combines live fleet telemetry,
external corridor conditions, and Standard Operating Procedure (SOP) retrieval
behind a conversational Streamlit dispatch console.

> **Project status:** Demonstration / engineering project. It is designed to
> show a production-oriented data and agent architecture; it is not a
> substitute for validated operational, safety, or regulatory systems.

## What it does

Dispatchers can ask questions such as:

> Find active shipments near Los Angeles, check the local weather, and tell me
> whether the cargo temperature violates the fresh-perishables SOP.

For an operational investigation, the agent can:

1. Query the read-only `FDE_VIEWS.VW_ACTIVE_FLEET` SQL Server view.
2. Retrieve current weather and corridor conditions from Open-Meteo.
3. Search indexed compliance material in Pinecone.
4. Produce a concise incident report with telemetry, risk, and recommended
   actions.
5. Record tool inputs, tool outputs, and final responses in an audit table.

The assistant is also instructed not to call operational tools for general
knowledge questions, such as explaining escalation tiers.

## Architecture

```text
                         +----------------------+
                         |  Streamlit UI         |
                         |  src/ui.py            |
                         +----------+-----------+
                                    |
                                    v
                         +----------------------+
                         | LangGraph orchestrator|
                         | src/orchestrator.py  |
                         +----------+-----------+
                                    |
                 +------------------+------------------+
                 |                  |                  |
                 v                  v                  v
        +----------------+  +---------------+  +------------------+
        | SQL Server      |  | Open-Meteo    |  | Pinecone         |
        | semantic view   |  | live weather  |  | SOP vector index |
        +----------------+  +---------------+  +------------------+
                 ^
                 |
        +----------------------+
        | CSV -> legacy table  |
        | -> protected view    |
        +----------------------+
```

### Agent tools

The tools exposed to the LangGraph agent are defined in
[`src/agent_tools.py`](./src/agent_tools.py):

- `query_telemetry_db` - executes read-only telemetry queries.
- `fetch_corridor_conditions` - retrieves current weather and derives a simple
  corridor congestion indicator.
- `search_compliance_sop` - retrieves relevant SOP passages from Pinecone.

The response format and tool-ordering policy are defined in
[`src/prompts/system_prompt.txt`](./src/prompts/system_prompt.txt).

## Repository layout

```text
.
├── data/
│   ├── cache/       Incremental ingestion hash cache
│   ├── policy/      SOP and compliance documents
│   ├── raw/         Source logistics CSV
│   └── source/      Source dataset reference
├── docs/
│   └── instrutions.md       Detailed project notes and deployment steps
├── scripts/
│   ├── ingest_legacy_data.py       CSV -> SQL Server legacy table
│   ├── ingest_sop_pinecone.py      Policy -> Pinecone
│   └── setup_security_and_view.sql SQL semantic/security layer
├── src/
│   ├── agent_tools.py
│   ├── orchestrator.py
│   ├── ui.py
│   └── prompts/system_prompt.txt
├── requirements.txt
└── README.md
```

## Prerequisites

- Python 3.12
- Docker
- Microsoft SQL Server 2022 container, or a compatible SQL Server instance
- Microsoft ODBC Driver 18 for SQL Server
- A Pinecone account and API key
- An LLM provider:
  - local Ollama (`qwen2.5:7b`, default fallback), or
  - OpenAI, or
  - DeepSeek
- API credentials for the selected cloud LLM/embedding providers

On Ubuntu, the SQL tooling also requires `unixodbc-dev`. The Microsoft ODBC
driver installation is platform-specific; follow Microsoft's current
installation instructions for your operating system.

## Installation

From the repository root:

```powershell
python -m venv .venv
.venv\Scripts\activate
python -m pip install --upgrade pip
pip install -r requirements.txt
```

For Linux/macOS, activate the environment with:

```bash
source .venv/bin/activate
```

## Configuration

Create a `.env` file in the repository root. It is ignored by Git and must
never be committed.

```dotenv
# SQL Server
SQL_SERVER_HOST=localhost
SQL_SERVER_PORT=1433
SQL_ADMIN_USER=sa
SQL_ADMIN_PASSWORD=<admin-password>
SQL_AGENT_USER=USR_FDE_RO
SQL_AGENT_PASSWORD=<agent-password>

# LLM selection: OLLAMA, OPENAI, or DEEPSEEK
Agent_llm=OLLAMA

# Embeddings selection: LOCAL or OPENAI
Embeddings_model=LOCAL
Local_Embedding_Model=BAAI/bge-m3

# Required for SOP indexing and retrieval
PINECONE_API_KEY=<pinecone-api-key>

# Required when using the matching provider
OPENAI_API_KEY=<openai-api-key>
DEEPSEEK_API_KEY=<deepseek-api-key>
```

Only define credentials for services you use. Keep production secrets in a
secret manager or deployment environment rather than in a local file.

## Data and database setup

### 1. Start SQL Server

Example Docker command for local development:

```powershell
docker run -e "ACCEPT_EULA=Y" `
  -e "MSSQL_SA_PASSWORD=<strong-password>" `
  -p 1433:1433 `
  --name legacy-mssql `
  -d mcr.microsoft.com/mssql/server:2022-latest
```

The SQL Server instance must be reachable at the host and port configured in
`.env`. The password must satisfy SQL Server password complexity rules.

### 2. Load the sample telemetry

The loader maps the clean CSV columns to a legacy-style table named
`dbo.TBL_SC_FLEET_HIST_RAW`:

```powershell
python scripts\ingest_legacy_data.py
```

### 3. Create the semantic and security layer

Run [`scripts/setup_security_and_view.sql`](./scripts/setup_security_and_view.sql)
as a SQL Server administrator. It creates:

- `FDE_VIEWS.VW_ACTIVE_FLEET`, a clean column-name view for the agent.
- A read-only agent login/user.
- Permissions that prevent the agent from reading the raw legacy table or
  modifying the `dbo` schema.

The UI audit view additionally expects the following table to exist:

```sql
CREATE TABLE FDE_VIEWS.AgentAuditLog (
    LogID INT IDENTITY(1,1) PRIMARY KEY,
    Timestamp DATETIME DEFAULT GETDATE(),
    SessionID VARCHAR(50),
    NodeExecuted VARCHAR(50),
    ToolName VARCHAR(100),
    Content NVARCHAR(MAX)
);

GRANT INSERT ON FDE_VIEWS.AgentAuditLog TO USR_FDE_RO;
```

Use your own strong credentials when creating database users. Do not copy
passwords from project notes into a shared or production environment.

## Index the SOP documents

The incremental Pinecone pipeline reads supported policy assets from
`data/policy/` (`.md`, `.txt`, `.pdf`, `.csv`, and `.xlsx`), chunks them, and
stores a hash cache at `data/cache/ingestion_hash_cache.json`.

```powershell
python scripts\ingest_sop_pinecone.py
```

The index name and dimension are selected from the embedding configuration:

| Embeddings setting | Pinecone index | Dimension |
|---|---|---:|
| `Embeddings_model=LOCAL` | `fde-sop-index-local` | 1024 |
| `Embeddings_model=OPENAI` | `fde-sop-index-openai` | 1536 |

Do not switch embedding providers for an existing index without allowing the
ingestion script to create or rebuild the matching index.

## Run the application

### Streamlit dispatch console

```powershell
streamlit run src\ui.py
```

The console has two modes:

- **Dispatch Console** - ask questions about fleet telemetry, weather, route
  conditions, and compliance.
- **Security & Audit Logs** - authenticate with the configured administrator
  credentials and inspect recorded agent activity.

### Command-line orchestrator

For a terminal-based smoke test:

```powershell
python src\orchestrator.py
```

Type `exit` or `quit` to stop the session.

### Useful test prompts

```text
Find active shipments near Los Angeles (Latitude ~33.8, Longitude ~-118.1).
Check the local weather there, and tell me if the current cargo temperature
violates the SOP for fresh perishables.
```

```text
I'm a new dispatcher on the night shift. Can you quickly explain the
difference between a Tier 1 and Tier 2 escalation?
```

## SOP rules represented by the sample policy

The included policy defines these example triggers:

- **Fresh perishables:** cargo temperature must remain between 0.0 C and
  4.0 C. Above 4.0 C is a critical breach.
- **Cold-chain mitigation:** contact the driver to restart auxiliary cooling;
  divert to emergency cold storage when the ETA delay exceeds one hour.
- **Port congestion:** above severity 7.0 at the Port of Long Beach/LA,
  divert active shipments to the Inland Empire Overflow Depot.
- **Tier 2 escalation:** `High Risk` combined with delay probability above
  `0.65`.

These are sample project rules. Confirm them against current operational
policies before using the system for real decisions.

## Deployment

The repository includes a manually triggered GitHub Actions workflow at
[`.github/workflows/deploy.yml`](./.github/workflows/deploy.yml). It pulls the
latest `main` branch on an EC2 host and restarts the Streamlit systemd service.

The target host must already have:

1. Python, a virtual environment, and project dependencies installed.
2. Microsoft ODBC Driver 18 and `unixodbc-dev`.
3. A configured `.env` file.
4. A running SQL Server endpoint.
5. A configured Pinecone index.
6. A systemd service that runs `streamlit run src/ui.py`.

Configure `EC2_HOST`, `EC2_USER`, and `EC2_SSH_KEY` as GitHub Actions secrets.
Do not place SSH keys, API keys, or database passwords in the repository.

## Troubleshooting

### SQL connection errors

- Confirm SQL Server is running and port `1433` is reachable.
- Confirm ODBC Driver 18 is installed and visible to `pyodbc`.
- Check the host, port, username, and password in `.env`.
- Run the semantic-view query as the agent user:

  ```sql
  SELECT TOP 5 * FROM FDE_VIEWS.VW_ACTIVE_FLEET;
  ```

- Confirm the agent user cannot access the raw table:

  ```sql
  SELECT TOP 5 * FROM dbo.TBL_SC_FLEET_HIST_RAW;
  ```

### Pinecone or embedding errors

- Confirm `PINECONE_API_KEY` is set.
- Confirm the selected embedding provider matches the index dimension.
- Re-run `python scripts\ingest_sop_pinecone.py` after adding or changing policy
  files.

### LLM errors

- Set `Agent_llm` to a supported value: `OLLAMA`, `OPENAI`, or `DEEPSEEK`.
- For local mode, ensure Ollama is running and the configured model is
  available.
- For cloud mode, confirm the provider API key is present and valid.

## Security notes

- Treat the sample dataset and policy documents as internal project assets.
- Never commit `.env`, passwords, API keys, SSH keys, or database connection
  strings.
- Keep the agent on the semantic read-only view; do not grant it broad database
  privileges.
- Review audit-log retention and access controls before production deployment.
- Validate all AI-generated recommendations with an authorized dispatcher.

## License and data notice

No license file is currently included in this repository. Add an explicit
license before distributing the project or its bundled data and documents.
The sample dataset and reference materials may have their own usage terms;
verify those terms independently.
