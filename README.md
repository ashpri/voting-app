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
#create acr registry secret
kubectl create secret docker-registry regrec \
    --namespace bluegreen \
    --docker-server=aksashuacr.azurecr.io \
    --docker-username=aksashuacr \
    --docker-password=****************
-----------------------------------
Create a Jenkins environment variable
A Jenkins environment variable is used to hold the ACR login server name. This variable is referenced during the Jenkins build job. To create this environment variable, complete the following steps:

On the left-hand side of the Jenkins portal, select Manage Jenkins > Configure System

Under Global Properties, select Environment variables. Add a variable with the name ACR_LOGINSERVER and the value of your ACR login server.

Jenkins environment variables

When complete, Select Save at the bottom of the page.

Create a Jenkins credential for ACR
During the CI/CD process, Jenkins builds new container images based on application updates, and needs to then push those images to the ACR registry.

To allow Jenkins to push updated container images to ACR, you need to specify credentials for ACR.

For separation of roles and permissions, configure a service principal for Jenkins with Contributor permissions to your ACR registry.

Create a service principal for Jenkins to use ACR
First, create a service principal using the az ad sp create-for-rbac command:

Azure CLI

Copy
az ad sp create-for-rbac
Bash

Copy
{
  "appId": "626dd8ea-042d-4043-a8df-4ef56273670f",
  "displayName": "azure-cli-2018-09-28-22-19-34",
  "name": "http://azure-cli-2018-09-28-22-19-34",
  "password": "1ceb4df3-c567-4fb6-955e-f95ac9460297",
  "tenant": "72f988bf-86f1-41af-91ab-2d7cd011db48"
}
Make a note of the appId and password. These values are used in following steps to configure the credential resource in Jenkins.

Get the resource ID of your ACR registry using the az acr show command, and store it as a variable.

Azure CLI

Copy
ACR_ID=$(az acr show --resource-group <Resource_Group> --name <acrLoginServer> --query "id" --output tsv)
Replace <Resource_Group> and <acrLoginServer> with the appropriate values.

Create a role assignment to assign the service principal Contributor rights to the ACR registry.

Azure CLI

Copy
az role assignment create --assignee <appID> --role Contributor --scope $ACR_ID
Replace <appId> with the value provided in the output of the pervious command use to create the service principal.

Create a credential resource in Jenkins for the ACR service principal
With the role assignment created in Azure, now store your ACR credentials in a Jenkins credential object. These credentials are referenced during the Jenkins build job.

Back on the left-hand side of the Jenkins portal, select Manage Jenkins > Manage Credentials > Jenkins Store > Global credentials (unrestricted) > Add Credentials

Ensure that the credential kind is Username with password and enter the following items:

Username - The appId of the service principal created for authentication with your ACR registry.
Password - The password of the service principal created for authentication with your ACR registry.
ID - Credential identifier such as acr-credentials
When complete, the credentials form looks like the following example:

Create a Jenkins credential object with the service principal information

Select OK and return to the Jenkins portal.

Create a Jenkins project
From the home page of your Jenkins portal, select New item on the left-hand side:

Enter azure-vote as job name. Choose Freestyle project, then select OK

Under the General section, select GitHub project and enter your forked repo URL, such as https://github.com/<your-github-account>/azure-voting-app-redis

Under the Source code management section, select Git, enter your forked repo .git URL, such as https://github.com/<your-github-account>/azure-voting-app-redis.git

Under the Build Triggers section, select GitHub hook trigger for GITscm polling

Under Build Environment, select Use secret texts or files

Under Bindings, select Add > Username and password (separated)

Enter ACR_ID for the Username Variable, and ACR_PASSWORD for the Password Variable

Jenkins bindings

Choose to add a Build Step of type Execute shell and use the following text. This script builds a new container image and pushes it to your ACR registry.

Bash

Copy
# Build new image and push to ACR.
WEB_IMAGE_NAME="${ACR_LOGINSERVER}/azure-vote-front:kube${BUILD_NUMBER}"
docker build -t $WEB_IMAGE_NAME ./azure-vote
docker login ${ACR_LOGINSERVER} -u ${ACR_ID} -p ${ACR_PASSWORD}
docker push $WEB_IMAGE_NAME
Add another Build Step of type Execute shell and use the following text. This script updates the application deployment in AKS with the new container image from ACR.

Bash

Copy
# Update kubernetes deployment with new image.
WEB_IMAGE_NAME="${ACR_LOGINSERVER}/azure-vote-front:kube${BUILD_NUMBER}"
kubectl set image deployment/azure-vote-front azure-vote-front=$WEB_IMAGE_NAME
Once completed, click Save.

Test the Jenkins build
Before you automate the job based on GitHub commits, manually test the Jenkins build.

This build validates that the job has been correctly configured. It confirms the proper Kubernetes authentication file is in place, and that authentication to ACR working.

On the left-hand menu of the project, select Build Now.

Jenkins test build

The first build longer as the Docker image layers are pulled down to the Jenkins server.

The builds do the following tasks:

Clones the GitHub repository
Builds a new container image
Pushes the container image to the ACR registry
Updates the image used by the AKS deployment
Because no changes have been made to the application code, the web UI is unchanged.

Once the build job is complete, select build #1 under build history. Select Console Output and view the output from the build process. The final line should indicate a successful build.

Create a GitHub webhook
With a successful manual build complete, now integrate GitHub into the Jenkins build. Use a webhook to run the Jenkins build job each time code is committed to GitHub.

To create the GitHub webhook, complete the following steps:

Browse to your forked GitHub repository in a web browser.

Select Settings, then select Webhooks on the left-hand side.

Choose to Add webhook. For the Payload URL, enter http://<publicIp:8080>/github-webhook/, where <publicIp> is the IP address of the Jenkins server. Make sure to include the trailing /. Leave the other defaults for content type and to trigger on push events.

Select Add webhook.

Create a GitHub webhook for Jenkins

Test the complete CI/CD pipeline
Now you can test the whole CI/CD pipeline. When you push a code commit to GitHub, the following steps happen:

The GitHub webhook notifies Jenkins.
Jenkins starts the build job and pulls the latest code commit from GitHub.
A Docker build is started using the updated code, and the new container image is tagged with the latest build number.
This new container image is pushed to Azure Container Registry.
Your application running on Azure Kubernetes Service updates with the latest image from Azure Container Registry.
    ---------------------
    
