---
title: "Part 2: Setting Up a CI/CD Pipeline with GitLab CI"
date: 2025-01-25T15:44:45-05:00
draft: false
image: https://upload.wikimedia.org/wikipedia/commons/thumb/c/c8/GitLab_logo_%282%29.svg/1024px-GitLab_logo_%282%29.svg.png
author: "Alejandro Salamanca"
github_link: "https://gitlab.com/devopsdemo2374651/CurrencyConversion"
tags:
  - Gitlab
  - Git
  - CI/CD
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

### Part 2: Setting Up a CI/CD Pipeline with GitLab CI

In today’s software industry, Continuous Integration and Continuous Delivery (CI/CD) are essential for every team or company. These pipelines help developers automate processes like testing, building, and deployment, making it easier to deliver software in a faster and more reliable way.

For this project, I chose to use GitLab to implement the CI/CD pipeline. GitLab is a platform that provides exceptional flexibility for development teams. In simple terms, it allows you to host Git repositories while also offering a comprehensive solution to adopt DevOps principles. GitLab helps manage the entire Software Development Lifecycle (SDLC) in one place. To achieve this, GitLab provides the `.gitlab-ci.yml` file, where all pipeline configurations can be defined in a single file. Additionally, it offers services like a container registry, package registry, and even security scanning to enhance the development workflow.

When deciding which CI/CD platform to use for this project, GitLab stood out to me because of its growing popularity in the software industry. Many companies rely on it for their development workflows, making it an important tool for aspiring developers like myself to learn. By exploring GitLab CI, I aimed not only to streamline the development process for this project but also to enhance my professional skill set and prepare for future career opportunities.

### Basics of GitLab CI

GitLab CI provides a simple yet powerful way to automate your workflows using pipelines. At its core, GitLab CI pipelines operate on top of Docker containers, making them highly flexible and reproducible. Each pipeline consists of stages and jobs, with configuration defined in the .gitlab-ci.yml file. Let’s break down the key components:

**Run on Containers**:  
Every job in a GitLab CI pipeline runs inside a container, which acts as the environment for each job. You can specify a single image for the entire pipeline, or you can define different images for each stage or job, depending on the use case. This flexibility allows you to tailor the environment to the specific needs of your workflow.  

**Jobs**:  
Jobs are individual tasks that run within the pipeline. Each job belongs to a stage and can be customized with specific configurations, making them the most critical part of the pipeline where all the "magic" happens. For example, a job might compile code, run tests, or deploy an application.  

**Stages**:  
Stages define the lifecycle of the pipeline and determine the order in which jobs are executed. Stages are executed sequentially, meaning all jobs within a stage must finish before the next stage begins. This ensures a logical flow for the pipeline.  


**Variables**:  
Variables allow you to store values that can be reused across the pipeline. They can be defined at the project level in the GitLab UI or directly in the `.gitlab-ci.yml` file. GitLab also provides predefined variables, such as `$CI_JOB_STAGE`, which can provide contextual information about the current pipeline. To learn more about variables, you can refer to the [official GitLab documentation](https://docs.gitlab.com/ee/ci/variables/).  

### Gitlab pipeline

This is the `.gitlab-ci.yml` file I created to define the CI/CD pipeline for my project. The pipeline is divided into **4 stages**, which are essential for maintaining organization and ensuring a logical flow of processes:


#### **Stages Overview**
```yaml
stages:
  - test
  - security
  - build
  - upload
```

In my case, I prepared four stages: `test`, `security`, `build`, and `upload`. These stages help break the pipeline into manageable steps and provide a clear structure to the YAML file, making it easier to maintain and extend as needed.


### **Stage 1: Test**
The first part of the pipeline focuses on code analysis and testing to ensure the codebase is clean and follows best practices.

#### **Format Job**
```yaml
format:
  stage: test
  script:
    - go vet ./...
  only:
    changes:
      - '**/*.go'
```

In this job, I used the **`go vet`** tool. This functionality helps analyze the code and check for potential issues or incorrect usage of Go language features. For example, it can catch bugs related to incorrect argument types, bad format strings, or invalid method calls.

Additionally, if I were to include unit tests in my code, I could create another test job using the **`go test`** command. This would allow me to run all the unit tests and verify the correctness of my code.


#### **Helm Lint Job**

```yaml
test_helm:
  image: dtzar/helm-kubectl
  stage: test
  script:
    - helm lint expense-tracker/
  only:
    changes:
      - expense-tracker/**
```

This job validates the syntax and structure of a Helm chart using the helm lint command. Helm charts are configuration templates used for Kubernetes deployments. This job ensures the chart is free from syntax errors or misconfigurations. While I am still exploring Helm charts, this job provides a solid foundation for their validation, and I plan to delve deeper into this topic in future posts.


### **Stage 2: Security**

The **security stage** focuses on identifying potential vulnerabilities in the code. This stage includes two jobs:

#### **Govulncheck Job**
```yaml
govulncheck:
  stage: security
  before_script:
    - go install golang.org/x/vuln/cmd/govulncheck@latest
  script:
    - govulncheck ./...
  only:
    changes:
      - '**/*.go'
```
In this job, I use the **`govulncheck`** tool, which is part of the Go vulnerability management system. It analyzes the dependencies of a Go project and checks for known security vulnerabilities. The **`before_script`** installs the tool, and the **`script`** runs the command **`govulncheck ./...`**, scanning the project for any vulnerable dependencies or insecure code.

#### **Gosec Job**
```yaml
gosec:
  stage: security
  before_script:
    - go install github.com/securego/gosec/v2/cmd/gosec@latest
  script:
    - gosec ./...
  only:
    changes:
      - '**/*.go'
```
The second job uses the **`gosec`** utility, a static analysis tool designed to identify security issues in Go code. It scans the source code for common vulnerabilities such as hardcoded credentials, unsafe file operations, or improper use of cryptographic libraries. Like the previous job, it includes a **`before_script`** to install the tool and a **`script`** to execute **`gosec ./...`**, ensuring the code is secure and adheres to best practices.

### **Stage 3: Build Stage**

In this stage, the job is responsible for building and deploying a Docker image for the application. The Docker image is defined as follows:

```dockerfile
FROM cgr.dev/chainguard/go AS builder

WORKDIR /app

COPY go.mod go.sum ./

RUN go mod download

COPY . .

RUN go build -o main cmd/main.go

FROM cgr.dev/chainguard/glibc-dynamic

COPY --from=builder /app/main /usr/bin/

EXPOSE 8081

ENTRYPOINT ["/usr/bin/main"]
```

This Dockerfile uses Chainguard images as its base. While the details of these images go beyond the scope of this post, Chainguard provides highly secure container images. If you're interested in learning more, check out the [Chainguard images documentation](https://www.chainguard.dev/chainguard-images), as they claim to deliver some of the most secure container images available.

To create a Docker image within GitLab, things get interesting because the pipeline itself runs inside Docker, but now Docker needs to run to generate the Docker image. This scenario is called **Docker-in-Docker** (DinD). 

To enable this in GitLab, it’s crucial to define the following variables:

```yaml
variables:
  DOCKER_HOST: tcp://docker:2376
  DOCKER_TLS_CERTDIR: "/certs"
```

Additionally, you need to specify the service `docker:27.5.0-dind` in your job. This service allows Docker to run inside the pipeline, making it possible to build and push images.

Here’s the job for this stage:

```yaml
build_push_image:
  image: docker:27.5.0
  services:
    - docker:27.5.0-dind
  stage: build
  script:
    - docker login -u $DOCKER_REGISTRY_USER -p $DOCKER_REGISTRY_TOKEN
    - docker build -t $DOCKER_REGISTRY_USER/expenses-tracker:$CI_COMMIT_SHORT_SHA .
    - docker push $DOCKER_REGISTRY_USER/expenses-tracker:$CI_COMMIT_SHORT_SHA
  only:
    changes:
      - '**/*.go'
      - Dockerfile
```

In this job:
- **`$DOCKER_REGISTRY_USER`** and **`$DOCKER_REGISTRY_TOKEN`** are user-defined variables used for Docker authentication.
- **`$CI_COMMIT_SHORT_SHA`** is a GitLab-provided variable that represents the short commit hash.

The `docker login` command authenticates with the Docker registry, `docker build` creates the image and tags it using the commit hash, and `docker push` uploads the image to the registry. These steps ensure that the pipeline produces a fresh Docker image for every change.

### **Stage 4: Upload Stage**

In the final stage of the pipeline, the goal is to package and upload a Helm chart to a GitLab repository. The job for this stage is as follows:

```yaml
package_and_upload_helm:
  image: dtzar/helm-kubectl
  stage: upload
  before_script:
    - helm plugin install https://github.com/chartmuseum/helm-push
  script:
    - helm package expense-tracker
    - 'helm repo add --username gitlab-ci-token --password ${CI_JOB_TOKEN} ${CI_PROJECT_NAME} ${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/packages/helm/stable'
    - 'helm cm-push expense-tracker-*.tgz ${CI_PROJECT_NAME}'
  only:
    changes:
      - expense-tracker/**
```

In this stage, I use a Helm image (`dtzar/helm-kubectl`) to help package and deploy a Helm chart to the GitLab repository. The process follows GitLab’s documentation, and I’m using the `helm plugin cm-push` to push the packaged Helm chart. GitLab offers two options for this process: using a `curl` command or using the Helm plugin. 

If you want to learn more about this, I encourage you to read the [GitLab charts documentation](https://docs.gitlab.com/charts/).

I will explore Helm in more detail in a future article.

### Important Note: The `only:` Keyword

You might be wondering what the `only:` keyword means in the YAML configuration. This special keyword helps control when each job is executed. It ensures that a job runs only if there are changes in specific files or directories. For example:

- `only: changes: - '**/*.go'`: This means that the job will only run if any Go file (`*.go`) is modified in the repository.
- `only: changes: - 'expense-tracker/**'`: This means the job will only run if something inside the `expense-tracker` directory is changed. This is where the Helm chart is defined, so this ensures the job only runs when there are changes related to the Helm chart.

### Final thoughts

I hope you found this blog helpful and enjoyed learning about setting up a CI/CD pipeline with GitLab! This post is just the beginning of a series where I’ll delve deeper into CI/CD concepts, Docker, Helm, and more advanced topics in DevOps. If you have any questions, feedback, or observations, feel free to reach out—I’d love to hear from you!

This is my first deep dive into GitLab CI/CD, and while there’s always room for improvement, I’m excited to continue exploring and refining my DevOps practices. Stay tuned for more content, and thank you for reading!