# Qumulo Audit Monitor

A simple audit log monitoring solution for Qumulo storage systems using Alloy, Loki, and Grafana.

This is to be used if one would like a simple quick way to monitor stats and query Qumulo audit logs, but does not currently have the infrastructure set up to do so. 
It is not intended on being treated as a fully supported Qumulo solution. Use this freely, but please self support. Suggestions are always welcome, but not necessarily going to be added.

This solution leverages Qumulo's default audit log CSV format. It is not currently designed to leverage Qumulo's more comprehensive audit log JSON format.

## 📋 Overview

This project captures and visualizes audit logs from Qumulo storage systems in real-time, providing:
- Real-time log monitoring and searching
- Statistical analysis and visualization
- Flexible filtering by operations, protocols, users, and file paths
- 31-day log retention

## 🏗️ Architecture

```
Qumulo Systems → Alloy (Port 514/TCP) → Loki → Grafana
                    ↓
               Parse CSV
               Extract Labels
                    ↓
                Store Logs
                    ↓
              Visualize Data
```

**Components:**
- **Grafana Alloy** - Receives syslog messages, parses CSV format, extracts metadata
- **Grafana Loki** - Stores and indexes log data with 31-day retention
- **Grafana** - Provides interactive dashboards for visualization and querying

## 📁 Project Structure

```
qumulo-audit-monitor_latest/
├── docker-compose.yml           # Main deployment configuration
├── alloy/
│   └── configs/
│       └── alloy-config.alloy   # Alloy pipeline configuration
├── loki/
│   ├── configs/
│   │   └── loki-config.yaml     # Loki storage and retention config
│   └── data/                    # Loki data storage (gitignored)
└── grafana/
    ├── configs/
    │   ├── grafana.ini          # Grafana server configuration
    │   └── provisioning/
    │       ├── datasources/
    │       │   └── datasource.yml
    │       └── dashboards/
    │           ├── dashboard.yml
    │           ├── qumulo-audit-query.json    # Log query dashboard
    │           └── qumulo-audit-stats.json    # Statistics dashboard
    └── data/                    # Grafana data storage (gitignored)
```

## 🚀 Quick Start

### Prerequisites

- A machine or VM with 4 CPU cores and 16GB of RAM (Loki likes memory)
- At least 200GB free disk space for log storage *this is an estimate* mileage varies based on how busy a cluster is
- Docker, Docker Compose, and GIT installed
- Network access from Qumulo systems to the host on TCP port 514

### Deployment

1. **Clone the project:**
   ```bash
   git clone https://github.com/fil-pereira-Qumulo/qumulo-audit-monitor.git
   ```
   
2. **Change directory into your new repo directory:**
   ```bash
   cd qumulo-audit-monitor
   ```

2. **Make data directories wide open (easiest way):**
   ```bash
   chmod 777 loki/data
   chmod 777 grafana/data
   ```

3. **Start the stack:**
   ```bash
   docker compose up -d
   ```

4. **Verify services are running:**
   ```bash
   docker compose ps
   ```
   All services should show as "Up"

5. **Access Grafana:**
   - URL: http://localhost:3000
   - Default credentials: admin / admin (change on first login)

6. **Configure Qumulo to send syslogs:**
   - Point Qumulo audit logs to your host IP on TCP port 514
   - Ensure logs are in CSV format with the following fields:
     `client_ip,"login_id",protocol,operation,operation_result,file_id,"primary_path","secondary_path"`

### Verification

1. Check Alloy doesn't have obvious errors in it's logs:
   ```bash
   docker logs Alloy --tail 50
   ```

2. Check Loki has data:
   ```bash
   curl http://localhost:3100/ready
   curl http://localhost:3100/loki/api/v1/labels
   ```

3. Open Grafana and view the dashboards:
   - **Qumulo Audit Query** - Search and filter logs
   - **Qumulo Audit Stats** - View statistics and analytics

## 📊 Dashboards

### Qumulo Audit Query Dashboard
Interactive log viewer with:
- Full-text search
- Filter by qumulo node, protocol, operation, result
- Filter by client IP, user, file path (regex support)
- Expandable log details with structured metadata

### Qumulo Audit Stats Dashboard
Statistical visualizations including:
- Operations over time (by type)
- Operations by result (pie chart)
- Operations by protocol (pie chart)
- Operations by node (pie chart)
- Top 10 most active client IPs
- Top 10 most active users
- Total log entry counts

## 🔧 Configuration

### Port Configuration

**Default Ports:**
- `514` - Alloy syslog input (TCP)
- `3100` - Loki API
- `3000` - Grafana web interface
- `12345` - Alloy web UI

To change ports, edit `docker-compose.yml`:
```yaml
ports:
  - "YOUR_PORT:514"  # Change YOUR_PORT to desired port
```

### Log Retention

Default retention is 31 days. To change:

Edit `loki/configs/loki-config.yaml`:
```yaml
limits_config:
  retention_period: 744h  # Change to desired hours (744h = 31 days)
```

Restart Loki:
```bash
docker compose restart loki
```

### Data Storage

Persistent data is stored in:
- `loki/data/` - Log data and indexes
- `grafana/data/` - Grafana database and settings

**Backup these directories** to preserve your data.

## 🔍 Data Model

### Indexed Labels (Low Cardinality)
- `job` - Always "qumulo_syslog"
- `qumulo_node` - Hostname of Qumulo cluster node
- `protocol` - SMB2, API, NFS, etc.
- `operation` - fs_read_metadata, fs_create_file, etc.
- `operation_result` - ok, fs_access_denied_error, etc.

### Structured Metadata (High Cardinality)
- `client_ip` - IP address of client
- `login_id` - Username/account
- `file_id` - Qumulo file ID (can be empty)
- `primary_filesystem_path` - Primary file path
- `secondary_filesystem_path` - Secondary file path (often empty) will poputate on moves/renames of directories or files

## 🛠️ Maintenance

### View Logs

```bash
# View all logs
docker compose logs

# View specific service logs
docker logs Alloy --tail 100
docker logs Loki --tail 100
docker logs Grafana --tail 100
```

### Restart Services

```bash
# Restart all services
docker compose restart

# Restart specific service
docker compose restart alloy
docker compose restart loki
docker compose restart grafana
```

### Stop Services

```bash
docker compose down
```

### Update Services

```bash
# Pull latest images
docker compose pull

# Restart with new images
docker compose up -d
```

### Clear Data (Start Fresh)

⚠️ **Warning: This will delete all logs and settings**

```bash
docker compose down
sudo rm -rf loki/data/* grafana/data/*
docker compose up -d
```

## 📈 Scaling Considerations

**For high-volume environments (>100k logs/day):**

1. **Increase Loki resources:**
   Edit `docker-compose.yml`:
   ```yaml
   loki:
     deploy:
       resources:
         limits:
           memory: 12G
         reservations:
           memory: 6G
   ```

2. **Adjust ingestion rate limits:**
   Edit `loki/configs/loki-config.yaml`:
   ```yaml
   limits_config:
     ingestion_rate_mb: 40
     ingestion_burst_size_mb: 80
   ```

3. **Use external storage** (S3, GCS) instead of local filesystem

## 🔒 Security Recommendations

1. **Change default Grafana password** immediately on first login
2. **Enable authentication** on Grafana (configured in `grafana/configs/grafana.ini`)
3. **Use TLS/SSL** for production deployments
4. **Restrict network access** to port 514 to only Qumulo systems
5. **Regular backups** of `loki/data/` and `grafana/data/`

## 🐛 Troubleshooting

### No logs appearing in Grafana

1. Check Alloy is receiving syslogs:
   ```bash
   docker logs Alloy | grep "received"
   ```

2. Verify Qumulo is sending to correct IP/port:
   ```bash
   sudo tcpdump -i any port 514
   ```

3. Check Loki has data:
   ```bash
   curl http://localhost:3100/loki/api/v1/labels
   ```

### Dashboard shows "No data"

1. Verify time range is correct (last 1-6 hours)
2. Check filter variables aren't too restrictive
3. Verify Loki datasource is connected in Grafana

### High disk usage

1. Check retention settings in `loki-config.yaml`
2. Ensure compactor is running (check Loki logs)
3. Manually compact old data if needed

### Container fails to start

```bash
# Check logs for errors
docker logs <container_name>

# Check port conflicts
sudo netstat -tlnp | grep -E '514|3000|3100'
```

## 📝 CSV Format Reference

Expected syslog message format:
```
<PRI>VERSION TIMESTAMP HOSTNAME APP - - - client_ip,"login_id",protocol,operation,result,file_id,"path1","path2"
```

Example:
```
<14>1 2026-01-27T14:12:23.9558Z cs-fil-q-k144-1 qumulo - - - 10.220.151.129,"admin",smb2,fs_create_file,ok,140624751830626345080086044861,"/data/file.txt",""
```

## 🤝 Contributing

To modify the project:

1. **Update Alloy config:** Edit `alloy/configs/alloy-config.alloy`
2. **Update Loki config:** Edit `loki/configs/loki-config.yaml`
3. **Update dashboards:** Export from Grafana UI and save to `grafana/configs/provisioning/dashboards/`
4. **Test changes:**
   ```bash
   docker compose restart <service>
   ```

## 📄 License

This project uses open-source components:
- Grafana Alloy (Apache 2.0)
- Grafana Loki (AGPL 3.0)
- Grafana (AGPL 3.0)

## 🆘 Support

For issues or questions:
1. Check the troubleshooting section above
2. Review Docker logs for error messages
3. Consult official documentation:
   - [Grafana Alloy](https://grafana.com/docs/alloy/)
   - [Grafana Loki](https://grafana.com/docs/loki/)
   - [Grafana](https://grafana.com/docs/grafana/)

---

**Version:** 1.0
**Last Updated:** January 2026
