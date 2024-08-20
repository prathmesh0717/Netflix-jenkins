# $${\color{red} \textbf{Project : Netflix}}$$

${\color{purple} \textbf{API KEY}}$
````
079c53c7a0369363ae29016c9c3b29f6
````

## ${\color{red} \textbf{Phase 1: Initial Setup and Deployment}}$

${\color{green} \textbf{1. Generate Token From TMDB}}$

$\color{green} \textbf{2. Launch ec2 instance}$
- Ubuntu
- t2.medium
- storage : 40
- SG : All traffic

$\color{green} \textbf{3. Git Clone }
````
git clone <repo>
````
$\color{green} \textbf{4. Install Docker and set up}$
````
sudo apt update 
sudo apt install docker.io -y
sudo systemctl start docker
sudo usermod -aG docker ubuntu
newgrp docker
sudo chmod 777 /var/run/docker.sock
````
$\color{green} \textbf{5. Build the image with TBDM API and run the container}$
````
docker build --build-arg TMDB-V3-API-KEY=<your-api-key> -t netflix .
docker run -d --name netflix -p 8081:80 netflix:latest
````
${\color{green} \textbf{6. Host : pub-ip:8081}}$

![image](https://github.com/user-attachments/assets/5930247a-559a-4149-9f0f-effa41be5f5c)



## ${\color{red} \textbf{Phase 2: Security}}$

### $\color{purple} \textbf{Install SonarQube and Trivy for Scan }$

$\color{green} \textbf{1. SonarQube for Code Testing with Direct pulling Image}$

````
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
````
$\color{green} \textbf{ 2. To Access SonarQube : pub-ip:9000}$

$\color{red} \textbf{ NOTE : }$ SonarQube have by default username and password is admin.


$\color{green} \textbf{ 3. Generate Token in SonarQube; it will need us in jenkins }$
- My Account  → Security → Token name  → Generate Token → Copy that Token


$\color{green} \textbf{ 4. Install Trivy for Scan Image}$

````
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy
````
$\color{green} \textbf{5. Scan Netflix Image}$
````
trivy image <imageid>
````

## ${\color{red} \textbf{Phase 3: CI/CD Setup}}$

$\color{green} \textbf{1. Install Jenkins for Automate Deployment}$
````
sudo apt update
sudo apt install fontconfig openjdk-17-jre
java -version
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian/jenkins.io-2023.key
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins
````
$\color{green} \textbf{2. To Access Jenkins : pub-ip:8080}$
- Add initial password
- Create user with password
- Start using jenkins

$\color{green} \textbf{3. Install Necessary Plugins}$

Goto Manage Jenkins → Plugins → Available Plugins → Install below plugins

1. Eclipse Temurin Installer 
2. SonarQube Scanner 
3. NodeJs 
4. Email Extension 

$\color{green} \textbf{4. Set up Plugins in Tool}$

Goto Manage Jenkins → Tool → 
- Add Node name : node16 / version : 16.16.0
- Add java name : jdk17 / version : 17.0.1+12 (automatic installation)
- Add SonarQube Scanner Name : sonar-scanner 

$\color{green} \textbf{5. SONARQUBE : Create Token}$

Jenkins Dashboard → Manage Jenkins → Credentials → Add Secret Text → Name : sonar-token → Add token which created  in SonarQube → apply&save

$\color{green} \textbf{6. The Configure System option}$

jenkins Dashboard → Manage jenkins → System

- Add jenkins installation → Add <pub_ip>:8080
- Add SonarQube installation → (sonar-scanner) → Add <pub_ip>:9000

$\color{green} \textbf{ 7. Add pipeline code }$
`pipeline`
````
pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {
        stage('Code Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/abhipraydhoble/netflix.git'
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-scanner') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectName=Netflix \
                    -Dsonar.projectKey=Netflix'''
                }
            }
       }
       
        stage('Quality-gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }
        
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        
         stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        
        stage('Docker-build') {
            steps {
                sh "docker build --build-arg TMDB_V3_API_KEY=079c53c7a0369363ae29016c9c3b29f6   -t netflix:v1 ."
            }
        }
        
        stage("TRIVY"){
            steps {
                sh "trivy image netflix:v1 > trivyimage.txt" 
            }
        }
        
        stage ('docker run') {
            steps {
                sh 'docker run -itd --name netflix -p 81:80 netflix:v1'
            }
        }
    }
}
````

$\color{red} \textbf\{NOTE :}$ Install Dependency-Check. (npm install).

$\color{green} \textbf{8. Install Docker Tools and Docker Plugins}$

jenkins Dashboard → Manage jenkins → Plugin → Available Plugin → install below plugin 

- Docker
- Docker Commons
- Docker Pipeline
- Docker API
- docker-build-step
- Prometheus metrics


$\color{red} \textbf\{NOTE :}$ Add DockerHub Credential if You want to push Image on DockerHub Registry.

jenkins Dashboard → Manage jenkins → Credential → global → Add credential

Generate syntax → withcredential:username and password variable → add "uname" and "passwd" variable → add docker credential.

## ${\color{red} \textbf{Phase 4: Monitoring}}$

$\color{purple} \textbf{Install Prometheus and Grafana for Monitor Application}$

## $\color{brown} \textbf{Install Prometheus}$

$\color{green} \textbf{1. First, create a dedicated Linux user for Prometheus and download Prometheus:}$
````
sudo useradd --system --no-create-home --shell /bin/false prometheus
wget https://github.com/prometheus/prometheus/releases/download/v2.47.1/prometheus-2.47.1.linux-amd64.tar.gz
````
$\color{green} \textbf{2. Extract Prometheus files, move them, and create directories:}$
````
tar -xvf prometheus-2.47.1.linux-amd64.tar.gz
cd prometheus-2.47.1.linux-amd64/
sudo mkdir -p /data /etc/prometheus
sudo mv prometheus promtool /usr/local/bin/
sudo mv consoles/ console_libraries/ /etc/prometheus/
sudo mv prometheus.yml /etc/prometheus/prometheus.yml
````
$\color{green} \textbf{3. Set ownership for directories:}$
````
sudo chown -R prometheus:prometheus /etc/prometheus/ /data/
````
$\color{green} \textbf{4. Create a systemd unit configuration file for Prometheus:}$
````
sudo nano /etc/systemd/system/prometheus.service
````
$\color{green} \textbf{5. Add the following content to the prometheus.service file:}$
`prometheus.service`
````
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=prometheus
Group=prometheus
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/data \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.listen-address=0.0.0.0:9090 \
  --web.enable-lifecycle

[Install]
WantedBy=multi-user.target
````
$\color{brown} \textbf{Here's a brief explanation of the key parts in this prometheus.service file:}$

   - `User` and `Group` specify the Linux user and group under which Prometheus will run.

   - `ExecStart` is where you specify the Prometheus binary path, the location of the configuration file (`prometheus.yml`), the storage directory, and other settings.

   - `web.listen-address` configures Prometheus to listen on all network interfaces on port 9090.

   - `web.enable-lifecycle` allows for management of Prometheus through API calls.


$\color{green} \textbf{6. start Prometheus.}$
````
sudo systemctl start prometheus
````
````
sudo systemctl status prometheus
````
$\color{green} \textbf{7. Access Prometheous : <pub-ip>:9090}$

## $\color{brown} \textbf{Install Node Exporter}$

$\color{green} \textbf{1. Create a system user for Node Exporter and download Node Exporter:}$
````
sudo useradd --system --no-create-home --shell /bin/false node_exporter
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
````
$\color{green} \textbf{2. Extract Node Exporter files, move the binary, and clean up:}$
````
tar -xvf node_exporter-1.6.1.linux-amd64.tar.gz
sudo mv node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/
rm -rf node_exporter*
````
$\color{green} \textbf{3. Create a systemd unit configuration file for Node Exporter:}$
````
sudo nano /etc/systemd/system/node_exporter.service
````
$\color{green} \textbf{4. Add the following content to the node-exporter.service file:}$
`node_exporter.servic`
````
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=node_exporter
Group=node_exporter
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/node_exporter --collector.logind

[Install]
WantedBy=multi-user.target
````
$\color{green} \textbf{5. Start Node Exporter}$
````
sudo systemctl start node_exporter
````
````
sudo systemctl status node_exporter
````
$\color{green} \textbf{6. Configure Prometheus Plugin Integration:}$

````
sudo nano /etc/prometheus/prometheus.yml
````
`prometheus.yml`
````
# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing multiple endpoints to scrape:
scrape_configs:
  # The job name is added as a label job=<job_name> to any timeseries scraped from this config.
  - job_name: "prometheus"
    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node_exporter"
    static_configs:
      - targets: ["localhost:9100"]

  - job_name: "jenkins"
    metrics_path: "/prometheus"
    static_configs:
      - targets: ["3.83.236.123:8080"]
````
$\color{red} \textbf{NOTE :}$ Add Jenkins-ip:jenkins-port (pub-ip:8080)

$\color{green} \textbf{7. Check the validity of the configuration file.}$
````
promtool check config /etc/prometheus/prometheus.yml
````
$\color{green} \textbf{8. Reload the Prometheus configuration without restarting:}$
````
curl -X POST http://localhost:9090/-/reload
````
$\color{green} \textbf{9. You can access Prometheus targets at:}$

`http://<your-prometheus-ip>:9090/targets`


## $\color{brown} \textbf{Install Grafana}$

$\color{green} \textbf{1. Install Dependencies:}$
````
sudo apt-get update
sudo apt-get install -y apt-transport-https software-properties-common
````
$\color{green} \textbf{2. Add the GPG key for Grafana:}$
````
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
````
$\color{green} \textbf{3. Add the repository for Grafana stable releases:}$
````
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
````
$\color{green} \textbf{4. Update the package list and install Grafana:}$
````
sudo apt-get update
sudo apt-get -y install grafana
````
$\color{green} \textbf{5. To automatically start Grafana after a reboot, enable the service and start service}$
````
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
````
````
sudo systemctl status grafana-server
````
$\color{green} \textbf{6. Access the Grafana}$

`<your-server-ip>:3000`

$\color{red} \textbf{The default username is "admin," and the default password is also "admin."}$

$\color{green} \textbf{7. Change the Default Password:}$

Grafana will prompt you to change the default password for security reasons. Follow the prompts to set a new password.

$\color{green} \textbf{8. Add Prometheus data source (on default)}$ 

- Add prometheus → click on data source
- on default option
- Add URL of Prometheus → pub-ip:9090
- save and test

$\color{green} \textbf{9. Import Dashboard}$
- Import Dashboard
- id : 1860 → Load
- Add promotheus data source (default)
- Import
