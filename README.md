# voting-app

page_type	languages	products	description
sample
python
azure
azure-redis-cache
This sample creates a multi-container application in an Azure Kubernetes Service (AKS) cluster.

sudo usermod -aG docker jenkins;
sudo usermod -aG docker azureuser;
sudo touch /var/lib/jenkins/jenkins.install.InstallUtil.lastExecVersion;
sudo service jenkins restart;
sudo cp ~/.kube/config /var/lib/jenkins/.kube/
sudo chmod 777 /var/lib/jenkins/
sudo chmod 777 /var/lib/jenkins/config

#Jenkins instance needs Docker installed and configured and kubectl.

Listed azure plugin is needed to deploy it
Azure Container Registry Task "Jenkins plug-in to send a docker-build request to Azure Container Registry."
Azure Container Service  "Jenkins plug-in to deploy configurations to Azure Container Service (AKS)."
Azure Credential "Jenkins plug-in to manage Azure credentials."

# Build new image and push to ACR.
WEB_IMAGE_NAME="${ACR_LOGINSERVER}/azure-vote-front:kube${BUILD_NUMBER}"
docker build -t $WEB_IMAGE_NAME ./azure-vote
docker login ${ACR_LOGINSERVER} -u ${ACR_ID} -p ${ACR_PASSWORD}
docker push $WEB_IMAGE_NAME

# Update kubernetes deployment with new image.
WEB_IMAGE_NAME="${ACR_LOGINSERVER}/azure-vote-front:kube${BUILD_NUMBER}"
kubectl set image deployment/azure-vote-front azure-vote-front=$WEB_IMAGE_NAME

#Execute shell in build
kubectl apply -f azure-vote-all-in-one-redis.yaml
