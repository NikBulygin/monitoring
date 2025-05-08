# Monitoring Stack Backup and Deployment Guide

This guide explains how to backup and restore the monitoring stack components (Prometheus, Grafana, Alertmanager, RabbitMQ, and Blackbox Exporter).

## Directory Structure

```
.
├── alertmanager/
│   └── config.yml
├── blackbox/
│   └── config.yml
├── grafana/
│   └── provisioning/
├── prometheus/
│   ├── prometheus.yml
│   └── rules/
├── rabbitmq/
│   ├── definitions.json
│   └── rabbitmq.conf
└── docker-compose.yml
```

## Prometheus Configuration Reload

After making changes to `prometheus.yml`, you can reload the configuration without restarting the service using:

```bash
# Using curl
curl -X POST http://localhost:9090/-/reload

# Or using Docker
docker exec prometheus wget -q --post-data='' --header='Content-Type:application/json' http://localhost:9090/-/reload
```

This is possible because Prometheus is started with the `--web.enable-lifecycle` flag in the docker-compose configuration.

## Backup Process

### 1. Configuration Files Backup

All configuration files are stored in the local directories and mounted into containers. To backup configurations:

```bash
# Create backup directory
mkdir -p backup/monitoring

# Backup all configuration files
cp -r alertmanager backup/monitoring/
cp -r blackbox backup/monitoring/
cp -r grafana backup/monitoring/
cp -r prometheus backup/monitoring/
cp -r rabbitmq backup/monitoring/
cp docker-compose.yml backup/monitoring/
```

### 2. Data Volumes Backup

The following volumes contain persistent data:

- prometheus-data: Prometheus time series data
- alertmanager-data: Alertmanager data
- grafana-data: Grafana database and plugins
- rabbitmq-data: RabbitMQ data and messages

To backup volumes:

```bash
# Stop the stack
docker-compose down

# Backup volumes
docker run --rm -v prometheus-data:/source -v $(pwd)/backup:/backup alpine tar -czf /backup/prometheus-data.tar.gz -C /source .
docker run --rm -v alertmanager-data:/source -v $(pwd)/backup:/backup alpine tar -czf /backup/alertmanager-data.tar.gz -C /source .
docker run --rm -v grafana-data:/source -v $(pwd)/backup:/backup alpine tar -czf /backup/grafana-data.tar.gz -C /source .
docker run --rm -v rabbitmq-data:/source -v $(pwd)/backup:/backup alpine tar -czf /backup/rabbitmq-data.tar.gz -C /source .

# Start the stack
docker-compose up -d
```

## Restore Process

### 1. Restore Configuration Files

```bash
# Restore configuration files
cp -r backup/monitoring/* .
```

### 2. Restore Data Volumes

```bash
# Stop the stack
docker-compose down

# Remove existing volumes
docker volume rm prometheus-data alertmanager-data grafana-data rabbitmq-data

# Create new volumes
docker volume create prometheus-data
docker volume create alertmanager-data
docker volume create grafana-data
docker volume create rabbitmq-data

# Restore data
docker run --rm -v prometheus-data:/target -v $(pwd)/backup:/backup alpine sh -c "cd /target && tar -xzf /backup/prometheus-data.tar.gz"
docker run --rm -v alertmanager-data:/target -v $(pwd)/backup:/backup alpine sh -c "cd /target && tar -xzf /backup/alertmanager-data.tar.gz"
docker run --rm -v grafana-data:/target -v $(pwd)/backup:/backup alpine sh -c "cd /target && tar -xzf /backup/grafana-data.tar.gz"
docker run --rm -v rabbitmq-data:/target -v $(pwd)/backup:/backup alpine sh -c "cd /target && tar -xzf /backup/rabbitmq-data.tar.gz"

# Start the stack
docker-compose up -d
```

## Automated Backup Script

Create a file named `backup.sh`:

```bash
#!/bin/bash

# Create backup directory
BACKUP_DIR="backup/monitoring/$(date +%Y%m%d_%H%M%S)"
mkdir -p "$BACKUP_DIR"

# Backup configuration files
cp -r alertmanager "$BACKUP_DIR/"
cp -r blackbox "$BACKUP_DIR/"
cp -r grafana "$BACKUP_DIR/"
cp -r prometheus "$BACKUP_DIR/"
cp -r rabbitmq "$BACKUP_DIR/"
cp docker-compose.yml "$BACKUP_DIR/"

# Stop the stack
docker-compose down

# Backup volumes
docker run --rm -v prometheus-data:/source -v "$BACKUP_DIR":/backup alpine tar -czf /backup/prometheus-data.tar.gz -C /source .
docker run --rm -v alertmanager-data:/source -v "$BACKUP_DIR":/backup alpine tar -czf /backup/alertmanager-data.tar.gz -C /source .
docker run --rm -v grafana-data:/source -v "$BACKUP_DIR":/backup alpine tar -czf /backup/grafana-data.tar.gz -C /source .
docker run --rm -v rabbitmq-data:/source -v "$BACKUP_DIR":/backup alpine tar -czf /backup/rabbitmq-data.tar.gz -C /source .

# Start the stack
docker-compose up -d

echo "Backup completed: $BACKUP_DIR"
```

Make it executable:
```bash
chmod +x backup.sh
```

## Important Notes

1. Always test the backup and restore process in a non-production environment first
2. Keep multiple backup copies in different locations
3. Regularly verify backup integrity
4. Consider using a backup rotation strategy to manage disk space
5. For production environments, consider using a proper backup solution like Velero or similar tools

## Security Considerations

1. Ensure backup files are stored securely
2. Encrypt sensitive data in backups
3. Implement proper access controls for backup files
4. Regularly audit backup access logs
5. Consider using a secure backup storage solution for production environments 