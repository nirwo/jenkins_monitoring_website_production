# Jenkins Monitoring System - Deployment Guide

## Overview

This guide provides instructions for deploying the Jenkins Monitoring System in your environment. The system consists of several components that work together to provide comprehensive monitoring of Jenkins instances.

## Prerequisites

Before deploying the system, ensure you have the following:

### Hardware Requirements

- **CPU**: Minimum 4 cores (recommended 8+ cores)
- **Memory**: Minimum 8GB RAM (recommended 16GB+ RAM)
- **Disk Space**: Minimum 50GB (recommended 100GB+ for log storage)
- **Network**: Stable network connection to Jenkins instances

### Software Requirements

- **Operating System**: Ubuntu 20.04 LTS or newer (or equivalent Linux distribution)
- **Python**: Version 3.8 or newer
- **Node.js**: Version 14 or newer (for frontend components)
- **Database**: PostgreSQL 12 or newer (optional, for production deployments)
- **Web Server**: Nginx or Apache (for production deployments)

### Access Requirements

- **Jenkins API Access**: API token with appropriate permissions
- **SAML Identity Provider**: For SSO authentication
- **SMTP Server**: For email notifications
- **Slack/Teams Webhooks**: For chat notifications (optional)
- **PagerDuty API Access**: For incident management (optional)

## Deployment Options

The Jenkins Monitoring System can be deployed in several ways:

1. **Docker Deployment**: Recommended for most environments
2. **Kubernetes Deployment**: For scalable, production environments
3. **Manual Deployment**: For customized installations

## Docker Deployment

### Step 1: Clone the Repository

```bash
git clone https://github.com/your-org/jenkins-monitoring-system.git
cd jenkins-monitoring-system
```

### Step 2: Configure Environment Variables

Create a `.env` file in the root directory with the following variables:

```
# Jenkins Connection
JENKINS_URL=https://jenkins.example.com
JENKINS_USERNAME=admin
JENKINS_API_TOKEN=your-api-token

# SSO Authentication
SSO_ENABLED=true
SAML_METADATA_URL=https://idp.example.com/metadata.xml
SAML_ENTITY_ID=jenkins-monitoring
SAML_ACS_URL=https://monitoring.example.com/saml/acs

# Notification Settings
SMTP_SERVER=smtp.example.com
SMTP_PORT=587
SMTP_USERNAME=notifications@example.com
SMTP_PASSWORD=your-smtp-password
SMTP_FROM=jenkins-monitoring@example.com

# Slack Integration (Optional)
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/xxx/yyy/zzz

# Teams Integration (Optional)
TEAMS_WEBHOOK_URL=https://outlook.office.com/webhook/xxx/yyy/zzz

# PagerDuty Integration (Optional)
PAGERDUTY_API_KEY=your-pagerduty-api-key
PAGERDUTY_SERVICE_ID=your-pagerduty-service-id

# Database Connection (Optional)
DB_HOST=localhost
DB_PORT=5432
DB_NAME=jenkins_monitoring
DB_USER=postgres
DB_PASSWORD=your-db-password
```

### Step 3: Build and Start Docker Containers

```bash
docker-compose build
docker-compose up -d
```

### Step 4: Verify Deployment

1. Access the monitoring system at `http://localhost:5000`
2. Check container logs for any errors:
   ```bash
   docker-compose logs -f
   ```

## Kubernetes Deployment

### Step 1: Clone the Repository

```bash
git clone https://github.com/your-org/jenkins-monitoring-system.git
cd jenkins-monitoring-system/kubernetes
```

### Step 2: Configure Kubernetes Secrets

Create a Kubernetes secret for sensitive information:

```bash
kubectl create namespace jenkins-monitoring

kubectl create secret generic jenkins-monitoring-secrets \
  --namespace jenkins-monitoring \
  --from-literal=jenkins-api-token=your-api-token \
  --from-literal=smtp-password=your-smtp-password \
  --from-literal=db-password=your-db-password
```

### Step 3: Update Configuration

Edit the `configmap.yaml` file to match your environment:

```bash
vim configmap.yaml
```

### Step 4: Deploy to Kubernetes

```bash
kubectl apply -f .
```

### Step 5: Verify Deployment

```bash
kubectl get pods -n jenkins-monitoring
kubectl get services -n jenkins-monitoring
```

Access the monitoring system at the LoadBalancer IP or Ingress URL.

## Manual Deployment

### Step 1: Clone the Repository

```bash
git clone https://github.com/your-org/jenkins-monitoring-system.git
cd jenkins-monitoring-system
```

### Step 2: Install Dependencies

```bash
# Install system dependencies
sudo apt update
sudo apt install -y python3 python3-pip python3-venv nodejs npm

# Create and activate virtual environment
python3 -m venv venv
source venv/bin/activate

# Install Python dependencies
pip install -r requirements.txt

# Install Node.js dependencies
cd frontend
npm install
npm run build
cd ..
```

### Step 3: Configure Environment

Create a `.env` file with the required environment variables (see Docker deployment section).

### Step 4: Initialize Database (Optional)

For production deployments with PostgreSQL:

```bash
# Install PostgreSQL if not already installed
sudo apt install -y postgresql postgresql-contrib

# Create database and user
sudo -u postgres psql -c "CREATE DATABASE jenkins_monitoring;"
sudo -u postgres psql -c "CREATE USER jenkins_monitor WITH PASSWORD 'your-password';"
sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE jenkins_monitoring TO jenkins_monitor;"

# Initialize database schema
python scripts/init_db.py
```

### Step 5: Start the Application

For development:

```bash
# Start the main application
python main_app.py
```

For production with Gunicorn:

```bash
# Install Gunicorn
pip install gunicorn

# Start the application with Gunicorn
gunicorn -w 4 -b 0.0.0.0:5000 main_app:app
```

### Step 6: Configure Web Server (Production)

For Nginx:

```bash
sudo apt install -y nginx

# Create Nginx configuration
sudo tee /etc/nginx/sites-available/jenkins-monitoring <<EOF
server {
    listen 80;
    server_name monitoring.example.com;

    location / {
        proxy_pass http://localhost:5000;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
    }
}
EOF

# Enable the site
sudo ln -s /etc/nginx/sites-available/jenkins-monitoring /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

### Step 7: Configure Systemd Service (Production)

```bash
sudo tee /etc/systemd/system/jenkins-monitoring.service <<EOF
[Unit]
Description=Jenkins Monitoring System
After=network.target

[Service]
User=ubuntu
WorkingDirectory=/home/ubuntu/jenkins-monitoring-system
Environment="PATH=/home/ubuntu/jenkins-monitoring-system/venv/bin"
ExecStart=/home/ubuntu/jenkins-monitoring-system/venv/bin/gunicorn -w 4 -b 0.0.0.0:5000 main_app:app
Restart=always

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable jenkins-monitoring
sudo systemctl start jenkins-monitoring
```

## SSO Configuration

### Configuring SAML Identity Provider

1. **Generate Service Provider Metadata**:
   - Access the monitoring system at `/saml/metadata`
   - Download the metadata XML file

2. **Register with Identity Provider**:
   - Upload the metadata XML to your IdP (Okta, Azure AD, ADFS, etc.)
   - Configure attribute mappings:
     - `NameID`: User email or username
     - `FirstName`: User's first name
     - `LastName`: User's last name
     - `Email`: User's email address
     - `Groups`: User's group memberships (optional)

3. **Download IdP Metadata**:
   - Download the metadata XML from your IdP
   - Place it in the `saml` directory or configure the URL in environment variables

## Monitoring Configuration

### Connecting to Jenkins Instances

1. **API Configuration**:
   - Ensure the Jenkins API token has appropriate permissions
   - For multiple Jenkins instances, configure each in the settings

2. **Metrics Collection**:
   - Configure metrics collection interval in the settings
   - Adjust thresholds for system alerts based on your environment

3. **Console Output Analysis**:
   - Configure error patterns for your specific environment
   - Add custom patterns for application-specific errors

## Notification Configuration

### Email Notifications

1. Configure SMTP settings in the environment variables
2. Add recipient email addresses in the notification settings
3. Configure alert severity levels for email notifications

### Slack Notifications

1. Create a Slack app and webhook URL
2. Configure the webhook URL in the environment variables
3. Add channel names in the notification settings
4. Configure alert severity levels for Slack notifications

### Microsoft Teams Notifications

1. Create a Teams webhook connector
2. Configure the webhook URL in the environment variables
3. Add channel names in the notification settings
4. Configure alert severity levels for Teams notifications

### PagerDuty Notifications

1. Create a PagerDuty service and integration key
2. Configure the API key and service ID in the environment variables
3. Configure alert severity levels for PagerDuty incidents

## Backup and Maintenance

### Database Backup

For PostgreSQL database:

```bash
# Create backup script
tee /home/ubuntu/backup-jenkins-monitoring.sh <<EOF
#!/bin/bash
BACKUP_DIR="/home/ubuntu/backups"
TIMESTAMP=\$(date +%Y%m%d_%H%M%S)
mkdir -p \$BACKUP_DIR
pg_dump -U jenkins_monitor jenkins_monitoring > \$BACKUP_DIR/jenkins_monitoring_\$TIMESTAMP.sql
find \$BACKUP_DIR -name "jenkins_monitoring_*.sql" -mtime +7 -delete
EOF

chmod +x /home/ubuntu/backup-jenkins-monitoring.sh

# Add to crontab
(crontab -l 2>/dev/null; echo "0 2 * * * /home/ubuntu/backup-jenkins-monitoring.sh") | crontab -
```

### Log Rotation

Configure log rotation to prevent disk space issues:

```bash
sudo tee /etc/logrotate.d/jenkins-monitoring <<EOF
/home/ubuntu/jenkins-monitoring-system/*.log {
    daily
    missingok
    rotate 7
    compress
    delaycompress
    notifempty
    create 0640 ubuntu ubuntu
}
EOF
```

## Upgrading

### Docker Deployment

```bash
cd jenkins-monitoring-system
git pull
docker-compose down
docker-compose build
docker-compose up -d
```

### Kubernetes Deployment

```bash
cd jenkins-monitoring-system/kubernetes
git pull
kubectl apply -f .
```

### Manual Deployment

```bash
cd jenkins-monitoring-system
git pull
source venv/bin/activate
pip install -r requirements.txt
cd frontend
npm install
npm run build
cd ..
sudo systemctl restart jenkins-monitoring
```

## Troubleshooting

### Common Issues

#### Connection to Jenkins Failed

- Verify Jenkins URL is correct
- Check API token permissions
- Ensure network connectivity between monitoring system and Jenkins
- Check Jenkins API is not blocked by firewall

#### SSO Authentication Issues

- Verify SAML metadata is correctly configured
- Check IdP is properly configured with SP metadata
- Ensure attribute mappings are correct
- Check browser console for SAML errors

#### Notification Delivery Problems

- Verify SMTP server settings
- Check webhook URLs for Slack/Teams
- Ensure PagerDuty API key is valid
- Test notifications from the settings page

### Logs

Check application logs for detailed error information:

```bash
# Docker deployment
docker-compose logs -f

# Kubernetes deployment
kubectl logs -f deployment/jenkins-monitoring -n jenkins-monitoring

# Manual deployment
tail -f /home/ubuntu/jenkins-monitoring-system/*.log
```

## Support

For additional support:

- Check the [GitHub repository](https://github.com/your-org/jenkins-monitoring-system) for issues and updates
- Contact the development team at support@example.com
- Join the community Slack channel at [slack.example.com](https://slack.example.com)
