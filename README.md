<!-- SPDX-License-Identifier: Apache-2.0 -->

# On-Premise Detailed Installation Guide

This guide provides a comprehensive walkthrough for deploying Tazama on an on-premises Kubernetes cluster. It includes provisioning the infrastructure, setting up MetalLB for networking, configuring persistent storage, and deploying all required components using Helm and Jenkins.

## Table of Contents

- [On-Prem Deployment Overview](#on-prem-deployment-overview)
- [Step 0 - Infrastructure Setup](#step-0---infrastructure-setup)
  - [Linux OS Preparation](#linux-os-preparation)
  - [Install Docker](#install-docker)
  - [Install Kubernetes (kubeadm)](#install-kubernetes-kubeadm)
  - [Cluster Initialization](#cluster-initialization)
  - [Install a Pod Network (Calico)](#install-a-pod-network-calico)
  - [Install Helm](#install-helm)
  - [Install MetalLB](#install-metallb)
  - [Persistent Storage Setup](#persistent-storage-setup)
- [Step 1 - Helm Charts](#step-1---helm-charts)
  - [Overview](#overview)
  - [Prerequisites](#prerequisites)
  - [Adding Required Namespaces](#adding-required-namespaces)
  - [Helm Repository Setup](#helm-repository-setup)
    - [Repo](#repo)
    - [Helm Repository Setup](#helm-repository-setup-1)
    - [Ingress Setup](#ingress-setup)
    - [Installing Helm Charts](#installing-helm-charts)
    - [Uninstalling Charts](#uninstalling-charts)
- [Step 2 - Configuration](#step-2---configuration)
  - [NATS](#nats)
  - [ElasticSearch](#elasticsearch)
  - [ArangoDB (Single Deployment)](#arangodb-single-deployment)
  - [Jenkins](#jenkins)
  - [Nginx Ingress Controller](#nginx-ingress-controller)
  - [APM, Logstash, Kibana (Elasticsearch)](#apm-logstash-kibana-elasticsearch)
  - [Grafana](#grafana)
  - [Prometheus](#prometheus)
  - [Vault](#vault)
  - [KeyCloak](#keycloak)
- [Step 3 - Post-Installation Configuration](#step-3---post-installation-configuration)
  - [Elasticsearch Kube Secret for LumberJack](#elasticsearch-kube-secret-for-lumberjack)
  - [Setting up TLS for Ingress](#setting-up-tls-for-ingress)
  - [Configuring Ingress Domain Names](#configuring-ingress-domain-names)
  - [Vault Configuration](#vault-configuration)
  - [Logstash Configuration](#logstash-configuration)
  - [APM Configuration](#apm-configuration)
  - [Jenkins Configuration](#jenkins-configuration)
    - [Accessing Jenkins UI](#accessing-jenkins-ui)
    - [Adding Credentials in Jenkins](#adding-credentials-in-jenkins)
      - [GitHub Credentials](#github-credentials)
      - [GitHub Read Package Credentials](#github-read-package-credentials)
      - [Kubernetes Credentials](#kubernetes-credentials)
    - [Adding Managed Files for NPM Configuration in Jenkins](#adding-managed-files-for-npm-configuration-in-jenkins)
    - [Adding Managed Files for Container Registry in Jenkins](#adding-managed-files-for-container-registry-in-jenkins)
    - [Jenkins Node.js Configuration](#jenkins-nodejs-configuration)
    - [Jenkins Docker Installation Configuration](#jenkins-docker-installation-configuration)
    - [Steps to Configure Jenkins Global Variables](#steps-to-configure-jenkins-global-variables)
    - [Adding Jenkins Jobs](#adding-jenkins-jobs)
      - [Download the Job Configurations](#download-the-job-configurations)
      - [Navigate to Configuration Directory](#navigate-to-configuration-directory)
      - [Copy Jobs to Jenkins Pod](#copy-jobs-to-jenkins-pod)
      - [Finalize the Setup](#finalize-the-setup)
      - [Reload Jenkins Configuration](#reload-jenkins-configuration)
    - [Building Jenkins Agent Locally](#building-jenkins-agent-locally)
    - [Setting up a Jenkins Cloud Agent](#setting-up-a-jenkins-cloud-agent)
- [Step 4 - Running Jenkins Jobs to Install Processors](#step-4---running-jenkins-jobs-to-install-processors)
  - [Overview](#overview-1)
  - [Populating ArangoDB](#populating-arangodb)
  - [Edit Jobs: Configuring Credentials and Kubernetes Endpoints in Jenkins](#edit-jobs-configuring-credentials-and-kubernetes-endpoints-in-jenkins)
    - [Configuring Rule Processors](#configuring-rule-processors)
  - [Deploying to the Cluster](#deploying-to-the-cluster)
  - [End-to-End Platform Testing with the "E2E Test" Jenkins Job](#end-to-end-platform-testing-with-the-e2e-test-jenkins-job)
    - [Overview of the "E2E Test" Job](#overview-of-the-e2e-test-job)
    - [Purpose and Benefits](#purpose-and-benefits)
    - [Running the Test and Post-Test Evaluation](#running-the-test-and-post-test-evaluation)
- [Common Errors](#common-errors)
  - [Arango Ingress Error](#arango-ingress-error)
  - [Network Access Error in Container Deployment](#network-access-error-in-container-deployment)
  - [Addressing Pod Restart Issues in Kubernetes](#addressing-pod-restart-issues-in-kubernetes)
  - [Addressing Jenkins Build Authentication Errors](#addressing-jenkins-build-authentication-errors)
  - [Jenkins Build Agent Terminating and Restarting](#jenkins-build-agent-terminating-and-restarting)
  - [Forbidden User on Jenkins Job Builds](#forbidden-user-on-jenkins-job-builds)
- [Conclusion: Finalizing Tazama System Installation](#conclusion-finalizing-tazama-system-installation)

---

**Important:** Access to the Tazama GIT Repository is required to proceed. If you do not currently have this access, or if you are unsure about your access level, please reach out to the Tazama Team to request the necessary permissions. It's crucial to ensure that you have the appropriate credentials to access the repository for seamless integration and workflow management.

## Step 0 - Infrastructure Setup

### Linux OS Preparation

Use Ubuntu Server or CentOS/RHEL. Ensure your servers meet the following:

- 4+ CPU Cores
- 8+ GB RAM
- Static IPs assigned
- Hostnames set
- Root or sudo access

### Install Docker

```bash
sudo apt update && sudo apt install -y docker.io
sudo systemctl enable docker && sudo systemctl start docker
```

### Install Kubernetes (kubeadm)

Follow these steps on **all nodes**:

```bash
sudo apt update && sudo apt install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update && sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### Cluster Initialization

On the **master node only**:

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Run the output `kubeadm join` command on all **worker nodes**.

### Install a Pod Network (Calico)

Kubernetes does not install a network plugin by default. After `kubeadm init`, you must manually install a Container Network Interface (CNI) such as **Calico**. Calico provides networking and network policy features and includes components like the `calico-node` DaemonSet and the `calico-kube-controllers` Deployment.

To install Calico (including the kube-controllers), run the following:

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml
```

This will deploy:
- `calico-node`: A DaemonSet that handles routing and enforcement on each node.
- `calico-kube-controllers`: A Deployment that manages Calico-specific resources such as IP address management and network policies.

To confirm the installation:

```bash
kubectl get pods -n kube-system -l k8s-app=calico-kube-controllers
```

Ensure all Calico pods show a `Running` status before proceeding.

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml
```

### Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### Install MetalLB

MetalLB is used to expose services using LoadBalancer type.

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.10/config/manifests/metallb-native.yaml
```

Wait for all MetalLB pods to be in the `Running` state:

```bash
kubectl get pods -n metallb-system
```

#### Configure MetalLB IP Pool:

Create a config file `metallb-config.yaml`:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-address-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.240-192.168.1.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2advertisement
  namespace: metallb-system
```

Apply the configuration:

```bash
kubectl apply -f metallb-config.yaml
```

> Note: Replace the IP range with a valid range from your local network.

### Persistent Storage Setup

You can use NFS or local persistent volumes. Here’s an example for local path provisioner:

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

---

## Helm Repository Setup

First, add the Tazama Helm repository to enable the installation of charts:

**The list below are the different helm charts:**

1. NATS
2. ElasticSearch
3. ArangoDB (single deployment)
4. Jenkins
5. Nginx ingress
6. APM (Elasticsearch)
7. Logstash (Elasticsearch)
8. Kibana (Elasticsearch)
9. Infra-chart
10. Grafana - **Optional**
11. Prometheus - **Optional**
12. Vault - **Optional**
13. KeyCloak - **Optional**

**Optional** - Please note that these are additional features; while not required, they can enhance the platform's capabilities. Implementing them is optional and will not hinder the basic operation or the end-to-end functionality of the platform.

**ie:** Another HELM chart exists for the clustered version of ArangoDB. However, the single deployment version is preferred over the clustered one because it includes functionality that is absent or required in the enterprise option.

### Repo

[https://github.com/frmscoe/GKE-helm](https://tazama-lf.github.io/GKE-helm)

### Helm Repository Setup

First, add the Tazama Helm repository to enable the installation of charts:

```bash
helm repo add Tazama https://tazama-lf.github.io/GKE-helm/
helm repo update
```

To confirm the FRMS repo has been successfully added:

```bash
helm search repo Tazama
```

### Ingress Setup

To expose services outside your cluster, enable ingress on necessary charts:

1. Kibana
2. ArangoDb
3. Jenkins
4. TMS

```bash
helm install kibana Tazama/kibana --namespace=development --set ingress.enabled=true
...
```

If you prefer not to configure an ingress controller, you can simply use port forwarding to access the front-end interfaces of your applications. This approach will not impact the end-to-end functionality of your system, as it is designed to utilize **fully qualified domain names** (FQDNs) for internal cluster communication.

Use `kubectl get svc -n ingress-nginx` to get the external IP assigned by MetalLB for accessing services.

Ensure DNS entries or `/etc/hosts` are updated to map service hostnames to the external IP.

### Installing Helm Charts

The Tazama system is composed of multiple Helm charts for various services and components. These need to be installed in a specific order due to dependencies.

1. **Infra-chart** - Sets up necessary namespaces and storage classes.

```bash
helm install infra-chart Tazama/infra-chart
```

2. Follow with the installation of other charts as listed, specifying the namespace as required:

```bash
helm install nginx-ingress-controller Tazama/ingress-nginx --namespace=ingress-nginx
helm install elasticsearch Tazama/elasticsearch --namespace=development
helm install kibana Tazama/kibana --namespace=development
helm install apm Tazama/apm-server --namespace=development
helm install logstash Tazama/logstash --namespace=development
helm install arango Tazama/arangodb --namespace=development
helm install jenkins Tazama/jenkins --namespace=cicd
helm install nats Tazama/nats --namespace=development
```
3. Install Valkey. Valkey is an open source (BSD) high-performance key/value datastore that supports a variety workloads such as caching, message queues, and can act as a primary database.

```bash
helm install valkey-cluster bitnami/valkey-cluster --version 2.1.1 --namespace=development
```

**Additional**
For optional components like Grafana, Prometheus, Vault, and KeyCloak, use similar commands if you decide to implement these features.

**Extra Information:** [https://helm.sh/docs/helm/helm\_install/](https://helm.sh/docs/helm/helm_install/)

### Uninstalling Charts

If you need to remove the Tazama deployment:

```
helm uninstall Tazama
```

# Step 2 : Configuration

For a system utilizing a variety of Helm charts, optimizing performance, storage, and configuration can significantly impact its efficiency and scalability. Below are details on how to configure and optimize each component for your FRMS system:

## 1. NATS

- **Configuration**: Adjust NATS cluster size, memory, and CPU limits according to your workload.
- **Storage**: For persistent storage, configure the `nats-streaming` stateful set with appropriate volume sizes.
- **Performance**: Tune the `max_connections`, `max_payload`, and `write_deadline` settings for better performance.
- **Documentation**: [NATS Documentation](https://docs.nats.io/)

## 2. ElasticSearch

- **Configuration**: Increase the number of nodes for redundancy and scalability. Adjust JVM settings for better performance.
- **Storage**: Utilize persistent volumes with dynamic provisioning for data storage. Consider SSDs for faster I/O.
- **Performance**: Optimize index settings, such as `number_of_shards` and `number_of_replicas`, based on your use case.
- **Documentation**: [Elasticsearch Documentation](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-elasticsearch-specification.html)

## 3. ArangoDB (Single Deployment)

- **Configuration**: Choose the right deployment mode (Single, Active Failover, or Cluster) based on your needs.
- **Storage**: Ensure that persistent volume claims (PVCs) are adequately sized for your database growth.
- **Performance**: Configure resource limits and requests to ensure that ArangoDB has enough CPU and memory.
- **Documentation**: [ArangoDB Documentation](https://docs.arangodb.com/3.11/deploy/single-instance/)

## 4. Jenkins

- **Configuration**: Utilize configuration as code (JCasC) for easier management and scalability of Jenkins settings.
- **Storage**: Configure persistent storage for Jenkins home to ensure data is retained across restarts.
- **Performance**: Adjust executors and resource limits based on your CI/CD pipeline requirements.
- **Documentation**: [Jenkins Documentation](https://www.jenkins.io/doc/book/)

## 5. Nginx Ingress Controller

- **Configuration**: Adjust rate limiting, body size limits, and SSL/TLS configurations to meet your security and usability requirements.
- **Performance**: Enable HTTP/2, configure SSL ciphers, and tuning worker processes for better performance.
- **Documentation**: [NGINX Ingress Controller Documentation](https://docs.nginx.com/nginx-ingress-controller/)

## 6. APM, Logstash, Kibana (Elasticsearch)

- **Configuration**: Link these components together for a cohesive monitoring and logging solution. Configure Kibana dashboards for easy visualization.
- **Storage**: Ensure Elasticsearch has enough storage for logs and APM data. Consider using faster storage classes for better performance.
- **Performance**: For Logstash, adjust pipeline workers and batch sizes. For APM, tune the sampling rate based on your needs.
- **Documentation**: [Kibana Documentation](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-kibana.html) [Logstash Documentation](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-logstash.html) [APM Documentation](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-apm-server.html)

## 7. Grafana

- **Configuration**: Use Grafana dashboards to visualize metrics from Prometheus and other data sources.
- **Security**: Implement OAuth or LDAP for authentication. Use HTTPS for secure connections.
- **Documentation**: [Grafana Documentation](https://grafana.com/docs/grafana/latest/) [Environment Monitoring](https://frmscoe.atlassian.net/wiki/spaces/FRMS/pages/65241090/Environment+Monitoring)

## 8. Prometheus

- **Configuration**: Adjust scrape intervals and retention policies to balance between performance and data granularity.
- **Storage**: Use persistent volumes for long-term metric storage. Consider high-performance storage for large datasets.
- **Documentation**: [Prometheus Documentation](https://prometheus.io/docs/introduction/overview/)

## 9. Vault

- **Configuration**: Set up Vault in HA mode with a proper storage backend like Consul for resilience.
- **Security**: Use auto-unseal feature to automate the unsealing process securely.
- **Documentation**: [Vault Documentation](https://developer.hashicorp.com/vault/docs)

## 10. KeyCloak

- **Configuration**: Set up realms, clients, and roles according to your authentication and authorization needs.
- **Performance**: Adjust caching settings and session limits to optimize for your workload.
- **Documentation**: [Keycloak Documentation](https://www.keycloak.org/documentation)

Each of these components plays a critical role in the Tazama system. By carefully configuring and optimizing them according to the guidelines provided, you can ensure that your system is secure, scalable, and performs optimally. Always refer to the official documentation for the most up-to-date information and advanced configuration options.

# Step 3: Post-Installation Configuration

## Elasticsearch Kube Secret for LumberJack

In order to get the processor pods to write logs to the lumberjack deployment which then writes the log information to elasticsearch. 

[Logging Data View](https://github.com/frmscoe/docs/blob/main/Technical/Logging/Logging-Data-View.md)

1. There is a secret that is created for elasticsearch after the HELM install. Duplicate the one created by elasticsearch.
2. Change the namespace for `development` to `processor`

**Example**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: elasticsearch-master-certs
  namespace: processor
type: kubernetes.io/tls
data:
  ca.crt: >-
  tls.crt: >-
  tls.key: >-

```

## Setting up TLS for Ingress

Secure your ingress with TLS by creating a tlscomsecret in each required namespace:

You can generate a self-signed certificate and private key with this command;

`openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=${HOST}/O=${HOST}" -addext "subjectAltName = DNS:${HOST}"`

1. Create a secret with your TLS certificate and key;

```nano
apiVersion: v1
kind: Secret
metadata:
  name: tlscomsecret
  namespace: fraudblok-development
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-cert>
  tls.key: <base64-encoded-key>
```

Or

You can use kubectl to create the secret by running the command below;

`kubectl create secret tlscomsecret ${CERT_NAME} --key tls.key --cert tls.crt -n development`

2. Apply this configuration for each relevant namespace (`development`, `processor`, `cicd`, `default`).

## Configuring Ingress Domain Names

Customize your ingress resources to match your domain names and assign them to the nginx-ingress-controller's IP address:

Using `example.test.com` as an example

**Please see the example below:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cicd-ingress
  namespace: fraudblok-cicd
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/backend-protocol: HTTP
    nginx.ingress.kubernetes.io/cors-allow-headers: X-Forwarded-For
    nginx.ingress.kubernetes.io/proxy-body-size: 50m
    nginx.ingress.kubernetes.io/use-regex: 'true'
...
spec:
  tls:
  - hosts:
    - example.test.com
      secretName: tlscomsecret
  rules:
  - host: example.test.com
      ...

```


**Please see the TMS example below:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-ingress
  namespace: processor
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/backend-protocol: HTTP
    nginx.ingress.kubernetes.io/cors-allow-headers: X-Forwarded-For
    nginx.ingress.kubernetes.io/proxy-body-size: 50m
    nginx.ingress.kubernetes.io/use-regex: 'true'
spec:
  tls:
    - hosts:
        - example.test.com
      secretName: tlscomsecret
  ingressClassName: nginx
  rules:
    - host: example.test.com
      http:
        paths:
          - path: /v1/evaluate/pain.001.001.11
            pathType: ImplementationSpecific
            backend:
              service:
                name: transaction-monitoring-service-rel-2-1-0
                port:
                  number: 3000
          - path: /
            pathType: ImplementationSpecific
            backend:
              service:
                name: transaction-monitoring-service-rel-2-1-0
                port:
                  number: 3000
          - path: /v1/evaluate/pacs.008.001.10
            pathType: ImplementationSpecific
            backend:
              service:
                name: transaction-monitoring-service-rel-2-1-0
                port:
                  number: 3000
          - path: /v1/evaluate/pacs.002.001.12
            pathType: ImplementationSpecific
            backend:
              service:
                name: transaction-monitoring-service-rel-2-1-0
                port:
                  number: 3000
          - path: /natsPublish
            pathType: ImplementationSpecific
            backend:
              service:
                name: nats-utilities-rel-1-0-0
                port:
                  number: 3000
```

## Vault Configuration

After installing the Vault chart, you'll need to initialize and unseal Vault manually. The process involves generating unseal keys and a root token which you'll use to access and configure Vault further.

**For now vault has been integrated into the system but the Jenkins variables haven't been add to vault to be pull through . This will be done sometime.**

[https://developer.hashicorp.com/vault/docs/concepts/seal](https://developer.hashicorp.com/vault/docs/concepts/seal)

![](./Images/Vault_Operator_Init.PNG)
![](./Images/Sign_In_With_Token.PNG)

## Logstash Configuration

If is set `LogLevel`to info, error etc.. in your Jenkins environment variables then you will need configure this.

For comprehensive instructions on how to configure logging to Elasticsearch, please refer to the accompanying document. It provides a step-by-step guide that covers all the necessary procedures to ensure your logging system is properly set up, capturing and forwarding logs to Elasticsearch. This includes configuring log shippers, setting up Elasticsearch indices, and establishing the necessary security and access controls. By following this documentation, you can enable efficient log management and monitoring for your services.

[Logging Data View](https://github.com/frmscoe/docs/blob/main/Technical/Logging/Logging-Data-View.md)

## APM Configuration

If is set `APMActive` to **true (default:true)** in your Jenkins environment variables then you will need configure this

Once configured, the APM tool will begin collecting data on application performance metrics, such as response times, error rates, and throughput, which are critical for identifying and resolving performance issues. The collected data is sent to the APM server, where it can be visualized and analyzed. For detailed steps on integrating and configuring APM with your Jenkins environment, please refer to the specific APM setup documentation provided in your APM tool's resources.

[Setting Up Elastic APM](https://github.com/frmscoe/docs/blob/main/Technical/Logging/Setting-Up-Elastic-APM.md)

## Jenkins Configuration

### Accessing Jenkins UI

The following sections of the guide require you to work within the Jenkins UI. You can either access the UI through a doamin if you configured an ingress or by port forwarding.

Port forward Jenkins to be accessible on localhost:8080 by running:
  `kubectl --namespace cicd port-forward svc/jenkins 8080:8080`

Get your 'admin' user password by running:
  `kubectl exec --namespace cicd -it svc/jenkins -c jenkins -- /bin/cat /run/secrets/additional/chart-admin-password && echo`

### Adding Credentials in Jenkins

Credentials are critical for Jenkins to interact with other services like source control management systems (like GitHub), container registries, and Kubernetes clusters securely. Jenkins provides a centralized credentials store where you can manage all these credentials. Here's a step-by-step guide based on the images you've provided:

#### GitHub Credentials

1. **Navigate to Manage Jenkins → Credentials → System → Global credentials (unrestricted).**
2. Click on **Add Credentials.**
3. Select Username with password from the drop-down menu.
4. Enter your GitHub username.
5. Enter your GitHub password. If two-factor authentication is enabled, you'll need to use a personal access token in place of your password.
6. Set the ID to something memorable, like **Github**.
7. Optionally, provide a description like **GitHub Credentials**.
8. Click **Save**.

#### GitHub Read Package Credentials

1. Again, follow the initial steps to reach the Add Credentials page.
2. Select **Secret text**.
3. Enter the personal access token you've created on GitHub with the necessary scopes to read packages.
4. Set the ID, for example, **githubReadPackage**.
5. In the description, note the purpose, such as GitHub package read access.
6. Click **Save**.

#### Kubernetes Credentials

To configure Jenkins to use Kubernetes secrets for authenticating with Kubernetes services or private registries, you can follow these steps, similar to setting up GitHub package read access:

1. **Retrieve the Kubernetes Token**:

- Access your Kubernetes environment and locate the secret intended for Jenkins authentication, in this case, `scjenkins-secret`.
  - Extract the token value from the secret, which is usually base64-encoded. You have need to decode.

1. **Add Secret in Jenkins**:

- Navigate to the Jenkins dashboard and go to the credentials management section.
  - Choose to add new credentials, selecting the "Secret text" type.
  - Paste the token you retrieved from the `scjenkins-secret` in namespace=**processor** into the Secret field.

3. **Configure the Credential ID**:

- Set the ID of the new secret to `kubernetespro`. This ID will be used to reference these credentials within your Jenkins pipelines or job configurations.

4. **Add a Description**:

- Provide a description for the secret to document its use, such as "Token for authenticating Jenkins with Kubernetes services."

5. **Save the Configuration**:

- Click Save to store the new credentials in Jenkins.

Following this process will allow Jenkins jobs to authenticate with Kubernetes using the token stored in the secret, enabling operations that require Kubernetes access or pulling images from private registries linked to your Kubernetes environment.

![image-20240215-150928.png](./Images/image-20240215-150928.png)
![image-20240215-151159.png](./Images/image-20240215-151159.png)

### Adding Managed Files for NPM Configuration in Jenkins

**Navigate to Manage Jenkins → Managed files**

[https://docs.github.com/en/packages/learn-github-packages/about-permissions-for-github-packages#about-scopes-and-permissions-for-package-registries](https://docs.github.com/en/packages/learn-github-packages/about-permissions-for-github-packages#about-scopes-and-permissions-for-package-registries)

[https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token)

![image-20240212-124706.png](./Images/image-20240212-124706.png)

The image shows a Jenkins configuration screen for adding a managed file, specifically an NPM config file (npmrc). Here's a breakdown of the steps and fields:

1. **Managed Files:** This section is for adding configuration files that Jenkins will manage and use across different jobs.
2. **ID:** A unique identifier for the managed file.
3. **Name:** In this case, it's **npmrcRulesCredentials**. This name helps users identify the file's purpose when selecting it for use in a job.
4. **Comment:** An optional field where you can provide additional information about the managed file, such as its intended use. Here, it is described as 'user config'.
5. **Enforce NPM version 9 registry format:** A checkbox that, when checked, enforces the file to be compatible with the NPM version 9 registry format.
6. **Add NPM Registry:**

- **URL:** The registry URL field should be filled with the NPM registry's URL. The provided URL, [**https://npm.pkg.github.com**](https://npm.pkg.github.com), this config is for accessing packages stored in GitHub Package Registry.
- **Credentials:** The dropdown is set to **github public read package**, indicating that the credentials stored in Jenkins should be used to authenticate with this registry.
- **Use this registry for specific scoped packages:** This option indicates that the registry URL and credentials should only be used for packages with a specific scope. In this case, the scope is **frmscoe**.
- **Registry scopes:** Here, you specify the scope for which this registry should be used. Scoped packages are prefixed with the scope in their package name, ie: **frmscoe**.

7. **Content:** The text area labeled 'Content' is where you can input the actual content of the **.npmrc** file. This content typically includes configuration settings like the registry URL, authentication tokens, and various other npm options. **always-auth = false** will not be always required (usually for public registries).

8. **Add:** After configuring all the fields, you would click "Add" to save this managed file configuration.

Once you've added this managed file, Jenkins can use it in various jobs that require npm to access private packages or specific registries. The managed file will be placed in the working directory of the job when it runs, ensuring that npm commands use the provided configuration.

### Adding Managed Files for Container registry in Jenkins

**Navigate to Manage Jenkins → Managed files**

![Jenkins_service account_file.png](./Images/Jenkins_service account_file.png)

The image shows a Jenkins configuration screen for adding a managed file, specifically the service account registry. Here's a breakdown of the steps and fields:

1. **Managed Files:** This section is for adding configuration files that Jenkins will manage and use across different jobs.
2. **ID:** A unique identifier for the managed file. ie: registry
3. **Name:** In this case, it's **ServiceAccountConfig**. This name helps users identify the file's purpose when selecting it for use in a job.
4. **Comment:** An optional field where you can provide additional information about the managed file, such as its intended use. Here, it is described as 'user config'.

5. **Content:** The text area labeled 'Content' is where you can input the actual content of the service acount json for the container registry.

6. **Add:** After configuring all the fields, you would click "Add" to save this managed file configuration.

Once you've added this managed file, Jenkins can use it in various jobs that require npm to access private packages or specific registries. The managed file will be placed in the working directory of the job when it runs, ensuring that npm commands use the provided configuration.

### Jenkins Node.js Configuration

**Navigate to Manage Jenkins → Tools**

- **Name**: Provide a descriptive name for the Node.js installation (e.g., "NodeJS 20.11.0").
- **Install automatically**: Check this option to have Jenkins handle the installation of Node.js automatically on the build agent if it is not already installed.
- **Install from** [http://nodejs.org](http://nodejs.org) : Select this to install Node.js directly from the official Node.js website.
- **Version**: Specify the version of Node.js you require for your projects (e.g., "NodeJS 20.11.0"). Ensure to input a valid version number.
- **Force 32bit architecture**: Check this only if there is a need to install a 32-bit version of Node.js on a 64-bit machine, which is uncommon for most modern applications.
- **Global npm packages to install**: List any npm packages that should be installed globally, **Add** newman
- **Global npm packages refresh hours**: Define how often Jenkins should update the npm package cache. If set to "0", the cache is updated with every build, otherwise, it's updated at the specified interval in hours (e.g., "72" for every three days).

![image-20240215-061559.png](./Images/image-20240215-061559.png)

### Jenkins Docker Installation Configuration

**Navigate to Manage Jenkins → Tools**

- **Name**: Assign a name to the Docker installation to be used as a reference within Jenkins (e.g., "Docker Stable Release").
- **Install automatically**: Enable this setting to have Jenkins take charge of Docker installation on the build agent if Docker isn't already installed.
  - **Download from** [http://docker.com](http://docker.com) : Opt to have Jenkins download Docker directly from the official Docker website.
  - **Docker version**: Specify which version of Docker Jenkins should install. If you want the most recent version, use "latest". Otherwise, provide a specific version number (e.g., "19.03.12").

![image-20240213-103453.png](./Images/image-20240213-103453.png)

### Steps to Configure Jenkins Global Variables

1. **Accessing Global Configuration**:

- Go to the Jenkins dashboard and navigate to **Manage Jenkins** > **Configure System**.
- Scroll down to the **Global properties** section.
- Check the box next to **Environment variables** to enable the definition of global environment variables.

2. **Updating Environment Variables**:

- You will find a list of predefined variables, which you may need to update with new values relevant to the current deployment. These variables include configuration URLs, passwords, and tokens required by Jenkins jobs during the deployment process.

**Passwords:** These passwords can be found in your Kubernetes Cluster Secrets, which are autogenerated when the HELM installations are carried out.

**Multiple ArangoDB passwords and endpoints:** The reason we have different names and passwords for ArangoDB is to keep things organized and safe. Each name points to a different part of the database where different information is kept. Just like having different keys for different rooms. This is useful when you have more than one ArangoDB running at the same time and you want to keep them separate. This way, you can connect to just the part you need.

If you have a single database instance you may be wondering why multiple password variants are needed. For example, if my `Configuration`, `Pseudonyms` and `TransactionHistory` databases are served from the same Arango instance, why must I include single quotes in their password input whereas that requirement was not needed in the `ArangoPassword` variable.

The `ArangoPassword` variable is utilised as a CLI argument by `newman`, for setting up the environment. Where it is called, there is some shell substitution of the `ArangoPassword` variable but because the substitution involves a special character, `$`, that has to be surrounded by quotes. 
`newman {omitted} "arangoPassword=${ArangoPassword}" --disable-unicode`

The same reasoning applies to passwords are that explicitly stated to need a single quote around them as they are substituted as is in processors' environments. This means that if your password contains special characters, then you **must** use single quotes to let the decoder know to interpret them as raw strings, or it will be taken as an indication of substitution

3. **Variables and Descriptions**:

- `APMActive`: A flag to enable or disable Application Performance Monitoring.
  - **Default:** True
  - **value:** true / false
- `ArtifactRegistryZone`: A variable used for the gcloud cli.
  - **eg:** us-central1-docker
- `ArangoPassword`: A secret password required for accessing the Database which is used for populating the Arango configuration.
  - **eg:** rm]ukXyA@M
- `ArangoConfigurationURL`: Endpoint for the ArangoDB configuration Database.
  - **value:** [http://arango.development.svc.cluster.local:8529](http://arango.development.svc.cluster.local:8529)
- `ArangoConfigurationPassword`: A secret password required for accessing the Database. **NB:** The single quotes need to be added with your password.
  - **eg:** 'rm]ukXyA@M'
- `ArangoDbURL`: Endpoint for the ArangoDB configuration Database.
  - **value:** [http://arango.development.svc.cluster.local:8529](http://arango.development.svc.cluster.local:8529)
- `ArangoDbPassword`: A secret password required for accessing the Database. **NB:** The single quotes need to be added with your password.
  - **eg:** 'rm]ukXyA@M'
- `ArangoPseudonymsURL`: Endpoint for the ArangoDB pseudonym Database.
  - **value:** [http://arango.development.svc.cluster.local:8529](http://arango.development.svc.cluster.local:8529)
- `ArangoPseudonymsPassword`: A secret password required for accessing the Database. **NB:** The single quotes need to be added with your password.
  - **eg:** 'rm\]ukXyA@M'
- `ArangoTransactionHistoryURL`: Endpoint for the ArangoDB transaction history Database.
  - **value:** [http://arango.development.svc.cluster.local:8529](http://arango.development.svc.cluster.local:8529)
- `ArangoTransactionHistoryPassword`: A secret password required for accessing the Database. **NB:** The single quotes need to be added with your password.
  - **eg:** 'rm\]ukXyA@M'
- `ArangoEvaluationURL`: Endpoint for the ArangoDB Evalation Database.
  - **value:** [http://arango.development.svc.cluster.local:8529](http://arango.development.svc.cluster.local:8529)
- `ArangoEvaluationPassword`: A secret password required for accessing the Database. **NB:** The single quotes need to be added with your password.
  - **eg:** 'rm\]ukXyA@M'
- `Branch`: The specific branch in source control that the deployment should target.
  - **value:** main
- `CacheEnabled`: A flag to enable or disable caching.
  - **value:** true / false
- `CacheTTL`: Time-to-live for the cache in seconds.
  - **eg:** 86400
- `ELASTIC_HOST`: The hostname for the Elasticsearch service.
  - **value:** [https://elasticsearch-master.development.svc:9200](https://elasticsearch-master.development.svc:9200)
- `ELASTIC_USERNAME`: The username for accessing Elasticsearch.
  - **value:** elastic
- `ELASTIC_PASSWORD`: The password for accessing Elasticsearch.
  - **eg:** ty6&dabnbh
- `ELASTIC_SEARCH_VERSION`: The version of Elasticsearch in use.
  - **value:** 8.5.1
- `EnableQuoting`: A flag to enable or disable quoting functionality adding Pain messages in the chain.
  - **value:** false
- `FLUSHBYTES`: The byte threshold for flushing data.
  - **value:** 10
- `ImageRepository`: The repository for Docker images.
  - **eg:** [example.io](http://example.io)
- `LogLevel`: The verbosity level of logging. **NB:** The single quotes need to be added in.
  - eg: 'info', 'error', 'silent', etc…
- `MaxCPU`: The maximum CPU resource limit.
  - **value:** leave blank
- `NATS_SERVER_TYPE`: The type of NATS server in use.
  - **value:** nats
- `NATS_SERVER_URL`: The URL for the NATS server.
  - **value:** nats.development.svc.cluster.local:4222
- `RedisCluster`: A flag to indicate if Redis is running in cluster mode.
  - **value:** true
- `RedisPassword`: The password for accessing Redis.
  - **eg:** ty6r5\*&p0
- `RedisServers`: The hostname for the Redis Cluster service. **NB:** The single quotes need to be added in to the host string.
  - **value:** '[{"host": "redis-cluster.development.svc.cluster.local", "port":6379}]'
- `Repository`: This parameter specifies the name of a repository
  - **value:** frmscoe

### Adding Jenkins Jobs

#### Download the Job Configurations:

- Provided is a `jobs.zip` file, which contains job configuration files that you need to add to your Jenkins instance.
- Extract the zip file.

[jobs.zip](./Images/jobs.zip)

#### Navigate to Configuration Directory:

- Open a terminal on your local machine and change the directory to where you have stored the `jobs.zip` file and where you unpack it.

```bash
cd <path to configuration>
```

- `<path to configuration>` is a placeholder for the actual directory path where your `jobs.zip` unzipped files are located.

**eg:** cd "C:\Documents\tasks\Jenkins\jobs"

![image-20240220-055517.png](./Images/image-20240220-055517.png)

#### Copy Jobs to Jenkins Pod:

- Use the `kubectl cp` command to copy the job configurations from your local machine to the Jenkins pod running in your Kubernetes cluster.

```bash
kubectl cp . <name of pod>:/var/jenkins_home/jobs/ -n cicd
```

- `<name of pod>` is a placeholder for the actual name of your Jenkins pod. You need to replace it with the correct pod name which you can find by running `kubectl get pods -n cicd`.
- The `-n cicd` specifies the namespace where your Jenkins is deployed, which in this case is `cicd`.

#### Finalize the Setup:

- After copying the job configurations, your Jenkins instance should recognize the new jobs. Jenkins will automatically load job configurations found in the `/var/jenkins_home/jobs/` directory.

### Building Jenkin Agent Locally

**This needs to be completed before adding the Jenkins Cloud agent.**

Please follow the following document to help you build and push the image to the container registry.

[Building the Jenkins Agent Image](https://github.com/frmscoe/docs/blob/main/Technical/Release-Management/building-the-jenkins-image.md) - This link is t show you how to build the docker image , the dockerfile that needs to be used specifically to GC is as follows:

```dockerfile
# Use a base Jenkins agent image
FROM jenkins/inbound-agent:latest as jnlp

USER root

# Update and install necessary packages for adding new repositories
RUN apt-get update && apt-get install -y \
    apt-transport-https \
    gnupg2 \
    curl \
    sudo

# Add Kubernetes package key and set up the repository
RUN echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
RUN curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add the Google Cloud SDK repository
RUN mkdir -p /usr/share/keyrings && \
    curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | gpg --dearmor -o /usr/share/keyrings/cloud.google.gpg && \
    echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" > /etc/apt/sources.list.d/google-cloud-sdk.list

# Update and install required packages
RUN apt-get update

# Install kubectl
RUN apt-get install -qq -y kubectl

# Install Buildah
RUN apt-get install -y buildah

# Install Google Cloud SDK
RUN apt-get install -y google-cloud-sdk

# Switch back to the Jenkins user
USER jenkins
```

### Setting up a Jenkins cloud agent that will interact with your Kubernetes cluster

- **Navigate to Manage Jenkins → Clouds → Kubernetes**
- **Add the Path to Your Kubernetes Instance**: Enter the URL of your Kubernetes API server in the Kubernetes URL field. This allows Jenkins to communicate with your Kubernetes cluster.
- **Disable HTTPS Certificate Check**: If your Kubernetes cluster uses a self-signed certificate or you are in a development environment where certificate validation is not critical, you can disable the HTTPS certificate check. However, for production environments, it is recommended to use a valid SSL certificate and leave this option unchecked for security reasons.
- **Add Kubernetes Namespace**: Enter `cicd` in the Kubernetes Namespace field. This is where your Jenkins agents will run within the Kubernetes cluster.
- **Add Jenkins URL**: This should be the internal service URL for Jenkins within your Kubernetes cluster, like `http://jenkins.cicd.svc.cluster.local:8080`
- **Add Pod Label**: Labels are key-value pairs used for identifying resources within Kubernetes. Here, you should add a label with the key `jenkins` and the value `agent`. This label will be used to associate the built pods with the Jenkins service.

![image-20240212-115316.png](./Images/image-20240212-115316.png)![image-20240212-111931.png](./Images/image-20240212-111931.png)

**Add a Pod Template**: This step involves defining a new pod template, which Jenkins will use to spin up agents on your Kubernetes cluster.

- **Name**: Name the pod template `jenkins-builder`. This name is used to reference the pod template within Jenkins pipelines or job configurations.
- **Namespace**: Specify `cicd` as the namespace where the Jenkins agents will be deployed within the Kubernetes cluster.
- **Labels**: Set `jenkins-agent` as the label. This is a key identifier that Jenkins jobs will use to select this pod template when running builds.

![image-20240212-112102.png](./Images/image-20240212-112102.png)

**Add a Container**: In this part of the configuration, you define the container that will run inside the pod created from the pod template.

**NOTE** This needs to point to the docker image built in this step : [Building the Jenkins Agent Image](https://github.com/frmscoe/docs/blob/main/Technical/Release-Management/building-the-jenkins-image.md)

- **Name**: The container name is set to `jnlp`. This is a conventional name for a Jenkins agent container that uses the JNLP (Java Network Launch Protocol) for the master-agent communication.
- **Docker Image**: The Docker image to use is [example.io/jenkins-inbound-agent:1.0.0](http://example.io/jenkins-inbound-agent:1.0.0) . This image is pre-configured with all the necessary tools to run as a Jenkins agent.
- **Always Pull Image**: This option ensures that Jenkins always pulls the latest version of the specified Docker image before starting a build. This is important to keep your build environment up-to-date with the latest changes to the image.
- **Working Directory**: The working directory is set to `/home/jenkins/agent`. This is the directory inside the container where Jenkins will execute the build steps.
- **Command to Run**: This field is left blank, which means the default command from the Docker image will be used to start the agent.

![image-20240212-115159.png](./Images/image-20240212-115159.png)

**Run in Privileged Mode**: This is an advanced container setting that allows processes within the container to execute with elevated privileges, similar to the root user on a Linux system.

To select "Run in Privileged Mode" in Jenkins Kubernetes plugin:

1. Within the container configuration, look for the "Advanced..." button or link (not visible in the screenshot) and click it to expand the advanced options.
2. In the advanced settings, find the checkbox labeled "Run in privileged mode" and select it.

![image-20240212-114225.png](./Images/image-20240212-114225.png)

**Image Pull Secret**

Needs to be set to - **frmpullsecret - see screenshot below**

1. **Private Registry Authentication**: If the container images used by your Jenkins jobs are hosted in a private registry, Kubernetes needs to authenticate with that registry. The image pull secret stores the required credentials (like a username and password or token).
2. **Adding Image Pull Secret to Pod Template**:

- Navigate to the Kubernetes cloud configuration within the Jenkins system settings.
- Under the specific pod template that you are configuring, find the `ImagePullSecrets` section.
- Enter the name of the Kubernetes secret that contains your private registry credentials in the `Name` field. This secret should already exist within the same namespace as where your Jenkins builder pods are running.
- If you have multiple registries or need to pull from multiple private sources, you can add additional image pull secrets by clicking on the “Add Image Pull Secret” dropdown and entering the names of these secrets.

3. **YAML Merge Strategy**: The YAML merge strategy determines how Jenkins should handle the YAML definitions from inherited pod templates. If set to 'Override', it means that the current YAML will completely replace any inherited YAML, which could be important if you need to ensure that the image pull secrets are applied without being altered by any inherited configurations.

By properly configuring image pull secrets in your Jenkins Kubernetes pod templates, you enable Jenkins to pull the necessary private images to run your builds within the Kubernetes cluster. Without these secrets, the image pull would fail, and your builds would not be able to run.

![image-20240215-144955.png](./Images/image-20240215-144955.png)

#### Reload Jenkins Configuration:

- You might need to manually reload the Jenkins configuration or restart the Jenkins service for the new job configurations to take effect. This can be done from the Jenkins interface or by restarting the Jenkins pod:

```bash
kubectl rollout restart deployment <jenkins-deployment-name> -n cicd
```

- Make sure to replace `<jenkins-deployment-name>` with the actual deployment name of your Jenkins instance.
- You can also safeRestart through the URL.

**eg:** [http://localhost:52933/safeRestart](http://localhost:52933/safeRestart)

![image-20240215-054140.png](./Images/image-20240215-054140.png)

# Step 4 :Running Jenkins Jobs to Install Processors

### Overview

The process involves configuring Jenkins to deploy various processors into the Tazama cluster. These processors are essential components of the system and require specific configurations, such as database connections and service endpoints, to function correctly.

### Populating ArangoDB:

**Dashboard → Deployments→ Applications→ ArangoDB**

Run the `Create Arango Setup` and then `Populate Arango Configuration` jobs to populate the ArangoDB with the correct configuration required by the system. This job would utilize the variables set in the global configuration to connect to ArangoDB and perform the necessary setup.

![image-20240215-052351.png](./Images/image-20240215-052351.png)

### Edit Jobs: Configuring Credentials and Kubernetes Endpoints in Jenkins

After importing the Jenkins jobs, you need to configure each job with the appropriate credentials and Kubernetes server endpoint details. This setup is crucial to ensure that each job has the necessary permissions and access to interact with other services and the Kubernetes cluster.

#### Configuring Rule Processors:

1. **Access Each Rule Processor Job:**

- Navigate to the job configuration for each rule processor, such as TMS, Typology, etc.
- Within each job, look for the section where you can define or edit the repository from which the job will fetch the code or artifacts.

2. **Repository Configuration:**

- Set the **Repository URL** to the Git repository where the code for the processor is located. This is typically a URL like https://github.com/<Repository>/event-director/.
- Under Credentials, select the appropriate credentials from the drop-down list, such as **Github Creds**, which should correspond to the credentials that have access to the repository.

3. **Binding Credentials:**

- Under the **Bindings** section, define the environment variables that the job will use internally.
- For username and password types, such as container registry credentials, set the appropriate **Username Variable** and **Password Variable**. Use **REG\_USER** and **REG\_PASS** for registry credentials.
- Choose the specific credentials from the drop-down list, like **Login info for the Sybrin Azure container registry.**
- For secret texts, such as a GitHub access token, set the Variable to an environment variable name, such as **READ\_GH\_TOKEN**, and select the appropriate credentials, like **github public read package**.

By completing these steps, you ensure that each Jenkins job can access the necessary repositories and services with the correct permissions and interact with your Kubernetes cluster using the right endpoints and credentials. It's essential to review and verify these settings regularly, especially after any changes to the credentials or infrastructure.

![image-20240213-064147.png](./Images/image-20240213-064147.png)
![image-20240213-064707.png](./Images/image-20240213-064707.png)
![image-20240213-064256.png](./Images/image-20240213-064256.png)

### Deploying to the Cluster:

**Dashboard → Deployments → Jenkins Agent → Pipelines→ Deploying All Rules and Rule Processors**

Run the Jenkins jobs that deploy the processors to the Tazama cluster. These jobs will reference the global environment variables you've configured, ensuring that each processor has the required connections and configurations.

Run the **Deploying All Rules and Rule Processors Pipeline Job**

![image-20240212-161403.png](./Images/image-20240212-161403.png)

### End-to-End Platform Testing with the "E2E Test" Jenkins Job

**Dashboard → Testing→ E2E Test**

#### Overview of the "E2E Test" Job

The "E2E Test" job in Jenkins is an essential component for ensuring the integrity and robustness of the platform. It is specifically designed to perform comprehensive end-to-end testing, replicating user behaviors and interactions to verify that every facet of the platform functions together seamlessly.

#### Purpose and Benefits

- **Comprehensive Assessment:** The E2E Test tests all aspects of the platform, from individual services to data processing, ensuring that all integrated components are operating together.
- **Confidence in Releases:** Successful E2E tests will display the stability and readiness of the platform for production releases.

#### Running the Test and Post-Test Evaluation

- **Update your endpoints:** Update both the 'ofUrl' and the 'arangoUrl' used by your ingresses .
- **Test Execution:** Trigger the E2E Test job.
- **Monitoring Results:** Jenkins provides an overview of the job's results, including the last successful run, failures, and test durations.
- **Post-Test Actions:** Upon completion of the E2E Test, it's crucial to examine the **evaluationResults** database within ArangoDB.
- Navigate to the **transactions** collection within the database.
- Confirm the presence of a transaction record, which signifies a successful end-to-end test execution.

If for some reason the E2E jobs doesnt run double check your variables in the configuration of the job, if it still doesnt work you can use postman to test the E2E.

![image-20240213-122427.png](./Images/image-20240213-122427.png)

# Common Errors**

![image-20240213-144301.png](./Images/image-20240213-144301.png)

### Arango ingress error**

To resolve this issue, you would need to:

1. Ensure that the `tlscomsecret` secret contains the necessary TLS certificates and keys.
2. Add the `tlscomsecret` to the `development` namespace, if it's not already present.
3. After the secret is correctly placed in the namespace, restart the affected pod by deleting the existing pod. Kubernetes will automatically spin up a new pod which should now successfully mount the required volumes, including the TLS secrets, and run as expected.

![image-20240215-142735.png](./Images/image-20240215-142735.png)![image-20240215-142727.png](./Images/image-20240215-142727.png)

### Network Access Error in Container Deployment

To address the network access error encountered when deploying containers that require communication with `arango.development.svc`, follow these steps:

1. Verify that the network policies and service discovery configurations are correctly set up within your cluster to allow connectivity to `arango.development.svc`.

2. If your deployment is within a Kubernetes environment and you're using network namespaces, consider enabling the Host Network option. This grants the pod access to the host machine's network stack, which can be necessary if the service is only resolvable or accessible in the host's network:

- Navigate to the configuration settings of your pod or service deployment.
- Locate the "Host Network" option, which allows the pod to use the network namespace of the host machine.
- Enable the "Host Network" checkbox to allow direct access to the network services on the host, which can resolve DNS issues if `arango.development.svc` is only available on the host's network.

Implementing these steps should help in resolving connectivity issues related to the `arango.development.svc` hostname not being found, facilitating successful POST requests to the specified endpoints.

![image-20240215-143113.png](./Images/image-20240215-143113.png)

### Addressing Pod Restart Issues in Kubernetes

If you are experiencing problems with your Kubernetes pods that may be related to environmental variables or configuration issues, such as frequent restarts or failed connections to services like ArangoDB, follow these steps to troubleshoot and resolve the issue:

1. Check the Environment Variables in Jenkins:

- Ensure that all required environment variables are properly set in Jenkins. These variables might include database connection strings, service endpoints, credentials, or other configuration parameters necessary for your application to run correctly.
- Review the build and deployment scripts in Jenkins to confirm that the environment variables are being injected into the deployment manifests or pod configurations.

2. Verify ArangoDB Configuration:

- Double-check the ArangoDB configuration to ensure that it is correct and aligns with the requirements of your application. This may include database URLs, user credentials, database names, and any other related configuration details.
- If you are using Kubernetes ConfigMaps or Secrets to manage the ArangoDB configuration, make sure they are correctly defined and mounted into your pods.

3. Monitor Pod Status and Logs:

- Observe the status of the pods through the Kubernetes dashboard or using `kubectl get pods` command. Take note of any pods that are in a CrashLoopBackOff state or that are frequently restarting.
- Use `kubectl describe pod <pod-name>` to get more details about the pod's state and events that might indicate what is causing the restarts.
- Examine the logs of the restarting pods using `kubectl logs <pod-name>` to look for any error messages or stack traces that could point to a configuration problem or a missing environment variable.

4. Address Possible Configuration Drifts:

- In a dynamic environment like Kubernetes, configuration drifts can occur where the running state of the system deviates from the defined state. Ensure that all deployments, StatefulSets, or other controller resources match the intended configuration.

5. Update and Restart Pods if Necessary:

- Once any necessary changes have been made to the environment variables or ArangoDB configuration, update the relevant Kubernetes resources.
- You can restart the affected pods to apply the changes by deleting them and letting the ReplicaSet create new ones with the correct configuration.

By carefully checking your Jenkins environment variables and ensuring the ArangoDB configuration is correct, you can resolve issues leading to pod instability and ensure that your services run smoothly in the Kubernetes environment.

![image-20240215-143411.png](./Images/image-20240215-143411.png)
![image-20240215-143443.png](./Images/image-20240215-143443.png)
![image-20240215-143505.png](./Images/image-20240215-143505.png)

### Addressing Jenkins Build Authentication Errors

When encountering authentication errors during a Jenkins build process that involve Kubernetes plugin issues or Docker image push failures, follow these troubleshooting steps:

1. Kubernetes Plugin Error:

- The error message suggests a `NullPointerException`, which is often due to missing or improperly configured credentials within Jenkins. This could be an issue with the Kubernetes plugin configuration where a required value is not being set, resulting in a null being passed where an object is expected.
- Review your Jenkins job configurations and ensure that all the Kubernetes-related credentials are correctly set and that the plugin is properly configured.
- If you are using credential substitution (injected variables), ensure that the substitutions are correctly configured. If necessary, as per the provided instruction, deselect all credential substitutions to see if this resolves the error. This can help isolate the issue by reverting to default or hardcoded credentials, which can then be individually reinstated to identify the problematic substitution.

2. Docker Image Push Error:

- The failure to push a Docker image to a registry, with an error indicating "unable to retrieve auth token," typically points to incorrect credentials being used for Docker registry authentication.
- Confirm that the Docker registry credentials set in Jenkins are accurate. You may need to update the username and password or use an access token if the registry requires it.
- Ensure that the credentials are correctly mapped in the Jenkins job and that any credential substitution is correctly applied.

3. Rerun the Jenkins Jobs:

- After making the necessary corrections, rerun the Jenkins jobs to confirm that the issue is resolved.
- Monitor the build output for any further authentication-related errors and address them as needed.

4. Additional Steps:

- If the issue persists, consider regenerating or re-obtaining the necessary credentials and updating them in Jenkins.
- Check the Jenkins system logs and the specific job's console output for more detailed error messages that can provide additional context for the failure.

By following these steps, you can address the authentication issues that are causing the Jenkins build process to fail, ensuring a successful connection to Kubernetes and Docker registry services.

### Jenkins Build Agent terminating and restarting

If for some reason the jenkins agent starts up on your kubernetes instance and then termnates and restarts. You might need to change to frmpullsecret with namespace`cicd`to .dockerconfigjson data instead of the AWS data.

**Docker Config JSON: Understanding the** `auth` **Field**

The `auth` field in the `.dockerconfigjson` file is a base64-encoded string that combines your Docker registry username and password in the format `username:password`. Here's how you can construct it:

**Steps to Construct the** `auth` **Field**

1. **Combine the Username and Password**

   Format the string as `username:password`. For example, your username is `frms` and your password is `yourpassword`.

2. **Base64 Encode the String**

You can use a command-line tool like `base64` or an online base64 encoder to encode the string.

Using a command-line tool:

```sh
echo -n 'frms:yourpassword' | base64
```

This will produce a base64-encoded string, which you then place in the auth field.

Here is an example of what the .dockerconfigjson data in the secret file might look like after encoding:

**NB: The vales below come from your service account json that you download from GC**

```json
{"auths":{"gcr.io":{"username":"<private_key_id>","password":"<PRIVATE KEY>","email":"no@email.local"}}}
```

**Please see the example below:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: frmpullsecret
  namespace: fraudblok-cicd
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: >-
```

### Forbidden user on Jenkins job builds to deploy/restart pods 

![Jenkins_service account_error.png](./Images/Jenkins_service account_error.png)

The error indicates that the Kubernetes service account `system:serviceaccount:cicd:default` does not have the necessary permissions to access the `deployments` resource in the `apps` group in the `processor` namespace.

You can fix this by creating a RoleBinding or ClusterRoleBinding to grant the required permissions. Here's how:

#### For a Namespace-Scoped Role:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: processor
  name: deployment-access
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "delete", "patch"]
```

#### For a Cluster-Wide ClusterRole:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: deployment-access
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "delete", "patch"]
```

#### RoleBinding or ClusterRoleBinding
Ensure your Role or ClusterRole is bound to the `cicd` service account. If it's already bound, you don’t need to change this part:

#### For Namespace-Scoped RoleBinding:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: processor
  name: deployment-access-binding
subjects:
- kind: ServiceAccount
  name: default
  namespace: fraudblok-cicd
roleRef:
  kind: Role
  name: deployment-access
  apiGroup: rbac.authorization.k8s.io
```

#### For Cluster-Wide ClusterRoleBinding:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: deployment-access-binding
subjects:
- kind: ServiceAccount
  name: default
  namespace: fraudblok-cicd
roleRef:
  kind: ClusterRole
  name: deployment-access
  apiGroup: rbac.authorization.k8s.io
```

### Apply the Updated RBAC Configuration
Save the YAML files and apply them with the following commands:
```bash
kubectl apply -f role.yaml
kubectl apply -f rolebinding.yaml
```
Or, if you’re using a ClusterRole:
```bash
kubectl apply -f clusterrole.yaml
kubectl apply -f clusterrolebinding.yaml
```

This will resolve the access issue and allow the `default` service account in the `cicd` namespace to interact with `deployments` in the `processor` namespace (or all namespaces, depending on your approach).

# Conclusion: Finalizing Tazama System Installation

With the Helm charts and Jenkins jobs successfully executed, your Tazama (Real-time Monitoring System) should now be operational within your Kubernetes cluster. This comprehensive setup leverages the robust capabilities of Kubernetes orchestrated by Jenkins automation to ensure a seamless deployment process.

As you navigate through the use and potential customization of the Tazama system, keep in mind the importance of maintaining the configurations as documented in this guide. Regularly update your environment variables, manage your credentials securely, and ensure that the pipeline scripts are kept up-to-date with any changes in your infrastructure or workflows.

Should you encounter any issues or have questions regarding the installation and configuration of the Tazama system, support is readily available. You can reach out via email or join the dedicated Slack workspace for collaborative troubleshooting and community support.

For direct assistance:

- Slack: [Tazama.slack.com](http://Tazama.slack.com)

Joining the Tazama CoE workspace on Slack will connect you with a community of experts and peers who can offer insights and help you leverage the full potential of your Tazama system. Always ensure that you are working within secure communication channels and handling sensitive information with care.