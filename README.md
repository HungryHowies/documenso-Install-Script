It’s a one-shot installer that sets up the entire Documenso stack (Node.js + PostgreSQL + systemd service) with minimal manual steps.
### NOTE: NGINX is not installed during this setup.
Tested on Ubuntu 22.0.4

🛠️ Main Tasks Performed in Script.

### Pre-checks & Dependencies

* Installs Git if missing.
* Ensures Node.js v22 is installed (replaces older versions if needed).
* Checks that npm is available.
* Installs system tools: vim, sendmail, curl, build-essential, etc.

### User Input & Configuration

* Asks user for:
  * Installation directory (/opt/documenso default).
  * Server IP (used in WEBAPP_URL).
  * Database name, user, and password.
  * NEXTAUTH_SECRET (used for session security).
  * SMTP port (defaults to 25).
* Shows a configuration summary before continuing.

### PostgreSQL Setup

* Installs PostgreSQL.
* Creates the Documenso database and user, and sets their password.

### Download & Configure Documenso

* Clones the Documenso GitHub repo into /opt/documenso.
* Copies .env.example → .env.
* Configures .env with:
  * Database URL.
  * WebApp URL.
  * SMTP settings (comments out username/password if not provided).
Prepares .env.local (empty, but ready for overrides).

### Build Application

* Runs npm install for dependencies.
* Builds Documenso (npm run build).
* Applies database migrations with Prisma.

## Optional Signing Certificate

* If a signing cert exists, it updates .env with its path.

### Systemd Service Setup

* Creates a systemd unit file for Documenso.
* Ensures it starts at boot.
* Runs the app under production mode using npx dotenv.

### Service Startup

* Reloads systemd.
* Starts Documenso as a background service.
* Shows service status.
* Drops you into the Documenso working directory.

## Documenso Install Script.
```
#!/usr/bin/env bash
set -euo pipefail

# ------------------------------
# Documenso 1.10_rc.5 Automated Installer
# ------------------------------

# Install Git if missing
if ! command -v git >/dev/null 2>&1; then
    echo "Git not found. Installing Git..."
    sudo apt update
    sudo apt install -y git
else
    echo "Git is already installed: $(git --version)"
fi

# Check Node.js version and install if missing or wrong
INSTALL_NODE=false
if command -v node >/dev/null 2>&1; then
    NODE_VERSION=$(node -v | sed 's/v//')
    MAJOR_VERSION=${NODE_VERSION%%.*}
    if [ "$MAJOR_VERSION" -ne 22 ]; then
        echo "Node.js v22 is required. Detected version: $NODE_VERSION"
        INSTALL_NODE=true
    else
        echo "Node.js version $NODE_VERSION detected. OK."
    fi
else
    echo "Node.js not found. Will install Node.js v22."
    INSTALL_NODE=true
fi

if [ "$INSTALL_NODE" = true ]; then
    echo "Installing Node.js v22..."
    sudo mkdir -p /etc/apt/keyrings
    curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key | sudo gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg
    echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_22.x nodistro main" | sudo tee /etc/apt/sources.list.d/nodesource.list
    sudo apt update
    sudo apt install -y nodejs
fi

# Check for npm AFTER Node.js installation
command -v npm >/dev/null 2>&1 || { echo "npm is required but not installed. Aborting."; exit 1; }

echo "Node version: $(node -v)"
echo "npm version: $(npm -v)"

# ------------------------------
# Variables & User Input with Defaults
# ------------------------------

# Default installation directory
DEFAULT_DOCUMENSO_DIR="/opt/documenso"
read -rp "Enter installation directory [default: $DEFAULT_DOCUMENSO_DIR]: " DOCUMENSO_DIR
DOCUMENSO_DIR=${DOCUMENSO_DIR:-$DEFAULT_DOCUMENSO_DIR}
ENV_FILE="$DOCUMENSO_DIR/.env"

# Auto-detect local IP address
LOCAL_IP=$(hostname -I | awk '{print $1}')

# Prompt for WebApp IP with default
read -rp "Enter the IP for the WebApp URL [default: $LOCAL_IP]: " USER_IP
WEBAPP_IP=${USER_IP:-$LOCAL_IP}
WEBAPP_URL="http://$WEBAPP_IP:3000"

# Prompt for database name
read -rp "Enter Documenso database name [default: documenso]: " DB_NAME
DB_NAME=${DB_NAME:-documenso}

# Prompt for database user
read -rp "Enter Documenso database user [default: documenso]: " DB_USER
DB_USER=${DB_USER:-documenso}

# Prompt for database password (hidden input)
read -rsp "Enter password for database user $DB_USER [default: password]: " DB_PASS
DB_PASS=${DB_PASS:-password}
echo

# Prompt for NEXTAUTH_SECRET
read -rp "Enter NEXTAUTH_SECRET [default: secret]: " NEXTAUTH_SECRET
NEXTAUTH_SECRET=${NEXTAUTH_SECRET:-secret}

# Prompt for SMTP port
read -rp "Enter SMTP port [default: 25]: " SMTP_PORT
SMTP_PORT=${SMTP_PORT:-25}

# Node heap size
NODE_HEAP_SIZE=4096

# Display configuration summary
echo
echo "---------------------------------------"
echo "Configuration Summary:"
echo "Installation Directory: $DOCUMENSO_DIR"
echo "WEBAPP_URL: $WEBAPP_URL"
echo "DB_NAME: $DB_NAME"
echo "DB_USER: $DB_USER"
echo "DB_PASS: ${DB_PASS//?/*}"  # mask password
echo "NEXTAUTH_SECRET: $NEXTAUTH_SECRET"
echo "SMTP_PORT: $SMTP_PORT"
echo "NODE_HEAP_SIZE: $NODE_HEAP_SIZE"
echo "---------------------------------------"
echo

# ------------------------------
# System Update & Prerequisites
# ------------------------------
echo "Updating system and installing prerequisites..."
sudo apt update && sudo apt upgrade -y
sudo timedatectl set-timezone America/Chicago
sudo apt install -y net-tools plocate vim git sendmail ca-certificates curl gnupg build-essential

# ------------------------------
# Install Node.js v22
# ------------------------------
echo "Installing Node.js v22..."
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key | sudo gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg
echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_22.x nodistro main" | sudo tee /etc/apt/sources.list.d/nodesource.list
sudo apt update
sudo apt install -y nodejs
echo "Node version: $(node --version)"

# ------------------------------
# Install PostgreSQL 14
# ------------------------------
echo "Installing PostgreSQL..."
sudo apt install -y postgresql postgresql-contrib

echo "Configuring PostgreSQL user and database..."
sudo -u postgres psql <<EOF
CREATE ROLE $DB_USER LOGIN;
CREATE DATABASE $DB_NAME;
GRANT CONNECT, CREATE ON DATABASE $DB_NAME TO $DB_USER;
ALTER USER $DB_USER PASSWORD '$DB_PASS';
\q
EOF

# ------------------------------
# Download Documenso
# ------------------------------
echo "Cloning Documenso repository into $DOCUMENSO_DIR..."
sudo mkdir -p "$DOCUMENSO_DIR"
sudo chown "$USER":"$USER" "$DOCUMENSO_DIR"
git clone https://github.com/documenso/documenso.git "$DOCUMENSO_DIR"
cd "$DOCUMENSO_DIR"
cp .env.example .env

# Ensure .env.local exists
touch "$DOCUMENSO_DIR/.env.local"

# ------------------------------
# Configure .env
# ------------------------------
echo "Configuring .env file..."

sed -i "s|NEXTAUTH_SECRET=.*|NEXTAUTH_SECRET=\"$NEXTAUTH_SECRET\"|" "$ENV_FILE"
sed -i "s|NEXT_PUBLIC_WEBAPP_URL=.*|NEXT_PUBLIC_WEBAPP_URL=\"$WEBAPP_URL\"|" "$ENV_FILE"
sed -i "s|NEXT_PRIVATE_DATABASE_URL=.*|NEXT_PRIVATE_DATABASE_URL=\"postgres://$DB_USER:$DB_PASS@127.0.0.1:5432/$DB_NAME\"|" "$ENV_FILE"
sed -i "s|NEXT_PRIVATE_DIRECT_DATABASE_URL=.*|NEXT_PRIVATE_DIRECT_DATABASE_URL=\"postgres://$DB_USER:$DB_PASS@127.0.0.1:5432/$DB_NAME\"|" "$ENV_FILE"
sed -i "s|NEXT_PRIVATE_SMTP_PORT=.*|NEXT_PRIVATE_SMTP_PORT=$SMTP_PORT|" "$ENV_FILE"

# Handle optional SMTP username
if [ -n "$SMTP_USER" ]; then
  sed -i "s|.*NEXT_PRIVATE_SMTP_USERNAME.*|NEXT_PRIVATE_SMTP_USERNAME=\"$SMTP_USER\"|" "$ENV_FILE"
else
  sed -i "s|.*NEXT_PRIVATE_SMTP_USERNAME.*|# NEXT_PRIVATE_SMTP_USERNAME=|" "$ENV_FILE"
fi

# Handle optional SMTP password
if [ -n "$SMTP_PASS" ]; then
  sed -i "s|.*NEXT_PRIVATE_SMTP_PASSWORD.*|NEXT_PRIVATE_SMTP_PASSWORD=\"$SMTP_PASS\"|" "$ENV_FILE"
else
  sed -i "s|.*NEXT_PRIVATE_SMTP_PASSWORD.*|# NEXT_PRIVATE_SMTP_PASSWORD=|" "$ENV_FILE"
fi


# ------------------------------
# Set Node.js Heap Size
# ------------------------------
export NODE_OPTIONS="--max_old_space_size=$NODE_HEAP_SIZE"

# ------------------------------
# Install Dependencies & Build
# ------------------------------
echo "Installing npm dependencies..."
npm install

echo "Building Documenso..."
npm run build
npm run prisma:migrate-deploy

# ------------------------------
# Configure Signing Certificate (optional)
# ------------------------------
SIGN_CERT="$DOCUMENSO_DIR/apps/remix/example/cert.p12"

if [ -f "$SIGN_CERT" ]; then
  echo "Configuring signing certificate..."
  sed -i "s|NEXT_PRIVATE_SIGNING_LOCAL_FILE_PATH=.*|NEXT_PRIVATE_SIGNING_LOCAL_FILE_PATH=$SIGN_CERT|" "$ENV_FILE"
  sed -i "s|NEXT_PRIVATE_SIGNING_PASSPHRASE=.*|NEXT_PRIVATE_SIGNING_PASSPHRASE=\"\"|" "$ENV_FILE"
else
  echo "No signing certificate found. Skipping certificate configuration."
  # Comment out the variables in .env to disable signing
  sed -i "s|^NEXT_PRIVATE_SIGNING_LOCAL_FILE_PATH=.*|# NEXT_PRIVATE_SIGNING_LOCAL_FILE_PATH=|" "$ENV_FILE"
  sed -i "s|^NEXT_PRIVATE_SIGNING_PASSPHRASE=.*|# NEXT_PRIVATE_SIGNING_PASSPHRASE=|" "$ENV_FILE"
fi

# ------------------------------
# Create systemd Service for Documenso
# ------------------------------
SERVICE_NAME="documenso"
SERVICE_FILE="/etc/systemd/system/$SERVICE_NAME.service"

echo "Creating systemd service for Documenso..."

sudo tee "$SERVICE_FILE" > /dev/null <<EOF
[Unit]
Description=Documenso Service
After=network.target postgresql.service

[Service]
Type=simple
User=$USER
WorkingDirectory=$DOCUMENSO_DIR/apps/remix
Environment=NODE_OPTIONS=--max_old_space_size=$NODE_HEAP_SIZE
# Load environment variables from .env and .env.local
ExecStart=$(which npx) dotenv -e $DOCUMENSO_DIR/.env -e $DOCUMENSO_DIR/.env.local -- cross-env NODE_ENV=production node build/server/main.js
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
EOF

echo "Reloading systemd daemon..."
sudo systemctl daemon-reload

echo "Enabling Documenso service to start on boot..."
sudo systemctl enable "$SERVICE_NAME"

echo "Starting Documenso service..."
sudo systemctl start "$SERVICE_NAME"

echo "Documenso service status:"
sudo systemctl status "$SERVICE_NAME" --no-pager

# ------------------------------
# Start Documenso
# ------------------------------
echo "Starting Documenso..."
cd "$DOCUMENSO_DIR/apps/remix"
```
