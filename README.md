# Corda5MgmSetup
instruction on how to setup Corda5MGM 


Corda5 MGM Setup



Get tar file from customer hub 



check for previous corda docker images:
docker images --format="{{.Repository}}:{{.Tag}}"

To remove previous version example:

docker rmi corda5mgmsetup.azurecr.io/corda-ent-rest-worker:5.0.0.0



Installing Corda5

Installing the corda-ent-worker-images-5.0.0.tar

docker load -i <the corda worker images tar file>
	: docker load -i corda-ent-worker-images-5.0.0.tar





Create Azure Container Registry

login in to the registry

docker login <the ACR login server name>

Example
docker login corda5multicluster.azurecr.io



Create the push script shown below:

name file: push.sh

#!/usr/bin/bash
if [ -z "$1" ]; then
 echo "Specify target registry"
 exit 1
fi

declare -a images=(
 "rest-worker" "flow-worker"
 "member-worker" "p2p-gateway-worker"
 "p2p-link-manager-worker" "db-worker"
 "crypto-worker" "plugins" )
tag=5.0.0.0
cordaVersion=corda-ent-
target_registry=$1


for image in "${images[@]}"; do
 source=corda/${cordaVersion}${image}:${tag}
 target=$target_registry/${cordaVersion}${image}:${tag}
 echo "Publishing image $source to $target"
 docker tag $source $target
 docker push $target
done

docker tag postgres:latest $target_registry/postgres:latest
docker push $target_registry/postgres:latest


./push.sh <Your ACR login server name>



Setup using Minikube on AzureVM:



To Install minikube

prerequisite

docker

minikube



to install docker

URL for instructions on how to download and install docker:

# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add the repository to Apt sources:
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update


sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin




for more info see below:
https://docs.docker.com/engine/install/ubuntu/ 



Important to get docker to work with minikube without root privileges 

Access docker as non-root user:
sudo usermod -aG docker $USER
newgrp docker



Install minikube

curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube



for more info see below:

https://minikube.sigs.k8s.io/docs/start/ 



Use minikube kubectl

alias kubectl="minikube kubectl --"



Configure minikube

minikube config set cpus 8
minikube config set memory 32043
minikube config set driver docker

 

Start minikube:

minikube start



Create a namespace:
kubectl create namespace corda



ENSURE TO SET THE CREATED namespace as default:
kubectl config set-context --current --namespace=corda



Create secrets:

kubectl create secret -n corda docker-registry cred04 --docker-server "<your azure register server name>" \
--docker-username "<your Azure container registry username (Container registry username)>" --docker-password "<your Azure container registry password as shown in the >"



example

kubectl create secret -n corda docker-registry cred04 --docker-server http://corda5mgmsetup.azurecr.io  --docker-username Corda5MgmSetup --docker-password AcFmiPk7rKShjUy00qTYku+Cjva5VxkgJiInZSQwro+ACRCMtAY+





Install helm:

curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt update
sudo apt install helm



for more info:

https://helm.sh/docs/intro/install/ 



Install Kafka to minikube k8

helm repo add bitnami https://charts.bitnami.com/bitnami

create a kafka.yaml file

image:
  tag: 3.2.0-debian-11-r20
allowPlaintextListener: true
replicaCount: 1
autoCreateTopicsEnable: false
auth:
  enabled: true
  clientProtocol: sasl_tls
  interBrokerProtocol: tls
  tls:
    type: pem
    autoGenerated: true
authorizerClassName: "kafka.security.authorizer.AclAuthorizer"
zookeeper:
  startupProbe:
    enabled: true
resources:
  requests:
    memory: 1390Mi
    cpu: 1000m
  limits:
    memory: 1800Mi
    cpu: 1000m
startupProbe:
  enabled: true




install kafka

helm install -n corda kafka bitnami/kafka --version 19.1.0 --values <location to kafka.yaml file> --wait





Install Postgresql to minikube

Create Postgresql.yaml file

# Configuring PostgreSQL image
image:
  tag: 14.4.0-debian-11-r23
# values configuring the permissions of volumes.
volumePermissions:
  # -- enable/disable an init container which changes ownership of the mounted volumes.
  enabled: true
# Configuring TLS
tls:
  # -- enable TLS for postgres.
  enabled: true
  # -- enable automatic certificate generation for postgres.
  autoGenerated: true
# auth configuration.
auth:
  # -- name of database to be created.
  database: cordacluster
  # -- name of the user to be created.
  username: user
  # -- name of the password of the user to be created - defaults to generated value
  password:
# primary (read/write instance) configuration.
primary:
  # Custom init configuration..
  initdb:
    # -- ConfigMap-like object containing scripts to be executed on startup.
    # @default -- corda_user_init.sh
    scripts:
      corda_user_init.sh: |
        export PGPASSFILE=/tmp/pgpasswd$$
        touch $PGPASSFILE
        chmod 600 $PGPASSFILE
        trap "rm $PGPASSFILE" EXIT
        echo "localhost:${POSTGRES_PORT_NUMBER:-5432}:$POSTGRESQL_DATABASE:postgres:$POSTGRES_POSTGRES_PASSWORD" > $PGPASSFILE
        psql -v ON_ERROR_STOP=1 cordacluster <<-EOF
          ALTER ROLE "$POSTGRES_USER" NOSUPERUSER CREATEDB CREATEROLE INHERIT LOGIN;
        EOF



install postgresql

helm install -n corda postgres bitnami/postgresql --values <location to postgres.yaml file> --wait





Install Corda5 into minikube



image:
  registry: "corda5beta3hc01.azurecr.io" <Your ACR server name>
  tag: "5.0.0.0-Beta3-HC01" <Docker image tags>
imagePullSecrets:
  - cred04
bootstrap:
  kafka:
    replicas: 1
    partitions: 10
kafka:
  bootstrapServers: "kafka:9092"
  sasl:
    enabled: true
    username:
      value: "user"
    password:
      valueFrom:
        secretKeyRef:
          name: "kafka-jaas"
          key: "client-passwords"
  tls:
    enabled: true
    truststore:
      valueFrom:
        secretKeyRef:
          name: "kafka-0-tls"
          key: "ca.crt"
db:
  cluster:
    host: "postgres-postgresql"
    username:
      value: "user"
    password:
      valueFrom:
        secretKeyRef:
          name: "postgres-postgresql"
          key: "password"
-----------------------------------------------------------------------------------------------






helm upgrade --install corda <location to the tgz file> --values <location to the corda.yaml file> --wait

the the tgz file that was downloaded from customer hub



example

helm upgrade --install corda Corda5/corda-enterprise-5.0.0.tgz --values Corda5/corda.yaml --wait



check to ensure that the pods are on line

corda-corda-enterprise-create-topics-7gdbw                        0/1     Completed   0          6m42s
corda-corda-enterprise-crypto-worker-747c4f858d-fbdjh             1/1     Running     0          5m41s
corda-corda-enterprise-db-worker-6ff78f5695-54x54                 1/1     Running     0          5m41s
corda-corda-enterprise-flow-worker-76fb7b89fc-m6b9m               1/1     Running     0          5m41s
corda-corda-enterprise-membership-worker-659f4bfdff-zzjp7         1/1     Running     0          5m41s
corda-corda-enterprise-p2p-gateway-worker-669877b65f-s4rx8        1/1     Running     0          5m41s
corda-corda-enterprise-p2p-link-manager-worker-74d77db699-5m97s   1/1     Running     0          5m41s
corda-corda-enterprise-rest-worker-6f9d79b7b7-sr4pw               1/1     Running     0          5m41s
corda-corda-enterprise-setup-db-wjgt9                             0/1     Completed   0          6m11s
corda-corda-enterprise-setup-rbac-cld69                           0/3     Completed   0          4m59s
kafka-0                                                           1/1     Running     0          44m
kafka-zookeeper-0                                                 1/1     Running     0          44m
postgres-postgresql-0                                             1/1     Running     0          16m



if you receive the following error



helm upgrade --install corda Corda5/corda-enterprise-5.0.0.tgz --values Corda5/corda.yaml --wait
Release "corda" does not exist. Installing it now.
Error: failed pre-install: 1 error occurred:
        * timed out waiting for the condition



check that the kubectl secret are present

NAME                                    TYPE                             DATA   AGE
corda-corda-enterprise-cluster-db       Opaque                           1      173m
corda-corda-enterprise-config           Opaque                           2      173m
corda-corda-enterprise-rest-api-admin   Opaque                           2      173m
cred04                                  kubernetes.io/dockerconfigjson   1      3h31m



cred04 secret should be present


If helm fail to install the Corda5 helm charts



helm list
helm uninstall <remove all corda chart from list>

Note that helm uninstall doesn't remove K8 secrets create by corda

to remove secrets
kubectl delete secrets <corda secrets>


Setup MGM



open port 7878 and 8181 on the Azure VM



open the port 7878 on the Azure VM by going to the Networking section to establish connection for the RPC worker
then add inbound port rules:


kubectl port-forward --address 0.0.0.0 --namespace corda deployment/corda-corda-enterprise-rest-worker 7878:8888


open the port 8181 on the Azure VM by going to the Networking section to establish connection for the P2P worker



kubectl port-forward --address 0.0.0.0 --namespace corda deployment/corda-corda-enterprise-p2p-gateway-worker 8181:8080



Download and install java11



Download MGM script script from:

https://jor3cev-my.sharepoint.com/personal/peter_li_r3_com/_layouts/15/onedrive.aspx?ga=1&id=%2Fpersonal%2Fpeter%5Fli%5Fr3%5Fcom%2FDocuments%2FWebinar%20Series%2FPS%2DTraining%2FCF%2DDeployment%2FLabs 





get your cluster password:

kubectl get secret corda-corda-enterprise-rest-api-admin -o go-template='{{ .data.password | base64decode }}'



Download the Multi_ClusterLab as a zip file

shown below:



extract the file 





GenerateTLS script

then navigate to the clusterTLSsetup directory modify the script GenerateTLS.sh script

modify the script

set your environment in the GenerateTLS.sh script:

RPC_HOST=localhost
RPC_PORT=8888

and 

P2P_GATEWAY_HOST=localhost
P2P_GATEWAY_PORT=8080



as shown below:

echo "---Set Env---"
RPC_HOST=localhost
RPC_PORT=8888
P2P_GATEWAY_HOST=localhost
P2P_GATEWAY_PORT=8080
API_URL="https://$RPC_HOST:$RPC_PORT/api/v1"



run:

bash ./GenerateTLS.sh <the cluster password> <name of the certificate: example Corda5TlsCertNow>







MGM script

then navigate to the mgmdirectory which is under the Multi-ClusterLab directory

set your environment in the DeployMGM.sh script:

RPC_HOST=localhost
RPC_PORT=8888

and 

P2P_GATEWAY_HOST=localhost
P2P_GATEWAY_PORT=8080



as shown below:

echo "---Set Env---"
RPC_HOST=localhost
RPC_PORT=8888
P2P_GATEWAY_HOST=localhost
P2P_GATEWAY_PORT=8080
API_URL="https://$RPC_HOST:$RPC_PORT/api/v1"





the mgm directory should contain files:

DeployMGM.sh  digicert-ca.pem  gradle-plugin-default-key.pem  MgmGroupPolicy.json

run:

 bash ./DeployMGM.sh <the cluster password>





Notary Script

then navigate to the notary directory which is under the Multi-ClusterLab directory

set your environment in the DeployNotary.sh script:

RPC_HOST=localhost
RPC_PORT=8888

and 

P2P_GATEWAY_HOST=localhost
P2P_GATEWAY_PORT=8080



as shown below:

echo "---Set Env---"
RPC_HOST=localhost
RPC_PORT=8888
P2P_GATEWAY_HOST=localhost
P2P_GATEWAY_PORT=8080
API_URL="https://$RPC_HOST:$RPC_PORT/api/v1"



the notary directory should contain files:

DeployNotary.sh  notary-plugin-non-validating-server-5.0.0.0-package.cpb



run:

bash ./DeployNotary.sh <the cluster password>





Vnode script

then navigate to the localMember directory which is under the Multi-ClusterLab directory

set your environment in the DeployNotary.sh script:

RPC_HOST=localhost
RPC_PORT=8888

and 

P2P_GATEWAY_HOST=localhost
P2P_GATEWAY_PORT=8080



as shown below:

echo "---Set Env---"
RPC_HOST=localhost
RPC_PORT=8888
P2P_GATEWAY_HOST=localhost
P2P_GATEWAY_PORT=8080
API_URL="https://$RPC_HOST:$RPC_PORT/api/v1"



run:

bash ./DeployVNode.sh <the cluster password>
