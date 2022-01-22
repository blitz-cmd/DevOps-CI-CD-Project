# Ubuntu 20.04 LTS Setup
## Java Installation
```
$ apt install openjdk-11-jre-headless -y
```
## Jenkin Installation
### Port:8080
```
$ apt install openjdk-11-jre-headless -y
$ wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -
$ sudo sh -c 'echo deb http://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
$ sudo apt update
$ sudo apt install jenkins -y
$ sudo systemctl start jenkins
$ sudo systemctl enable jenkins
$ sudo systemctl status jenkins 
```
## Sonar Installation
### Port:9000
### t2.medium
### Pre-requisite
#### 1)Install OpenJDK-11
```
$ apt install openjdk-11-jre-headless
```
#### 2)Install & Configure PostgreSQL
```
$ sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" >> /etc/apt/sources.list.d/pgdg.list'
$ wget -q https://www.postgresql.org/media/keys/ACCC4CF8.asc -O - | sudo apt-key add -
$ sudo apt install postgresql postgresql-contrib -y
$ sudo systemctl enable postgresql
$ sudo systemctl start postgresql
$ sudo passwd postgres
$ su - postgres
$ createuser sonar
$ psql
$ ALTER USER sonar WITH ENCRYPTED password 'my_strong_password';
$ CREATE DATABASE sonarqube OWNER sonar;
$ GRANT ALL PRIVILEGES ON DATABASE sonarqube to sonar;
$ \q
$ exit
```
#### 3)Download & Install SonarQube
```
$ sudo apt-get install zip -y
```
#### Sonar version Number:https://www.sonarqube.org/downloads/
```
$ sudo wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-<VERSION_NUMBER>.zip
$ sudo unzip sonarqube-<VERSION_NUMBER>.zip
$ sudo mv sonarqube-<VERSION_NUMBER> /opt/sonarqube
```
#### 4)Add SonarQube Group and User
```
$ sudo groupadd sonar
$ sudo useradd -d /opt/sonarqube -g sonar sonar
$ sudo chown sonar:sonar /opt/sonarqube -R
```
#### 5)Configure SonarQube
```
$ sudo nano /opt/sonarqube/conf/sonar.properties
```
#### Add following lines/replace & save it
```
sonar.jdbc.username=sonar
sonar.jdbc.password=<my_strong_password>
sonar.jdbc.url=jdbc:postgresql://localhost:5432/sonarqube
```
```
$ sudo nano /opt/sonarqube/bin/linux-x86-64/sonar.sh
```
#### Add following line/replace & save it
```
RUN_AS_USER=sonar
```
#### 6)Setup Systemd service
```
$ sudo nano /etc/systemd/system/sonar.service
```
#### Add following lines & save it
```
[Unit]
Description=SonarQube service
After=syslog.target network.target

[Service]
Type=forking

ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop

User=sonar
Group=sonar
Restart=always

LimitNOFILE=65536
LimitNPROC=4096

[Install]
WantedBy=multi-user.target
```
```
$ sudo systemctl enable sonar
$ sudo systemctl start sonar
$ sudo systemctl status sonar
```
#### 7)Modify Kernel System Limits
```
$ sudo nano /etc/sysctl.conf
```
#### Add following lines & save it
```
vm.max_map_count=262144
fs.file-max=65536
ulimit -n 65536
ulimit -u 4096
```
```
$ sudo reboot
```
OR
```
$ docker run -d -p 9000:9000 sonarqube:lts
```
## Docker Installation
```
$ sudo apt-get update
$ sudo apt-get install ca-certificates curl gnupg lsb-release
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
$ echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
$ sudo apt-get update
$ sudo apt-get install docker-ce docker-ce-cli containerd.io -y
```
## Nexus Installation
### Port:8081
### t2.medium
#### 1)Installation
```
$ apt update && apt upgrade -y
$ apt-get install openjdk-8-jdk -y
$ useradd -M -d /opt/nexus -s /bin/bash -r nexus
$ echo "nexus ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/nexus
$ mkdir /opt/nexus
$ wget https://sonatype-download.global.ssl.fastly.net/repository/downloads-prod-group/3/nexus-3.29.2-02-unix.tar.gz
$ tar xzf nexus-3.29.2-02-unix.tar.gz -C /opt/nexus --strip-components=1
$ chown -R nexus:nexus /opt/nexus
$ nano /opt/nexus/bin/nexus.vmoptions
```
#### Add following lines/replace & save it
```
-Xms1024m
-Xmx10243m
-XX:MaxDirectMemorySize=1024m
-XX:+UnlockDiagnosticVMOptions
-XX:+LogVMOutput
-XX:LogFile=../sonatype-work/nexus3/log/jvm.log
-XX:-OmitStackTraceInFastThrow
-Djava.net.preferIPv4Stack=true
-Dkaraf.home=.
-Dkaraf.base=.
-Dkaraf.etc=etc/karaf
-Djava.util.logging.config.file=etc/karaf/java.util.logging.properties
-Dkaraf.data=./sonatype-work/nexus3
-Dkaraf.log=./sonatype-work/nexus3/log
-Djava.io.tmpdir=./sonatype-work/nexus3/tmp
-Dkaraf.startLocalConsole=false
```
```
$ nano /opt/nexus/bin/nexus.rc
```
#### Add following lines/replace & save it
```
run_as_user="nexus"
```
#### 2)Create a Systemd Service File for Nexus
```
$ nano /etc/systemd/system/nexus.service
```
#### Add following lines/replace & save it
```
[Unit]
Description=nexus service
After=network.target

[Service]
Type=forking
LimitNOFILE=65536
ExecStart=/opt/nexus/bin/nexus start
ExecStop=/opt/nexus/bin/nexus stop
User=nexus
Restart=on-abort

[Install]
WantedBy=multi-user.target
```
```
$ sudo systemctl daemon-reload
$ sudo systemctl start nexus
$ sudo systemctl enable nexus
$ sudo systemctl status nexus
```
## Kubernetes Installation
#### Instance: t2.medium, EC2 & Ubuntu 20.04 LTS
#### Run commands as normal user (ubuntu)
#### 1)Turn off swap memory
```
$ swapoff -a
$ sudo reboot
```
#### 2)Install Docker
```
$ sudo apt update
$ sudo apt install docker.io -y
$ sudo systemctl status docker.service
$ sudo systemctl enable docker.service
```
#### 3)Install Kubernetes
```
$ sudo apt install apt-transport-https
$ sudo apt install curl
$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
$ sudo apt-add-repository "deb https://apt.kubernetes.io/ kubernetes-xenial main"
$ sudo apt install kubelet kubeadm kubectl
$ sudo apt-mark hold kubelet kubeadm kubectl
```
```
$ sudo sed -i "s/cgroup-driver=systemd/cgroup-driver=cgroupfs/g" /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
$ sudo systemctl daemon-reload
$ sudo systemctl restart kubelet
```
#### 4)Command to prevent kubeadm init error
#### https://stackoverflow.com/questions/52119985/kubeadm-init-shows-kubelet-isnt-running-or-healthy
```
$ sudo nano /etc/docker/daemon.json
{
    "exec-opts": ["native.cgroupdriver=systemd"]
}
```
```
$ sudo systemctl daemon-reload
$ sudo systemctl restart docker
$ sudo systemctl restart kubelet
```
#### 5)Run this command in K8-Master Node
```
$ sudo kubeadm init --pod-network-cidr=192.168.10.0/24
```
#### Note: In the above step you will get below command as output. Run this command in K8-Slave Node after Step 6
```
Ex: $ kubeadm join 172.31.84.61:6443 --token ek3958.jwy7ulgwt81xzr2i \
--discovery-token-ca-cert-hash sha256:5c0453b633092c1a9109003e5bdc7985f6f2692830dd3905810061a28626971d
```
#### 6)Run this command in K8-Master Node & add network model for K8
```
$ sudo mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
$ kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```
#### 7)Run kubeadm join command from above
#### 8)Run this command in K8-Master Node to label K8-Slave node
```
$ kubectl label node ip-172-31-92-175 node-role.kubernetes.io/worker=worker
$ kubectl get nodes
```
## Helm Installation
```
$ curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 -o helm.sh
$ chmod 700 helm.sh
$ ./helm.sh
```
## Datree Installation
```
Refer Datree Documentation: https://hub.datree.io/
$ helm plugin install https://github.com/datreeio/helm-datree
```