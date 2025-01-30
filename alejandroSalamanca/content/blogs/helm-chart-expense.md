---
title: "Part 3: Deploying with Helm in Kubernetes"
date: 2025-01-30T10:11:27-05:00
draft: false
image: https://upload.wikimedia.org/wikipedia/en/thumb/5/5e/Helm_%28package_manager%29_logo.svg/100px-Helm_%28package_manager%29_logo.svg.png
author: "Alejandro Salamanca"
github_link: "https://gitlab.com/devopsdemo2374651/CurrencyConversion"
tags:
  - Helm
  - Kubernetes
  - Go template
---

[GitLab Repository](https://gitlab.com/devopsdemo2374651/CurrencyConversion)
#### Building a Cloud-Native Application: My Journey with Go, GitLab CI, Helm, and CNPG  



#### Introduction  
This project is an application I created to improve my skills in building cloud-native applications. The idea behind this project was to explore key concepts that are essential in today’s software development landscape: developing a REST API in Go, setting up a basic CI/CD pipeline using GitLab, creating Helm charts for Kubernetes deployments, and learning how to self-manage a PostgreSQL database in Kubernetes using the CNPG operator.

I started this project with the goal of honing my skills in Go, a language I’ve been eager to master for its speed, simplicity, and efficiency in building modern applications. Alongside that, I wanted to dive into GitLab CI to understand how pipelines streamline the development process, ensuring automated testing, building, and deployment. Learning the basics of Helm was a natural extension of this journey, as deploying applications in Kubernetes has become an industry standard. Finally, managing a PostgreSQL database in Kubernetes using CNPG (Cloud Native PostgreSQL) taught me the fundamentals of operating databases in a cloud-native environment.

In this blog series, I’ll take you through each step of the process, showing how the project came together and sharing the lessons I learned along the way. I hope this inspires others to take on similar challenges to grow in their professional careers and expand their skill sets in building cloud-native applications.

---

#### Structure of the Blog Series  

This blog will be divided into four parts, each focusing on a critical aspect of the project:  

**Part 1: Building the REST API with Go**  
   In the first part, I’ll cover how I built a REST API using the Go programming language and the Gin framework. This section will include the project structure, key features, and how the API handles user registration and authentication.  

**Part 2: Setting Up a CI/CD Pipeline with GitLab CI**  
   The second part will focus on implementing a CI/CD pipeline in GitLab. I’ll show how the pipeline was configured to automate tasks such as testing, building, and containerizing the application.  

**Part 3: Deploying with Helm in Kubernetes**  
   In the third part, I’ll walk you through the basics of creating a Helm chart to deploy the application to a Kubernetes cluster. This section will demonstrate how Helm simplifies application deployment and management in a cloud-native environment.  

**Part 4: Managing PostgreSQL with CNPG Operator**  
   In the final part, I’ll discuss how I used the CNPG (Cloud Native PostgreSQL) operator to set up and manage a self-managed PostgreSQL database in Kubernetes. This part will explore the operator’s features, such as high availability, backups, and monitoring, and how it integrated seamlessly with the application.  

---

### Part 3: Deploying with Helm in Kubernetes

In the first two parts of this series, I focused on the basics of creating a REST API, and building a CI/CD pipeline to test it, identifying security issues, and packaging it for deployment. Now, in this part, let's shift our focus to deployment by creating a Helm chart.    

Helm is a powerful tool that helps developers and DevOps teams package, deploy, and manage applications in Kubernetes. A **Helm chart** is essentially a package that contains YAML files and templates defining the necessary configuration for deploying an application in a Kubernetes cluster. Helm simplifies installation, upgrades, and rollbacks, making Kubernetes application management much more efficient. It is built on top of Go templates, allowing for flexible and reusable configurations.  

The core components of a Helm chart include:  
- **`Chart.yaml`** – Defines the metadata of the Helm chart, such as the name, version, and description.  
- **`values.yaml`** – Contains configurable values that can be substituted into templates, allowing for customization and reuse.  
- **`templates/` directory** – Stores Kubernetes resource definitions (Deployments, Services, Ingress, etc.), which Helm processes using the defined values.  
- **`charts/` directory** – Holds dependencies for the Helm chart, allowing modular and scalable configurations.  

In this part, I’ll walk you through creating a Helm chart for our application, explaining each step along the way. Let's get started!  

### Setting Up Helm and Creating a Helm Chart  

#### Installing Helm  
Before we create a Helm chart, we need to install Helm on our local machine. There are multiple installation methods available, so I recommend checking the official documentation for the latest options: [Helm Installation Guide](https://helm.sh/docs/intro/install/).  

For this setup, I’ll use the script-based installation method:  

```sh
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```  

Once Helm is installed, we’re ready to create our Helm chart.  

### Creating a Helm Chart for the Application  
To generate a new Helm chart, we use the following command:  

```sh
helm create <CHART-NAME>
```  

In my case, I run:  

```sh
helm create expense-tracker
```  

This command creates a directory called `expense-tracker` containing a demo Helm chart with a default Nginx application. Now, we can start customizing it for our application.  

#### Customizing the Helm Chart  
The first file to modify is `Chart.yaml`, which contains metadata about the Helm chart. Here’s how I updated mine:  

```yaml
apiVersion: v2
name: expense-tracker
description: A Helm chart for Kubernetes, to manage my app about a personal expense tracker made in Go.

type: application

version: 0.1.5

appVersion: "1.16.0"

maintainers:
  - email: aleelaleyi@gmail.com
    name: Alejandro Salamanca Lozano
```  

After these modifications, the directory structure of my Helm chart looks like this:  

```
expense-tracker/
├── Chart.yaml
├── charts/
├── templates/
│   ├── _helpers.tpl
│   ├── cluster-cnpg.yaml
│   ├── configmap.yaml
│   ├── deployment.yaml
│   ├── ingress.yaml
│   ├── init-sql-configmap.yaml
│   ├── secrets.yaml
│   ├── service.yaml
│   └── tests/
│       └── test-connection.yaml
└── values.yaml
```

With this setup, we have the foundation of a Helm chart that we can further customize to define Kubernetes resources such as Deployments, Services, ConfigMaps, and Secrets.  


#### Configuring Helm Templates for Kubernetes Manifests  

Now, let’s dive into how I set up each template to generate the necessary Kubernetes manifests. As you can see in the directory structure, there are essential resources like:  

- **Deployment** (`deployment.yaml`) – Defines how the application runs in the cluster.  
- **Service** (`service.yaml`) – Exposes the application to other services or external users.  
- **ConfigMap** (`configmap.yaml`) – Stores environment variables and configuration files.  
- **Secret** (`secrets.yaml`) – Manages sensitive information such as API keys and credentials.  

Additionally, you may have noticed two files related to the database:  

- **`cluster-cnpg.yaml`** – Manages the database cluster using CloudNativePG (CNPG).  
- **`init-sql-configmap.yaml`** – Contains initialization scripts for setting up the database.  

Since these files are specific to database management, I won’t cover them in this blog post. Instead, I’ll save them for **Part 4**, where I’ll focus on setting up and integrating the database with our application.  

Now, let’s go step by step into configuring each of the core templates in the Helm chart.  


#### ConfigMap and Secret for Environment Variables  

Let’s start with the basic resources: the **ConfigMap** and **Secret**. If you recall from the Go application, we need three environment variables for the application to run properly:  

1. **`PORT`** – Defines the port where the application will run.  
2. **`JWT_KEY`** – The secret key used for signing JWT tokens.  
3. **`DB_URI`** – The database connection string (which we will cover later).  

To inject these values into the container as environment variables, we define the following templates:  

#### `configmap.yaml`  

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Chart.Name }}-config
data:
  PORT: "{{ .Values.config.port }}"
```

This **ConfigMap** allows us to store the application's port configuration, making it easy to modify without redeploying the application.  

#### `secret.yaml`  

```yaml
{{- $existingSecret := lookup "v1" "Secret" .Release.Namespace "{{$.Chart.Name}}-secret" }}
{{- if not $existingSecret }}
apiVersion: v1
kind: Secret
metadata:
  name: {{ .Chart.Name }}-secret
type: Opaque
data:
  JWT_KEY: {{ randAlphaNum 32 | b64enc | quote }}
{{- end }}
```

Here, we use two interesting Helm functions:  

1. **`lookup`** – This checks if the secret already exists in the namespace. If it does, Helm will **not** attempt to create it again.  
2. **`randAlphaNum 32`** – Generates a random 32-character alphanumeric string for the `JWT_KEY`, which is then base64-encoded (`b64enc`).  

#### Considerations for Production  

Since this is a demo application, generating a random secret key dynamically works fine. However, in a **production** environment, it's better to use a dedicated secret management tool such as:  

- **External Secrets Operator** (ESO) – Integrates with external secret managers.  
- **Vault** – HashiCorp’s secrets management tool.  
- **AWS KMS / Secrets Manager** – Secure key storage for AWS-based applications.  

By using these tools, you can manage sensitive credentials securely and rotate secrets when needed.  

#### Deployment Configuration  

The **Deployment** file defines how our application runs in the Kubernetes cluster. Below is the `deployment.yaml` template:  

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Chart.Name }}
  labels:
    app: {{ .Chart.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: {{ .Values.service.port }}
          env:
            - name: DATABASE_URI
              valueFrom:
                secretKeyRef:
                  name: {{ .Values.cnpg.clusterName }}-app
                  key: {{ .Values.secretPlaceholder.secretKey }}
            - name: JWT_KEY
              valueFrom:
                secretKeyRef:
                  name: {{ .Chart.Name }}-secret
                  key: JWT_KEY
            - name: PORT
              valueFrom:
                configMapKeyRef:
                  name: {{ .Chart.Name }}-config
                  key: PORT
```

#### Key Elements in the Deployment  

- **Metadata Naming**: The deployment name follows the **Helm chart name**, but in a production setup, it’s recommended to **include the release name** using `{{ .Release.Name }}` for better differentiation across environments.  
- **Replica Count**: The number of replicas is dynamically set from `values.yaml` using `{{ .Values.replicaCount }}`, allowing easy scaling.  
- **Templating with Helm**: Helm allows defining values in `values.yaml` to make configurations more flexible.  
- **Environment Variables**:  
  - `DATABASE_URI` comes from a **secret generated by the CNPG operator** (we will cover this in the next part).  
  - `JWT_KEY` is injected from the **previously defined Helm secret**.  
  - `PORT` is set from the **ConfigMap** we created earlier.  

Helm acts as a **templating tool** for Kubernetes manifests. Instead of manually defining values in YAML files, we create templates that dynamically adapt to different environments through `values.yaml`. This improves **maintainability, consistency, and reusability** when deploying applications.  

#### Service Configuration (`service.yaml`)  

The **Service** in Kubernetes is responsible for exposing the application within the cluster and enabling communication between different components. Below is the `service.yaml` template:  

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Chart.Name }}
  labels:
    app: {{ .Chart.Name }}
    service: {{ .Chart.Name }}
spec:
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.port }}
      name: http
  selector:
    app: {{ .Chart.Name }}
```

#### What Does This Service Do?  

This Service acts as a **stable network endpoint** for the application’s **Pods** running in Kubernetes. Since Pods are ephemeral (they can be restarted or moved to another node), using a Service ensures that other components can reliably communicate with the application **without worrying about Pod IPs changing**.  

#### How Does It Connect to the Deployment?  

**Selector (`spec.selector`)**:  
The Service routes traffic to Pods that match the labels defined here:  
    ```yaml
    selector:
      app: {{ .Chart.Name }}
    ```
This means that any **Pod** that has the label `app: {{ .Chart.Name }}` (which we defined in the **Deployment**) will receive traffic from this Service.  

 **Ports (`spec.ports`)**:  
  - The `port` value (from `values.yaml`) defines the port **on the Service** that will receive traffic.  
  - `targetPort` is the port **inside the Pod** where the application is listening.  
  - Since both values are set to `{{ .Values.service.port }}`, traffic received on the Service will be forwarded to the same port inside the Pod.  

This setup allows seamless **scalability**—if new Pods are created or old ones are removed, the Service dynamically updates the connections without any manual changes.

#### Understanding the Ingress Manifest

The Ingress resource in Kubernetes is crucial for managing external access to services inside the cluster, typically via HTTP(S). Instead of exposing each service individually, an Ingress Controller handles all incoming traffic and routes it accordingly.

#### **Ingress Manifest Explanation**  

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Chart.Name }}-ingress
  labels:
    app: {{ .Chart.Name }}
  annotations:
    {{- with .Values.ingress.annotations }}
    {{- toYaml . | nindent 4 }}
    {{- end }}
spec:
  rules:
  - http:
      paths:
      {{- range .Values.ingress.paths }}
      - path: {{ .path }}
        pathType: {{ .pathType }}
        backend:
          service:
            name: {{ $.Chart.Name }}
            port:
              number: {{$.Values.service.port}} 
      {{- end }}
{{- end }}
```

As you can see, in the first line, this template will **only be generated** if the `values.yaml` file has the key `ingress.enabled` set to `true`.  

Another important thing about this file is the **range function**. This works as a loop, similar to a `for` statement. For example, if we have the following in the `values.yaml` file:  

```yaml
ingress:
  enabled: true
  annotations:
    ingress.kubernetes.io/ssl-redirect: "false"
  paths:
    - path: /login
      pathType: "Prefix"
    - path: /users/register
      pathType: "Prefix"
    - path: /users
      pathType: "Prefix"
    - path: /user/
      pathType: "Prefix"
    - path: /user/update/:id
      pathType: "Prefix"
    - path: /user/delete/:id
      pathType: "Prefix"
    - path: /add
      pathType: "Prefix"
    - path: /list
      pathType: "Prefix"
    - path: /expense
      pathType: "Prefix"
    - path: /update/
      pathType: "Prefix"
    - path: /delete/
      pathType: "Prefix"
```

The **range function** helps us avoid defining each path one by one in the manifest, making it seamless to generate the whole Ingress manifest dynamically.  

Also, you may notice that inside the range, we use the **`$` character**, but it was not needed before. This is because, in Go templating, inside a `range`, the scope of variables is lost. That’s why we need to use `$` to reference the **full scope of the template**.


### Deploying the Application to a Kubernetes Cluster  

There are several ways to deploy the application in a Kubernetes cluster.  

#### **1. Manual Deployment**  

This is the easiest method and is quite useful for **local development and testing**. The way to do it is with the command:  

```sh
helm install <release> <path/to/helm/chart> [flags]
```

After running this command, you will get a message indicating the result of the installation. You can then use **kubectl commands** to track the state of the deployment.  

It's also important to mention the **`helm upgrade`** command. This is useful when the Helm chart is already installed and you want to upgrade it to a **new version**.  

#### **2. Deploying via a CI/CD Pipeline**  

Another way to deploy the application is through a **CI/CD pipeline**. Let’s take an example where you have a **Kubernetes cluster** running on **AWS, Azure, GCP, or any public cloud provider of your choice**.  

Since we are using **GitLab**, we can automate the deployment with the **`.gitlab-ci.yml`** file. If you recall the previous section, we have **two jobs** that help us **test and package** the Helm chart. After these jobs, we can have another job to execute the necessary **commands** to deploy the chart.  

In GitLab, this is straightforward because it provides a **built-in integration** to connect with a Kubernetes cluster. You can learn more about it here:  

🔗 [GitLab Kubernetes Agent Documentation](https://docs.gitlab.com/ee/user/clusters/agent/)  

Once the connection is set up, you can run a command like:  

```sh
helm upgrade <release> <path/to/helm/chart> [flags]
```

This allows you to **automatically upgrade** the Helm chart every time a **new version** is created.  

**However, in my case, I don't do it that way because it's a paid feature** *jeje* 😆


#### **3. Adopting a GitOps Approach**

The third option is to adopt a **GitOps approach**. The two main GitOps tools nowadays are **Argo CD** and **Flux CD**. In my case, I have only worked with **ArgoCD**.  

With the GitOps approach, you have two options for deploying the Helm chart:  

1. **Point to a Git Repository**: You can point ArgoCD to a Git repository where the Helm chart is defined.  
2. **Point to a Helm Repository**: Alternatively, you can point ArgoCD to a Helm repository, like the one provided by GitLab or GitHub.  

For more information, you can refer to the official ArgoCD documentation:  

🔗 [ArgoCD Helm Documentation](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/)

### Final thoughts

I hope you found this blog helpful and enjoyed learning about setting up a Helm chart for deploying applications in Kubernetes! This post is the third part of a series where I explore Helm, Kubernetes, CI/CD pipelines, and more advanced topics in cloud-native development. If you have any questions, feedback, or suggestions, feel free to reach out—I’d love to hear from you!

This is my first deep dive into Helm, and while there’s always room for improvement, I’m excited to continue refining my deployment strategies. Stay tuned for more content, and thank you for reading!