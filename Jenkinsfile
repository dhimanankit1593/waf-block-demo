pipeline {
   agent any

    environment {
        BANTIME_SECONDS = "600"   // Change here if needed
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/dhimanankit1593/waf-block-demo.git'
            }
        }

        stage('Calculate Block Time') {
            steps {
                script {
                    def sec = BANTIME_SECONDS.toInteger()
                    def min = sec / 60
                    def hr  = min / 60

                    if (hr >= 1) {
                        env.BLOCK_MESSAGE = "${hr} hour(s)"
                    } else {
                        env.BLOCK_MESSAGE = "${min} minute(s)"
                    }

                    echo "Fail2ban ban time (human readable): ${env.BLOCK_MESSAGE}"
                }
            }
        }

      stage('Install Apache2 & Deploy Website') {
    steps {
        sh '''
        sudo apt update -y
        sudo apt install apache2 -y

        # Disable default site to avoid port conflict
        sudo a2dissite 000-default.conf || true
        sudo a2dissite default-ssl.conf || true

        # Remove any existing Listen 80 lines
        sudo sed -i '/Listen 80/d' /etc/apache2/ports.conf

        # Add new port
        echo "Listen 8090" | sudo tee -a /etc/apache2/ports.conf

        # Create new VirtualHost file
        sudo bash -c 'cat > /etc/apache2/sites-available/waf-site.conf <<EOF
<VirtualHost *:8090>
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html
    ErrorLog \${APACHE_LOG_DIR}/error.log
    CustomLog \${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
EOF'

        # Enable new site
        sudo a2ensite waf-site.conf

        # Deploy website file
        sudo cp index.html /var/www/html/index.html

        sudo systemctl restart apache2
        sudo systemctl status apache2 --no-pager
        '''
    }
}

stage('Install & Configure Nginx WAF') {
    steps {
        sh '''
        sudo apt install nginx -y

        # Create limit_req_zone config in http block
        sudo bash -c 'cat > /etc/nginx/conf.d/req_limit.conf <<EOF
limit_req_zone $binary_remote_addr zone=req_limit:10m rate=5r/m;
EOF'

        # Create server block
        sudo bash -c 'cat > /etc/nginx/sites-available/waf.conf <<EOF
server {
    listen 8091;
    server_name _;

    # Custom block message
    error_page 429 /429.json;

    location /429.json {
        default_type application/json;
        return 429 "{\"message\": \"Your request has been blocked. Please wait ${BLOCK_MESSAGE} before trying again.\"}";
    }

    location / {
        limit_req zone=req_limit burst=5 nodelay;

        proxy_pass http://127.0.0.1:8090;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;

        access_log /var/log/nginx/access.log;
        error_log /var/log/nginx/error.log;
    }
}
EOF'

        # Enable waf site
        sudo ln -sf /etc/nginx/sites-available/waf.conf /etc/nginx/sites-enabled/waf.conf

        # Test and restart nginx
        sudo nginx -t
        sudo systemctl restart nginx
        sudo systemctl status nginx --no-pager
        '''
    }
}

        

        stage('Install & Configure Fail2ban') {
            steps {
                sh '''
                sudo apt install fail2ban -y

                # Jail config
                sudo bash -c 'cat > /etc/fail2ban/jail.d/nginx-limit.conf <<EOF
[nginx-limit]
enabled = true
port = 8091
filter = nginx-limit
logpath = /var/log/nginx/error.log
maxretry = 6
findtime = 60
bantime = ${BANTIME_SECONDS}
EOF'

                # Filter config
                sudo bash -c 'cat > /etc/fail2ban/filter.d/nginx-limit.conf <<EOF
[Definition]
failregex = limiting requests, excess: .* by zone.* client: <HOST>
EOF'

                sudo systemctl restart fail2ban
                '''
            }
        }

        stage('Verification') {
            steps {
                sh '''
                echo "===== SERVICE STATUS ====="
                sudo systemctl status apache2 --no-pager
                sudo systemctl status nginx --no-pager
                sudo systemctl status fail2ban --no-pager

                echo "Access your WAF at: http://<server-ip>:8091/"
                '''
            }
        }
    }

    post {
        always {
            echo "Deployment Completed Successfully!"
        }
    }
}
