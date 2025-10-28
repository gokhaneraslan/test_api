#!/bin/bash

# Stop the script on error
set -e

# Update system packages and install essential tools
sudo apt-get update
sudo apt-get install -y nginx python3-pip python3-venv git

# Enable and start NGINX
sudo systemctl enable nginx
sudo systemctl start nginx

# Clone the app code (OR download it from a Cloud Storage bucket)
# NOTE: Write your own project's repo address here
git clone https://github.com/gokhaneraslan/test_api.git /home/ubuntu/app

cd /home/ubuntu/app

# Create and activate the Python virtual environment
python3 -m venv venv
source venv/bin/activate

# Install necessary Python packages
pip install "fastapi[all]" gunicorn

# Set NGINX Configuration
# Delete the default nginx config file
sudo rm /etc/nginx/sites-enabled/default

# Create our own config file
sudo tee /etc/nginx/sites-available/fastapi_app <<'EOF'
server {
    listen 80;
    server_name _; # Reply to all domains

    location / {
        # Redirect incoming requests to Gunicorn running internally on port 8000
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # If you have static files you can use this block
    # location /static {
    #     alias /home/ubuntu/app/static;
    # }
}
EOF

# Activate the config we created
sudo ln -s /etc/nginx/sites-available/fastapi_app /etc/nginx/sites-enabled/

# Apply settings by restarting Nginx
sudo systemctl restart nginx

# Create a systemd service to run the app in the background with Gunicorn
# This will ensure the app starts automatically even if the server restarts
sudo tee /etc/systemd/system/fastapi_app.service <<'EOF'
[Unit]
Description=Gunicorn instance to serve FastAPI app
After=network.target

[Service]
User=ubuntu
Group=www-data
WorkingDirectory=/home/ubuntu/app
Environment="PATH=/home/ubuntu/app/venv/bin"
ExecStart=/home/ubuntu/app/venv/bin/gunicorn --workers 3 --worker-class uvicorn.workers.UvicornWorker -b 0.0.0.0:8000 main:app

[Install]
WantedBy=multi-user.target
EOF

# Enable and start the new service
sudo systemctl daemon-reload
sudo systemctl enable fastapi_app
sudo systemctl start fastapi_app
