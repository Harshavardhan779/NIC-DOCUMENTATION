# Argus – Oracle Intelligence Platform
## Support PC Deployment & Setup Guide

---

## 1. Overview

Argus – Oracle Intelligence Platform is deployed locally on the Support PC using Docker Desktop.

The platform analyzes Oracle Forms/Reports source code and can integrate Oracle database and Jira information for production support and RCA.

### Deployment Architecture

```text
Support PC
│
├── Docker Desktop / WSL2
│   ├── Argus Web
│   ├── Argus API
│   ├── PostgreSQL
│   ├── Redis
│   ├── Neo4j
│   ├── Ollama
│   ├── Prometheus
│   └── Grafana
│
├── Oracle Middleware
│   ├── Java/JDK
│   ├── Forms2XML
│   └── Oracle Forms libraries
│
├── Oracle Forms Source
│   └── D:\FMB\FMB
│
└── Extraction Runner
    └── FMB → XML → Argus Parser
```

---

# 2. Prerequisites

## 2.1 Hardware

| Requirement | Recommended |
|---|---|
| RAM | 16 GB+ |
| Free Disk | 20 GB+ |
| CPU | Suitable for Docker and local LLM processing |
| Virtualization | Enabled in BIOS |

## 2.2 Software

- Docker Desktop
- WSL2 enabled
- Oracle Forms/Reports 12c
- Oracle Java/JDK
- PowerShell
- VPN access to the relevant PREMIA environment

## 2.3 Oracle Forms Requirements

The Support PC must have:

- Oracle Forms source dump (`.FMB`)
- Oracle Middleware installation
- Forms2XML tooling
- Oracle Forms JDAPI/native libraries

Example:

```text
Forms Source:
D:\FMB\FMB

Oracle Middleware:
D:\Oracle\Middleware\Oracle_Home
```

## 2.4 Database Requirements

If Oracle database extraction or runtime evidence is required, provide:

- Oracle Host
- Port
- Service Name
- Username
- Password
- TNS alias / `tnsnames.ora`, where applicable
- VPN/network access

> Database connectivity is not required for the initial FMB-to-XML extraction.

## 2.5 Integration Requirements

If required:

- Jira access/details
- Approved LLM/model access

---

# 3. Obtain the Argus Package

Download the `premia_oip` deployment package and extract it to the Support PC.

Example:

```text
D:\premia_oip
```

Open PowerShell:

```powershell
cd D:\premia_oip
```

---

# 4. Import Docker Images

Run:

```powershell
.\scripts\import-images.ps1
```

Verify:

```powershell
docker images
```

---

# 5. Configure Environment

Create the environment file:

```powershell
Copy-Item .env.template .env
```

Generate the required secrets:

```powershell
.\scripts\generate-secrets.ps1 -WriteToEnv
```

Open the file:

```powershell
notepad .env
```

---

# 6. Configure Oracle Paths

Configure the actual Oracle installation paths.

Example:

```text
ORACLE_HOME_PATH=D:\Oracle\Middleware\Oracle_Home

RWCONVERTER_PATH=D:\Oracle\Middleware\Oracle_Home\bin\rwconverter.exe

FRM2XML_PATH=D:\Oracle\Middleware\Oracle_Home\forms\templates\scripts\frmf2xml.bat

TNSNAMES_PATH=D:\Oracle\Middleware\Oracle_Home\user_projects\domains\base_domain\config\fmwconfig\tnsnames.ora

WINDOWS_BASE_PATH=D:\
VCRA_BASE_PATH=D:\
```

> Update these paths according to the Oracle installation on the target Support PC.

---

# 7. Start Argus

Start the Docker stack:

```powershell
.\scripts\docker-up.ps1 -SkipPull
```

Verify the containers:

```powershell
docker compose ps
```

All required services should be running/healthy.

---

# 8. Access Argus

| Service | URL |
|---|---|
| Argus Web | http://localhost:3000 |
| Argus API | http://localhost:3001/api |
| Grafana | http://localhost:3002 |
| Neo4j | http://localhost:7474 |

Use the configured administrator credentials from `.env`.

---

# 9. Configure the Oracle Forms Source

The original FMB dump should remain on the Support PC.

Example:

```text
D:\FMB\FMB
```

Docker exposes the Windows `D:\` drive to the Argus API through `/host`.

Therefore, the Forms path configured in Argus should be:

```text
/host/FMB/FMB
```

### Path Mapping

```text
Windows:
D:\FMB\FMB

        ↓

Docker:
/host/FMB/FMB
```

> The Forms source path belongs to the customer configuration, not `.env`.

---

# 10. Start the Extraction Runner

Open a **separate PowerShell window**:

```powershell
cd D:\premia_oip
.\scripts\extraction-runner.ps1
```

Keep the extraction runner running while Forms extraction is being performed.

### Extraction Flow

```text
Argus API
    ↓
Extraction Queue
    ↓
Windows Extraction Runner
    ↓
Forms2XML
    ↓
XML
    ↓
Argus Parser
```

---

# 11. Validate Oracle Forms Conversion

Before processing the complete Forms dump, validate one FMB.

Example:

```text
D:\FMB\FMB\BM005.FMB
```

Expected flow:

```text
BM005.FMB
    ↓
Forms2XML
    ↓
BM005.xml
    ↓
Argus XML Parser
```

Confirm that the XML is generated and successfully parsed by Argus.

> Do not process the complete Forms dump until the single-FMB validation succeeds.

---

# 12. Configure Customer

Configure the customer in Argus with the applicable:

- Customer name/code
- Tenant
- Forms path
- Reports path
- Libraries path
- Menus path
- VCRA path
- Oracle Host
- Oracle Port
- Oracle Service Name
- TNS alias
- Database credentials, if required

For the current Forms source:

```text
/host/FMB/FMB
```

---

# 13. Full Forms Extraction

After successful single-FMB validation:

1. Confirm the extraction runner is running.
2. Confirm the Forms source path is accessible.
3. Select the customer.
4. Start **Full Extraction**.
5. Monitor the extraction.
6. Verify artifacts and dependencies.

### Full Extraction Flow

```text
FMB Dump
   │
   ▼
Extraction Queue
   │
   ▼
Windows Extraction Runner
   │
   ▼
Forms2XML
   │
   ▼
XML
   │
   ▼
Argus XML Parser
   │
   ▼
Oracle Artifacts
   │
   ▼
Dependencies
   │
   ▼
RCA Knowledge Base
```

---

# 14. Oracle Database Configuration

Database connectivity is separate from FMB extraction.

If database extraction is required, configure:

```text
Host:
Port:
Service Name:
Username:
Password:
TNS Alias:
```

Ensure the Support PC has VPN/network access to the required Oracle environment.

> Do not use sample or placeholder Oracle database credentials.

---

# 15. Jira Integration

If Jira integration is required, configure:

```text
JIRA_INSTANCE_URL=
JIRA_PROJECT_KEY=
JIRA_WEBHOOK_SECRET=
```

Use organization-approved Jira credentials/access.

---

# 16. LLM Configuration

Argus supports local Ollama-based processing.

Check available models:

```powershell
docker exec oip-ollama ollama list
```

Check LLM health:

```powershell
curl http://localhost:3001/api/llm/health
```

If the corporate network blocks model downloads, coordinate with IT or use the approved model provisioning method.

---

# 17. Day-to-Day Operations

### Start Argus

```powershell
.\scripts\docker-up.ps1
```

### Stop Argus

```powershell
.\scripts\docker-down.ps1
```

### View API logs

```powershell
docker compose logs -f api
```

### View Web logs

```powershell
docker compose logs -f web
```

### View Ollama logs

```powershell
docker compose logs -f ollama
```

### Check containers

```powershell
docker compose ps
```

### Backup

```powershell
.\scripts\backup.ps1
```

---

# 18. Troubleshooting

## Docker does not start

Verify:

- Docker Desktop is running.
- WSL2 is enabled.
- Hardware virtualization is enabled.

## Extraction is not working

Start the extraction runner:

```powershell
.\scripts\extraction-runner.ps1
```

## Oracle Forms tools are not found

Verify:

```powershell
Test-Path "D:\Oracle\Middleware\Oracle_Home"
Test-Path "D:\Oracle\Middleware\Oracle_Home\bin\rwconverter.exe"
Test-Path "D:\Oracle\Middleware\Oracle_Home\forms\templates\scripts\frmf2xml.bat"
Test-Path "D:\Oracle\Middleware\Oracle_Home\oracle_common\jdk\bin\java.exe"
```

## TNS connection fails

Verify:

- VPN is connected.
- Host and port are reachable.
- Service name is correct.
- `TNSNAMES_PATH` is correct.
- Database credentials are valid.

## Ollama models unavailable

```powershell
docker exec oip-ollama ollama list
```

Then:

```powershell
curl http://localhost:3001/api/llm/health
```

## API not starting

```powershell
docker compose logs -f api
```

## Port already in use

Check the configured ports in `.env`:

```text
PORT_WEB=3000
PORT_API=3001
PORT_GRAFANA=3002
PORT_POSTGRES=5433
PORT_REDIS=6379
PORT_OLLAMA=11435
PORT_PROMETHEUS=9090
PORT_NEO4J_HTTP=7474
PORT_NEO4J_BOLT=7687
```

---

# 19. Production Deployment Checklist

- [ ] Docker Desktop installed
- [ ] WSL2 enabled
- [ ] Hardware virtualization enabled
- [ ] Oracle Forms/Reports 12c installed
- [ ] Oracle Java/JDK available
- [ ] FMB source dump available
- [ ] Forms source path verified
- [ ] Oracle Middleware path verified
- [ ] `rwconverter.exe` verified
- [ ] Forms2XML verified
- [ ] Oracle Forms native libraries available
- [ ] `.env` configured
- [ ] Application secrets generated
- [ ] Docker images imported
- [ ] Argus containers healthy
- [ ] Web application accessible
- [ ] Extraction runner running
- [ ] Single FMB successfully processed
- [ ] Customer configured
- [ ] Oracle DB connectivity verified, if required
- [ ] Jira connectivity verified, if required
- [ ] LLM availability verified
- [ ] Full Forms extraction completed
- [ ] Artifacts and dependencies verified

---

# 20. Final Deployment Flow

```text
                    SUPPORT PC
                        │
        ┌───────────────┴────────────────┐
        │                                │
        ▼                                ▼
 Docker Desktop                    Oracle Middleware
 / WSL2                            Oracle Forms 12c
        │                                │
        │                                ├── Java/JDK
        │                                ├── Forms2XML
        │                                └── JDAPI
        │
        ├── Argus Web
        ├── Argus API
        ├── PostgreSQL
        ├── Redis
        ├── Neo4j
        ├── Ollama
        ├── Prometheus
        └── Grafana
                 │
                 ▼
        Customer Configuration
                 │
                 ▼
          /host/FMB/FMB
                 │
                 ▼
       Extraction Runner
                 │
                 ▼
                FMB
                 │
                 ▼
            Forms2XML
                 │
                 ▼
                XML
                 │
                 ▼
          Argus XML Parser
                 │
                 ▼
       Artifacts + Dependencies
                 │
                 ▼
                RCA
```

---

# 21. Important Notes

- The original `.FMB` files remain the source of truth.
- Argus uses the host extraction runner for Forms2XML conversion.
- The extraction runner must remain active during extraction.
- Validate a single FMB before processing the complete dump.
- Oracle database connectivity is separate from Forms source extraction.
- Do not use placeholder Oracle database credentials in production.
- Keep production credentials and secrets out of source control.
- For production deployment, the extraction runner can be installed as a Windows service.
