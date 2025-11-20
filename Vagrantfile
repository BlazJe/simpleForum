# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.define "app" do |config|

    config.vm.box = "ubuntu/jammy64"
    config.vm.network "forwarded_port", guest: 443, host: 8443
    config.vm.provision "shell", privileged: false, inline: <<-SHELL
      sudo apt update -y
      sudo apt install -y mysql-server

      SQL_USER="flaskuser"
      SQL_PASS="flaskpass"
      ROOT_SQL_PASS="rootpass"
      REDIS_USER="flaskuser"
      REDIS_PASS="flaskpass"
      SECRET_KEY="EnterYourSecretKeyHere"
      DATABASE_URL="mysql+pymysql://$SQL_USER:$SQL_PASS@127.0.0.1/flaskdb?charset=utf8mb4"
      REDIS_URL="redis://$REDIS_USER:$REDIS_PASS@127.0.0.1:6379/0"


      sudo mysql -e "ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '$ROOT_SQL_PASS'; FLUSH PRIVILEGES;"
      sudo mysql -uroot -p$ROOT_SQL_PASS -e "
CREATE DATABASE IF NOT EXISTS flaskdb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER IF NOT EXISTS '$SQL_USER'@'localhost' IDENTIFIED BY '$SQL_PASS';
GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, REFERENCES ON flaskdb.* TO '$SQL_USER'@'localhost';
FLUSH PRIVILEGES;
"

      sudo apt install -y redis-server
      sudo systemctl enable redis-server

      sudo redis-cli ACL SETUSER $REDIS_USER on "$REDIS_PASS" ~* +get +set +del +exists +expire +ttl
      sudo sed -i "s/^# requirepass .*/requirepass $REDIS_PASS/" /etc/redis/redis.conf
      sudo sed -i "s/^bind .*/bind 127.0.0.1/" /etc/redis/redis.conf
      sudo sed -i "s/^protected-mode no/protected-mode yes/" /etc/redis/redis.conf

      sudo systemctl restart redis-server
      sudo apt-get install -y git python3 python3-pip python3-venv
      cd ~
      git clone https://github.com/BlazJe/simpleForum.git
      cd simpleForum

      cat > .env <<EOF
SECRET_KEY="${SECRET_KEY}"
DATABASE_URL="${DATABASE_URL}"
REDIS_URL="${REDIS_URL}"
EOF

      python3 -m venv venv
      ./venv/bin/pip install --upgrade pip
      ./venv/bin/pip install -r requirements.txt

      sudo mkdir -p /home/vagrant/simpleForum/logs
      sudo chown -R vagrant:www-data /home/vagrant/simpleForum/logs
      sudo chmod 750 /home/vagrant/simpleForum/logs

      sudo bash -c 'cat > /etc/systemd/system/app.service <<EOL
[Unit]
Description=Gunicorn service for Flask simpleForum
After=network.target

[Service]
Restart=always
RestartSec=5
User=vagrant
Group=www-data
WorkingDirectory=/home/vagrant/simpleForum
Environment="PATH=/home/vagrant/simpleForum/venv/bin"
ExecStart=/home/vagrant/simpleForum/venv/bin/gunicorn \
    --workers 3 \
    --bind unix:/home/vagrant/simpleForum/gunicorn.sock \
    --access-logfile /home/vagrant/simpleForum/logs/access.log \
    --error-logfile /home/vagrant/simpleForum/logs/error.log \
    wsgi:app
[Install]
WantedBy=multi-user.target
EOL'

      sudo systemctl daemon-reload
      sudo systemctl enable app.service
      sudo systemctl start app.service

      sudo apt install -y nginx
      sudo systemctl enable nginx
      sudo mkdir -p /etc/nginx/ssl
      sudo openssl ecparam -genkey -name prime256v1 -out /etc/nginx/ssl/ecc.key
      sudo openssl req -new -x509 -key /etc/nginx/ssl/ecc.key -out /etc/nginx/ssl/ecc.crt -days 365 -subj "/CN=localhost"

      sudo bash -c 'cat > /etc/nginx/sites-available/simpleforum.conf <<EOL
server {
    listen 443 ssl;
    server_name localhost;

    ssl_certificate /etc/nginx/ssl/ecc.crt;
    ssl_certificate_key /etc/nginx/ssl/ecc.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ecdh_curve prime256v1;
    ssl_prefer_server_ciphers on;

    location / {
        include proxy_params;
        proxy_pass http://unix:/home/vagrant/simpleForum/gunicorn.sock;
    }
}
EOL'

      sudo ln -s /etc/nginx/sites-available/simpleforum.conf /etc/nginx/sites-enabled/
      sudo rm /etc/nginx/sites-enabled/default
      sudo chown -R vagrant:www-data /home/vagrant/simpleForum
      sudo chmod 711 /home/vagrant
      sudo chmod 750 /home/vagrant/simpleForum
      sudo systemctl restart nginx

    SHELL
  end
end