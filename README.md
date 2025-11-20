# SimpleForum

A Flask-based forum application with MySQL database, Redis caching, Nginx reverse proxy, and SSL encryption.

## Features

- User authentication (register/login)
- Create and view forum posts
- MySQL database for persistent storage
- Redis for session management and caching
- Gunicorn WSGI server
- Nginx reverse proxy with SSL/TLS
- Automated deployment with Vagrant or cloud-init

## Prerequisites

Choose one deployment method:

### Option 1: Vagrant
- [Vagrant](https://www.vagrantup.com/) installed
- [VirtualBox](https://www.virtualbox.org/) or another Vagrant provider

### Option 2: Cloud Deployment (Multipass, AWS, Azure, etc.)
- Cloud provider CLI tools
- `cloud-init` support on the target platform

## Configuration

### Environment Variables

Before deployment, customize the following variables in either `Vagrantfile` or `cloudInit.yaml`:

```bash
# Database credentials
SQL_USER="flaskuser"
SQL_PASS="flaskpass"
SQL_ROOT_PASS="rootpass"

# Redis credentials
REDIS_USER="flaskuser"
REDIS_PASS="flaskpass"

# Flask secret key - Used for session signing, CSRF protection, and cryptographic operations
# IMPORTANT: Generate a secure random key for production!
SECRET_KEY="<generate-your-own-secret-key>"
```

**What is SECRET_KEY?**
- Flask uses this key to secure cookies and sessions.
- It prevents users from tampering with their login session or other sensitive data.
- **Important**: Never share this key or commit it to GitHub—anyone with it can impersonate users.

**Generate a secure SECRET_KEY:**
```bash
python3 -c "import secrets; print(secrets.token_urlsafe(48))"
```

## Deployment

### Method 1: Vagrant

1. **Edit credentials** in `Vagrantfile`:
   - Modify `SQL_USER`, `SQL_PASS`, `REDIS_USER`, `REDIS_PASS`
   - **Generate and replace `SECRET_KEY`** using the command above

2. **Start the VM:**
   ```bash
   vagrant up
   ```

3. **Access the application:**
   - Open browser: `https://localhost:8443`
   - Accept the self-signed certificate warning

4. **SSH into the VM:**
   ```bash
   vagrant ssh
   ```

### Method 2: Cloud-Init (Multipass Example)

1. **Edit credentials** in `cloudInit.yaml`:
   - Modify `SQL_USER`, `SQL_PASS`, `REDIS_USER`, `REDIS_PASS`
   - **Generate and replace `SECRET_KEY`** using the command above

2. **Launch with Multipass:**
   ```bash
   multipass launch -n <vm_name> --cloud-init cloudInit.yaml
   ```

3. **Get VM IP:**
   ```bash
   multipass list
   ```

4. **Access the application:**
   - Open browser: `https://<VM-IP>`
   - Accept the self-signed certificate warning

## Architecture

```
Browser → Nginx (HTTPS:443) → Gunicorn (Unix Socket) → Flask App
                                                        ↓
                                                    MySQL + Redis
```

### Components

- **Flask**: Web application framework
- **Gunicorn**: WSGI HTTP server (3 workers)
- **Nginx**: Reverse proxy with TLS 1.2/1.3
- **MySQL**: Database storage (utf8mb4)
- **Redis**: Session store and caching
- **OpenSSL**: Self-signed ECC certificate (prime256v1)

## Security Features

- TLS 1.2/1.3 encryption
- ECC certificates (prime256v1 curve)
- MySQL user with limited privileges
- Redis password authentication
- Unix socket communication between Nginx and Gunicorn

## File Structure

```
simpleForum/
├── app.py              # Main Flask application
├── wsgi.py             # WSGI entry point
├── requirements.txt    # Python dependencies
├── static/             # CSS 
├── templates/          # HTML templates
└── logs/               # Application logs
```

## Management Commands

### Check Service Status
```bash
sudo systemctl status app.service
sudo systemctl status nginx
sudo systemctl status mysql
sudo systemctl status redis-server
```

### View Logs
```bash
# Application logs
tail -f ~/simpleForum/logs/error.log
tail -f ~/simpleForum/logs/access.log

# Nginx logs
sudo tail -f /var/log/nginx/error.log

### Restart Services
```bash
sudo systemctl restart app.service
sudo systemctl restart nginx
```