# Deploy Jenkins and Gitlab from Helm Chart in IBM Cloud Private (ICP)
## Pre-requisites
1. Install [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
2. Download and Install [IBM CLoud CLI](https://console.bluemix.net/docs/cli/reference/bluemix_cli/all_versions.html)
3. Download ICP Plugin for IBM Cloud CLI from your ICP Console: `https://your-icp-master-node:8443/console/tools/cli`
4. Install the downloaded [ICP Plugin for IBM Cloud CLI](https://www.ibm.com/support/knowledgecenter/SSBS6K_2.1.0/manage_cluster/install_cli.html):
`bx plugin install <replace-with-path-to-your-downloaded-plugin-file>`
5. Install [Helm Client](https://github.com/kubernetes/helm):
   - [OSX](https://kubernetes-helm.storage.googleapis.com/helm-v2.7.2-darwin-amd64.tar.gz)
   - [Linux](https://kubernetes-helm.storage.googleapis.com/helm-v2.7.2-linux-amd64.tar.gz)
   - [Windows](https://kubernetes-helm.storage.googleapis.com/helm-v2.7.2-windows-amd64.tar.gz)
6. Configure Helm Client:
```
bx pr login -a https://your-icp-master-node:8443 -u your-icp-user -p your-icp-password -skip-ssl-validation
bx pr clusters #Check Cluster Name
bx pr cluster-config your-cluster-name #Configure Kubectl
helm init --client-only
helm repo update
```
### Pre-requisites for deployment from ICP UI
7. Verify Helm Repository in ICP:
   - Navigate to Manage > Helm Repositories
   - Verify if kubernetes helm repository `https://kubernetes-charts.storage.googleapis.com` is available or create a new one
8. If you need to create a new repository:
   - Click *Add repository*
   - Fill the repository detail:
     name: `kube`
     URL: `https://kubernetes-charts.storage.googleapis.com`
   - Click *Sync repositories*
## Deploy Jenkins
Use either UI or CLI method below to deploy Jenkins chart
### Install Jenkins Chart with Helm CLI
*You can skip these steps if you prefer UI method*
1. Login to ICP and Configure Kubernetes Cluster on client:
```
bx pr login -a https://your-icp-master-node:8443 -u your-icp-user -p your-icp-password -skip-ssl-validation
bx pr clusters #Check Cluster Name
bx pr cluster-config your-cluster-name #Configure Kubectl
```
2. Verify kubernetes helm repository `helm repo list`, there should be at least one default repository:
   - name: `stable`
   - URL: `https://kubernetes-charts.storage.googleapis.com`

3. Install Jenkins from Helm CLI:
```
helm install --name your-jenkins-release-name --namespace your-namespace --set Master.Image=jenkins/jenkins --set Master.ImageTag=lts --set Master.InstallPlugins={kubernetes:1.3.3\,workflow-aggregator:2.5\,workflow-job:2.17\,credentials-binding:1.15\,git:3.8.0\,gitlab-plugin:1.5.3} --set Agent.Image=jenkins/jnlp-slave --set Agent.ImageTag=latest --set rbac.install=true stable/jenkins
```
4. Jenkins URL:
   - Query Jenkins NodePort: `kubectl get svc your-jenkins-release-name -o=jsonpath='{.spec.ports[?(@.name=="http")].nodePort}{"\n"}'`
   - Jenkins URL: `http://your-proxy-ip:jenkins-node-port`

### Install Jenkins Chart from ICP UI
*You can skip this step if you have installed Jenkins using CLI method*
1. Open Catalog > Helm Charts
2. Type in filter: `jenkins`
3. Open Jenkins chart and click `Configure`
4. Update the Variables:
   - Release name: *type-your-release-name*
   - Target namespace: `default`
   - Master.Image: `jenkins/jenkins`
   - Master.ImageTag: `lts`
   - InstallPlugins.0: `kubernetes:1.3.3`
   - InstallPlugins.1: `workflow-aggregator:2.5` 
   - InstallPlugins.2: `workflow-job 2.1.7` 
   - InstallPlugins.3: `credentials-binding:1.15`
   - InstallPlugins.4: `git:3.8.0`
   - Agent.Image: `jenkins/jnlp-slave`
   - Agent.ImageTag: *leave empty*
   - rbac.install: `true`
5. Wait for Deployment to complete
6. After deployment is completed, jenkins url can be found at:
   - Open Network Access > Services
   - Type in filter: *your-release-name* and open the jenkins service
   - Open link in **Node port** row
7. Get Jenkins initial password from kubectl command line:
   - username: `kubectl get secrets <your-jenkins-secret> -o jsonpath='{.data.jenkins-admin-user}' | base64 -D; echo`
   - password: `kubectl get secrets <your-jenkins-secret> -o jsonpath='{.data.jenkins-admin-password}' | base64 -D; echo`
8. Login to Jenkins with provided credentials
9. Open Jenkins > Manage Jenkins > Manage Plugins:
   - Select *Available* tab
   - Filter for GitLab
   - Select gitlab plugin
   - Install and restart Jenkins
## Deploy Gitlab Chart (with Helm CLI)
1. Login to ICP and Configure Kubernetes Cluster on client:
```
bx pr login -a https://your-icp-master-node:8443 -u your-icp-user -p your-icp-password -skip-ssl-validation
bx pr clusters #Check Cluster Name
bx pr cluster-config your-cluster-name #Configure Kubectl
```
2. Verify kubernetes helm repository `helm repo list`, there should be at least one default repository:
   - name: `stable`
   - URL: `https://kubernetes-charts.storage.googleapis.com`
3. Use helm to install gitlab:
```
helm install --name <type-your-gitlab-release-name> --set externalUrl=http://<type-your-gitlab-release-name>-gitlab-ce,gitlabRootPassword=<type-your-gitlab-root-password> stable/gitlab-ce
```
## Configure Jenkins-Gitlab Integration
### Jenkins Configuration
1. Get Jenkins initial password from kubectl command line:
   - username: `kubectl get secrets <your-jenkins-secret> -o jsonpath='{.data.jenkins-admin-user}' | base64 -D; echo`
   - password: `kubectl get secrets <your-jenkins-secret> -o jsonpath='{.data.jenkins-admin-password}' | base64 -D; echo`
2. Login to Jenkins with provided credentials   
3. Open Jenkins > Manage Jenkins > Configure System:
   - Under **Gitlab** section, untick `Enable authentication for '/project' end-point`
   - Under **Cloud** section, **Kubernetes** sub section, increase `Container Cap` value `1000`
   - Save configuration
4. Open Jenkins > Manage Jenkins > Configure Global Security:
   - Under **Agents** section, verify the `Agent protocols` and untick the deprecated protocols
5. To allow Jenkins build to be triggered from Gitlab:
   - Open the intended Jenkins job
   - Select *Build Triggers* tab
   - Tick *Build when a change is pushed to GitLab....*
   - Save
### Gitlab Configuration
1. Open Gitlab URL
2. Click *Register* to self register if you don't have credential
3. Login to Gitlab with your credential
4. Open a Project to be integrated with Jenkins:
   - Open *Settings* tab
   - Open *Integrations* tab
   - Type in **URL** field: Jenkins project URL, sample of the URL:
     `http://your-jenkins-release-name-jenkins:8080/project/your-jenkins-job-name`
   - Untick *Enable SSL verification*
   - Click *Add webhook*
