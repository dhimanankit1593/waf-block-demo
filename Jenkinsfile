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

        stage('Install & Configure Nginx + Website') {
            steps {
                sh """
                    sudo apt update -y
                    sudo apt install nginx -y

                    # Deploy website
                    sudo mkdir -p /var/www/html
                    sudo cp index.html /var/www/html/index.html || true

                    # Nginx rate-limit config
                    sudo bash -c 'cat > /etc/nginx/conf.d/req_limit.conf <<EOF
limit_req_zone \$binary_remote_addr zone=req_limit:10m rate=5r/m;
EOF'

                    # Nginx server block (WAF + website)
                    sudo bash -c "cat > /etc/nginx/sites-available/waf.conf <<EOF
server {
    listen 8091;
    server_name _;

    root /var/www/html;

    index index.html;

    error_page 429 /429.json;

    location /429.json {
        default_type application/json;
        return 429 '{\"message\": \"Your request has been blocked. Please wait ${BLOCK_MESSAGE} before trying again.\"}';
    }

    location / {
        limit_req zone=req_limit burst=5 nodelay;
        try_files \$uri \$uri/ =404;
    }
}
EOF"

                    sudo ln -sf /etc/nginx/sites-available/waf.conf /etc/nginx/sites-enabled/waf.conf

                    sudo nginx -t
                    sudo systemctl restart nginx
                    sudo systemctl enable nginx
                """
            }
        }

        stage('Install & Configure Fail2ban') {
            steps {
                sh """
                    sudo apt install fail2ban -y

                    # Jail config
                    sudo bash -c "cat > /etc/fail2ban/jail.d/nginx-limit.conf <<EOF
[nginx-limit]
enabled = true
port = 8091
filter = nginx-limit
logpath = /var/log/nginx/error.log
maxretry = 6
findtime = 60
bantime = ${BANTIME_SECONDS}
EOF"

                    # Filter config
                    sudo bash -c 'cat > /etc/fail2ban/filter.d/nginx-limit.conf <<EOF
[Definition]
failregex = limiting requests, excess: .* by zone.* client: <HOST>
EOF'

                    sudo systemctl restart fail2ban
                """
            }
        }

        stage('Verification') {
            steps {
                sh """
                    echo "===== SERVICE STATUS ====="
                    sudo systemctl status nginx --no-pager
                    sudo systemctl status fail2ban --no-pager

                    echo "Access your website + WAF at: http://<server-ip>:8091/"
                """
            }
        }
    }

    post {
        always {
            echo "Deployment Completed Successfully!"
        }
    }
}
