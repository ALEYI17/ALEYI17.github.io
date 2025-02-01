---
title: "Part 4: Managing PostgreSQL with CNPG Operator"
date: 2025-02-01T16:16:25-05:00
draft: false
image: https://upload.wikimedia.org/wikipedia/commons/thumb/2/29/Postgresql_elephant.svg/200px-Postgresql_elephant.svg.png
author: "Alejandro Salamanca"
github_link: "https://gitlab.com/devopsdemo2374651/CurrencyConversion"
tags:
  - Kubernetes
  - Postgresql
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

### Part 4: Managing PostgreSQL with CNPG Operator

#### **Database Management in the Cloud Native Era**

In the final part of this blog series, we’re diving into one of the most crucial aspects of modern applications: **databases**. In today’s data-driven world, databases are the backbone of most businesses, making them one of the most critical components of an application’s architecture. Whether it's customer data, transactions, or user preferences, managing data efficiently and securely is paramount.

Cloud-native databases have become increasingly popular, with many cloud providers offering fully managed database solutions like AWS RDS and Google Cloud BigQuery. However, in recent years, Kubernetes has given rise to a new breed of self-managed databases, offering more flexibility and control for teams looking to run databases within their own Kubernetes clusters.

Before the advent of **Operators** and **Custom Resource Definitions (CRDs)**, managing stateful applications like databases in Kubernetes was a challenge. Initially, the idea of running databases on Kubernetes with **StatefulSets** was bold, yet inefficient, as StatefulSets alone didn't provide the necessary automation or management features required for production-grade databases. It was commonly believed that Kubernetes was best suited for stateless workloads. 

However, everything changed with the introduction of **Operators**, which brought powerful automation capabilities to Kubernetes, making it possible to manage stateful applications like databases in a more streamlined and reliable way. In this part of the series, we'll explore **Cloud Native PostgreSQL (CNPG)**, one of the most robust and popular Kubernetes database solutions available today, and how it can be used to run PostgreSQL databases seamlessly in Kubernetes.

### Understanding Operators

In the Kubernetes ecosystem, **Operators** are a software extension that allows you to manage the lifecycle of an application or service using **Custom Resources** (CRs). The primary role of an operator is to automate tasks that would traditionally be carried out by a human operator, such as a **Database Administrator (DBA)**. The goal is to let Kubernetes handle routine, repetitive tasks, providing high availability, scaling, and failover management automatically.

The key feature of the operator pattern is its ability to act as a "real" operator would. Just like a human DBA, the operator monitors the state of the application, understands its requirements, and takes action when things go wrong. For example, if a database pod fails or needs scaling, the operator automatically responds by provisioning new resources, restoring backups, or reconfiguring the service.

Operators let you extend the functionality of a Kubernetes cluster without needing to modify Kubernetes' core code. This is achieved by building controllers, which are clients interacting with the Kubernetes API to manage the lifecycle of custom resources (CRs). **Custom Resources** are essentially a way to extend the Kubernetes API to represent the state of applications that Kubernetes doesn’t natively support.

Before operators, Kubernetes applications that required persistent storage were often managed with **StatefulSets**, which offer basic storage management for stateful applications. However, StatefulSets alone are not sufficient for fully managing complex services like databases. While StatefulSets provide stable storage and network identities to pods, they don’t address critical tasks such as **backups**, **high availability**, or **failover** handling.

Without operators, if a pod failed or the application required updates, these tasks would need to be done manually, creating room for errors and inefficiencies. Operators automate these processes, offering solutions like automatic backups, monitoring, scaling, and application recovery—functions that are essential for running production-grade databases like **PostgreSQL** in a Kubernetes environment.

### Installing the CNPG Operator

The recommended way to install the **CNPG Operator** in your Kubernetes cluster is by using **Helm**, which simplifies the process of managing Kubernetes applications. To get started, follow these steps:

1. **Add the CNPG Helm repository** to your Helm client by running the following command:
   
   ```bash
   helm repo add cnpg https://cloudnative-pg.github.io/charts
   ```

   This command adds the CNPG Helm repository to your list of available Helm charts.

2. **Install the CNPG Operator** by executing the following command:

   ```bash
   helm upgrade --install cnpg \
     --namespace cnpg-system \
     --create-namespace \
     cnpg/cloudnative-pg
   ```

   This command does two things:
   - Installs the CNPG Operator into the **cnpg-system** namespace.
   - Creates the **cnpg-system** namespace if it doesn’t already exist.

   After running this command, the CNPG Operator will be installed and ready to manage PostgreSQL clusters in your Kubernetes environment.

For more information about the Helm chart and additional configuration options, you can visit the [official CNPG GitHub repository](https://github.com/cloudnative-pg/charts).

### Deploying PostgreSQL with CNPG: Setting Up Your Database

In this section, we’ll walk through how I deploy a **PostgreSQL** cluster using the **CNPG Operator** for my demo application. All the necessary code for this setup can be found in the Helm chart repository for the application. As I mentioned earlier in **Part 3: Deploying with Helm in Kubernetes**, the database-related files were not presented before, but now it’s time to showcase them.

#### **Database Configuration: init-sql-configmap.yaml**

The first step in setting up the PostgreSQL database is defining the configuration. In the **init-sql-configmap.yaml** file, we set up the SQL schema that will initialize the database when the PostgreSQL cluster is created. Here’s the content of the **init-sql-configmap.yaml** file:

```yaml
{{- if .Values.cnpg.initSQL.configMap.enabled }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Values.cnpg.initSQL.configMap.name }}
data:
  users-expenses.sql: |
    {{ .Values.cnpg.initSQL.configMap.sql | nindent 4 }}
{{- end }}
```

This is a simple **ConfigMap** that holds the SQL schema in the **users-expenses.sql** key. This schema file will define the structure of the database tables. Below is the relevant section in the **values.yaml** file that configures the cluster settings:

```yaml
cnpg:
  clusterName: "cluster-expense-tracker"
  databaseName: "app"
  databaseOwner: "app"
  storageSize: "1Gi"
  replicas: 3
  initSQL:
    configMap:
      enabled: true
      name: "post-init-configmap"
      sql: |
        CREATE TABLE users (
            id SERIAL PRIMARY KEY,
            username VARCHAR(100) NOT NULL,
            email VARCHAR(100) UNIQUE NOT NULL,
            password VARCHAR(255) NOT NULL,
            country VARCHAR(50) NOT NULL,
            phone VARCHAR(15),
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        );

        CREATE TABLE IF NOT EXISTS expenses (
            id SERIAL PRIMARY KEY,
            description VARCHAR(255) NOT NULL,
            amount NUMERIC(10, 2) NOT NULL,
            date DATE NOT NULL,
            user_id INT NOT NULL,
            FOREIGN KEY (user_id) REFERENCES users (id) ON DELETE CASCADE
        );
        GRANT SELECT, INSERT, DELETE, UPDATE ON users, expenses TO app;
        GRANT USAGE, SELECT ON SEQUENCE users_id_seq TO app;
        GRANT USAGE, SELECT ON SEQUENCE expenses_id_seq TO app;
```
#### **Why Grant Privileges?**

It’s essential to grant privileges to the app user because when the database is initialized, it runs in superuser mode. Therefore, the app needs specific access rights to the tables and sequences. In this case, we are using the default app user created automatically by the CNPG Operator. While it’s possible to create additional users, I’ve kept it simple by using the default user.

#### **Deploying the PostgreSQL Cluster: Applying the Manifest**

Once the database configuration is ready, the next step is to create the **PostgreSQL** cluster using the CNPG operator. This is done by applying a manifest that defines the cluster's settings, including the number of replicas, initialization details, and storage configurations.

Here's the manifest to create the PostgreSQL cluster:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: {{ .Values.cnpg.clusterName }}
spec:
  instances: {{ .Values.cnpg.replicas }}
  bootstrap:
    initdb:
      database: {{ .Values.cnpg.databaseName }}
      owner: {{ .Values.cnpg.databaseOwner }}
      postInitApplicationSQLRefs:
        configMapRefs:
          - name: {{ .Values.cnpg.initSQL.configMap.name }}
            key: users-expenses.sql
  storage:
    size: {{ .Values.cnpg.storageSize }}
```

This manifest defines the basic setup for your PostgreSQL cluster. Let's break it down:

**Cluster Name**: The `metadata.name` specifies the name of the cluster, which is dynamically referenced from the Helm chart's `values.yaml` file.

**Replicas**: In the `spec.instances`, we define the number of PostgreSQL replicas to be created for high availability. This can be easily scaled by modifying the `replicas` value in the `values.yaml` file.

**Bootstrap Section**: 
   - The `initdb` section specifies the database to be created, its owner, and how the database schema should be initialized.
   - The `postInitApplicationSQLRefs` section references the **ConfigMap** where the database schema SQL file is stored (the `users-expenses.sql` file). This ensures that the database is initialized with the schema you defined earlier.

**Storage**: The `storage.size` specifies the amount of storage to be allocated for the database. In this case, we use `1Gi`, but this can be adjusted based on your application needs.

This manifest provides a basic setup for the PostgreSQL cluster, but CNPG offers many additional features that can enhance its capabilities. One of the key features is database migration, where you can configure the cluster to copy data from an external database or another cluster. This can be particularly useful when migrating from an existing environment or consolidating data from different sources.

Another important feature is backups. The CNPG operator supports backup configurations, ensuring that your data is safely backed up. This feature is crucial for maintaining data integrity and availability, providing peace of mind in case of any system failures or issues.

Scaling is another feature CNPG excels at. The operator allows you to easily adjust the number of replicas in your PostgreSQL cluster, improving availability and reliability. This means you can scale your database infrastructure in response to traffic spikes or increased demand without much hassle.

CNPG also provides features for managing seamless upgrades of your PostgreSQL instances. This makes it easier to keep your database up to date without experiencing downtime or service disruptions, ensuring your system remains secure and performant.

For more advanced features and detailed configuration options, I highly recommend checking out the official CNPG documentation. It is well-maintained and offers comprehensive guides with examples for all the features mentioned above, helping you make the most out of CNPG for your PostgreSQL clusters.

#### Connecting the Application to the PostgreSQL Database
To connect the application with the PostgreSQL database, CNPG provides various resources that facilitate this integration. If you want to explore more details, you can refer to the official documentation at [CNPG Applications](https://cloudnative-pg.io/documentation/1.25/applications/).  

CNPG automatically creates both a service and a secret that allow the application to connect to the database cluster. By running the command:  

```sh
kubectl get services
```

We obtain the following output:  

```sh
NAME                               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
cluster-expense-tracker-pgadmin4   ClusterIP   10.43.116.36    <none>        80/TCP     4d
cluster-expense-tracker-r          ClusterIP   10.43.95.52     <none>        5432/TCP   3d23h
cluster-expense-tracker-ro         ClusterIP   10.43.23.191    <none>        5432/TCP   3d23h
cluster-expense-tracker-rw         ClusterIP   10.43.45.89     <none>        5432/TCP   3d23h
```

Among these services, the most relevant one for application connections is `cluster-expense-tracker-rw`, which is the **read-write** service that the application will use for database interactions.  

However, in order to connect, we also need the correct credentials, which are stored in Kubernetes secrets. Running:  

```sh
kubectl get secrets
```

Generates the following output:  

```sh
NAME                                    TYPE                       DATA   AGE
cluster-expense-tracker-app             kubernetes.io/basic-auth   9      3d23h
cluster-expense-tracker-ca              Opaque                     2      3d23h
cluster-expense-tracker-pgadmin4        Opaque                     2      4d
cluster-expense-tracker-replication     kubernetes.io/tls          2      3d23h
cluster-expense-tracker-server          kubernetes.io/tls          2      3d23h
```

The secret we need to use is `cluster-expense-tracker-app`, as it contains the authentication details required to access the database.  

This secret includes multiple keys, but the one the application requires is the `uri` key. If you recall from **Part 1**, where we defined the Go REST API with Gin, we established that the database URI should be provided as an environment variable. This is handled in the `deployment.yaml` manifest using the following snippet:  

```yaml
env:
  - name: DATABASE_URI
    valueFrom:
      secretKeyRef:
        name: {{.Values.cnpg.clusterName}}-app
        key: {{.Values.secretPlaceholder.secretKey}}
```

By referencing the secret in this way, the application automatically retrieves the correct database URI from Kubernetes, ensuring a secure and seamless connection to the PostgreSQL cluster.

### Final Thoughts

I hope you enjoyed this series of blogs. This is the final part, where we explored the world of databases in Kubernetes. I started this mini-project to deepen my understanding of these technologies, learn more about cloud-native tools, and improve my programming skills.  

These are also my first blog posts, as it is often said that one of the best ways to learn is by teaching and sharing knowledge. I hope you found these blogs useful and learned something from them. If you have any advice or constructive criticism, feel free to reach out to me via email or LinkedIn—I’d be happy to connect and discuss.  

I plan to continue creating more blogs related to similar technologies, so stay tuned!