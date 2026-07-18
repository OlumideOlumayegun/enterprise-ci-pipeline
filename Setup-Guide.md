# Enterprise Continuous Integration Pipeline Using Jenkins, GitHub, Maven, SonarQube, Nexus Repository, Slack, and AWS EC2

![project banner](./images/project-banner.png)

## 1. Project Overview

This project implements an automated Continuous Integration pipeline for a Java web application.

Whenever a developer pushes a code change to GitHub, the pipeline automatically:

1. Triggers Jenkins through a GitHub webhook.
2. Retrieves the latest source code.
3. Builds the application using Maven.
4. Downloads Maven dependencies through Nexus Repository Manager.
5. Runs automated unit tests.
6. performs Checkstyle static code analysis.
7. Scans the source code using SonarQube.
8. Enforces a SonarQube Quality Gate.
9. Packages the application as a WAR artifact.
10. Uploads the versioned artifact to Nexus.
11. Sends the pipeline result to a Slack channel.

The pipeline follows a Pipeline-as-Code approach, with the complete workflow defined in a version-controlled `Jenkinsfile`.

---

# 2. Architecture

![project banner](./images/architecture-diagram.png)

```text
Developer
    │
    │ Git push
    ▼
GitHub Repository
    │
    │ Webhook
    ▼
Jenkins Server
    │
    ├── Maven build
    ├── Unit tests
    ├── Checkstyle analysis
    ├── SonarQube scan
    ├── Quality Gate validation
    └── Artifact packaging
          │
          ├──────────────► Nexus Repository
          │                 - Dependency caching
          │                 - Artifact storage
          │
          ├──────────────► SonarQube
          │                 - Code analysis
          │                 - Quality Gate
          │
          └──────────────► Slack
                            - Success notification
                            - Failure notification
```

---

# 3. Technologies Used

| Technology               | Purpose                                                       |
| ------------------------ | ------------------------------------------------------------- |
| AWS EC2                  | Hosts Jenkins, Nexus and SonarQube                            |
| Jenkins                  | Orchestrates the CI pipeline                                  |
| GitHub                   | Hosts source code and triggers builds                         |
| Git                      | Source code version control                                   |
| Maven                    | Builds the Java application and manages dependencies          |
| Nexus Repository Manager | Caches dependencies and stores build artifacts                |
| SonarQube                | Performs code quality and security analysis                   |
| Checkstyle               | Performs Java static code analysis                            |
| Slack                    | Sends pipeline status notifications                           |
| PostgreSQL               | Stores SonarQube application data                             |
| Nginx                    | Provides reverse-proxy access to SonarQube                    |
| Java                     | Runtime and build dependency for the application and CI tools |

---

# 4. Prerequisites

Before starting, ensure that you have:

* An AWS account.
* A GitHub account.
* A Slack account.
* Git installed locally.
* Git Bash, Linux terminal or macOS Terminal.
* Visual Studio Code or another code editor.
* Basic knowledge of Linux, Git, Maven and AWS EC2.
* The Java application source code.
* The Jenkins, Nexus and SonarQube EC2 User Data scripts.

---

# 5. Create the AWS Key Pair

Open the AWS EC2 console and select your preferred AWS Region.

Navigate to:

```text
EC2 → Network & Security → Key Pairs
```

Create a key pair with the following settings:

| Setting  | Example           |
| -------- | ----------------- |
| Name     | `vprofile-ci-key` |
| Key type | RSA               |
| Format   | `.pem`            |

Use `.pem` when connecting through Git Bash, Linux or macOS. PuTTY users may use `.ppk`.

Store the downloaded private key securely.

---

# 6. Configure Security Groups

Create separate security groups for Jenkins, Nexus and SonarQube. Do not unnecessarily modify the default outbound rules. 

## 6.1 Jenkins Security Group

Create:

```text
Jenkins-SG
```

Add the following inbound rules:

| Port | Source        | Purpose                              |
| ---- | ------------- | ------------------------------------ |
| 22   | My IP         | SSH administration                   |
| 8080 | Anywhere IPv4 | Jenkins interface and GitHub webhook |
| 8080 | Anywhere IPv6 | Jenkins interface and GitHub webhook |

Port `8080` must be reachable from GitHub for webhook delivery.

For a production deployment, place Jenkins behind a reverse proxy, load balancer or secure ingress endpoint rather than exposing it directly.

## 6.2 Nexus Security Group

Create:

```text
Nexus-SG
```

Add:

| Port | Source     | Purpose                                           |
| ---- | ---------- | ------------------------------------------------- |
| 22   | My IP      | SSH administration                                |
| 8081 | My IP      | Nexus web interface                               |
| 8081 | Jenkins-SG | Jenkins dependency downloads and artifact uploads |

## 6.3 SonarQube Security Group

Create:

```text
Sonar-SG
```

Add:

| Port | Source              | Purpose                             |
| ---- | ------------------- | ----------------------------------- |
| 22   | My IP               | SSH administration                  |
| 80   | My IP               | SonarQube through Nginx             |
| 80   | Jenkins-SG          | Jenkins analysis uploads            |
| 9000 | Optional/restricted | Direct SonarQube access if required |

If Jenkins port `8080` is restricted, allow inbound access to Jenkins from `Sonar-SG` so SonarQube can return Quality Gate results.

## 6.4 Dynamic IP Consideration

If your internet provider changes your public IP, update all rules configured with **My IP** before reconnecting.

---

# 7. Launch the EC2 Instances

Provision three EC2 instances using the appropriate User Data scripts. 

## 7.1 Jenkins Server

Use:

| Setting          | Value                   |
| ---------------- | ----------------------- |
| Name             | Jenkins Server          |
| Operating system | Ubuntu Server 24.04 LTS |
| Instance type    | `t2.small` minimum      |
| Security group   | Jenkins-SG              |
| Storage          | 8 GB or more            |
| Key pair         | `vprofile-ci-key`       |

Under **Advanced Details → User Data**, paste the Jenkins installation script.

Ensure the script begins with:

```bash
#!/bin/bash
```

The script should:

* Update package repositories.
* Install OpenJDK 17.
* Add the Jenkins repository.
* Install Jenkins.
* Start and enable Jenkins.

## 7.2 Nexus Server

Use:

| Setting          | Value             |
| ---------------- | ----------------- |
| Name             | Nexus Server      |
| Operating system | Amazon Linux 2023 |
| Instance type    | `t2.medium`       |
| Security group   | Nexus-SG          |
| Key pair         | `vprofile-ci-key` |

Paste the Nexus User Data script.

The script should:

* Install Java 17.
* Download the Nexus binary.
* Extract and configure Nexus.
* Create a dedicated service user.
* Create a systemd service.
* Start and enable Nexus.

## 7.3 SonarQube Server

Use:

| Setting          | Value                   |
| ---------------- | ----------------------- |
| Name             | Sonar Server            |
| Operating system | Ubuntu Server 24.04 LTS |
| Instance type    | `t2.medium`             |
| Security group   | Sonar-SG                |
| Key pair         | `vprofile-ci-key`       |

Paste the SonarQube User Data script.

The script should:

* Increase Linux file and process limits.
* Configure the required kernel settings.
* Install Java.
* Install and configure PostgreSQL.
* Create the SonarQube database and user.
* Download and configure SonarQube.
* Create a systemd service.
* Install and configure Nginx.
* Reboot the server to apply kernel changes.

Allow sufficient time for all installations to complete.

---

# 8. Complete Jenkins Post-Installation

Connect to the Jenkins server:

```bash
ssh -i /path/to/vprofile-ci-key.pem ubuntu@<jenkins-public-ip>
```

Check the service:

```bash
sudo systemctl status jenkins
```

Check Java:

```bash
java -version
```

Jenkins stores its main application data under:

```text
/var/lib/jenkins
```

Open Jenkins:

```text
http://<jenkins-public-ip>:8080
```

Retrieve the initial password:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Enter the password in the browser and select:

```text
Install suggested plugins
```

Create the Jenkins administrator account and complete the initial setup. 

---

# 9. Install Jenkins Plugins

Navigate to:

```text
Manage Jenkins → Plugins → Available Plugins
```

Install:

* Maven Integration
* GitHub Integration
* Nexus Artifact Uploader
* SonarQube Scanner
* Slack Notification
* Build Timestamp

Restart Jenkins if requested.

---

# 10. Complete Nexus Post-Installation

Connect to Nexus:

```bash
ssh -i /path/to/vprofile-ci-key.pem ec2-user@<nexus-public-ip>
```

Verify the service:

```bash
sudo systemctl status nexus
```

Open Nexus:

```text
http://<nexus-public-ip>:8081
```

Retrieve the initial password using the path displayed on the Nexus login page.

Sign in with:

```text
Username: admin
Password: <initial-password>
```

Complete the setup wizard:

1. Set a new administrator password.
2. Disable anonymous access.
3. Finish the configuration.

---

# 11. Create Nexus Maven Repositories

Navigate to:

```text
Administration → Repositories → Create Repository
```

Create the following repositories.

## 11.1 Release Repository

| Setting        | Value              |
| -------------- | ------------------ |
| Type           | Maven hosted       |
| Name           | `vprofile-release` |
| Version policy | Release            |

This repository stores validated application artifacts.

## 11.2 Maven Central Proxy

| Setting        | Value                        |
| -------------- | ---------------------------- |
| Type           | Maven proxy                  |
| Name           | `vprofile-maven-central`     |
| Remote storage | Maven Central repository URL |

This repository downloads and caches external Maven dependencies.

## 11.3 Snapshot Repository

| Setting        | Value               |
| -------------- | ------------------- |
| Type           | Maven hosted        |
| Name           | `vprofile-snapshot` |
| Version policy | Snapshot            |

## 11.4 Group Repository

| Setting | Value                  |
| ------- | ---------------------- |
| Type    | Maven group            |
| Name    | `vprofile-maven-group` |

Add:

* `vprofile-release`
* `vprofile-snapshot`
* `vprofile-maven-central`

The group repository provides a single endpoint for Maven dependency resolution.

---

# 12. Verify SonarQube

Open:

```text
http://<sonarqube-public-ip>
```

If Nginx is not configured, use:

```text
http://<sonarqube-public-ip>:9000
```

Sign in using the initial credentials:

```text
Username: admin
Password: admin
```

Change the default password when prompted.

Verify from the command line if necessary:

```bash
sudo systemctl status sonar
```

---

# 13. Fork and Clone the GitHub Repository

Sign in to GitHub and open the source repository:

```text
https://github.com/hkhcoder/vprofile-project/
```

Select **Fork** and ensure that **Copy the main branch only** is disabled so the `jenkins-ci` branch is also copied. 

Your repository should resemble:

```text
<your-username>/vprofile-project
```

Configure Git locally:

```bash
git config --global user.name "Your Name"
git config --global user.email "your-email@example.com"
```

---

# 14. Configure GitHub SSH Authentication

Create an SSH key:

```bash
cd ~/.ssh
ssh-keygen
```

Give the key a descriptive name, for example:

```text
github-vprofile
```

Display the public key:

```bash
cat ~/.ssh/github-vprofile.pub
```

Copy it and navigate to:

```text
GitHub → Settings → SSH and GPG keys → New SSH key
```

Paste the public key.

Create or edit:

```text
~/.ssh/config
```

Add:

```text
Host github-vprofile
    HostName github.com
    User git
    IdentityFile ~/.ssh/github-vprofile
```

Clone your fork using the alias:

```bash
git clone git@github-vprofile:<your-username>/vprofile-project.git
cd vprofile-project
git checkout jenkins-ci
```

Test authentication by creating and pushing a small commit.

---

# 15. Configure Jenkins Build Tools

Navigate to:

```text
Manage Jenkins → Tools
```

## 15.1 Configure JDK

Connect to Jenkins and identify the Java path:

```bash
ls -l /usr/lib/jvm
```

Add a JDK installation:

| Setting   | Example                              |
| --------- | ------------------------------------ |
| Name      | `JDK17`                              |
| JAVA_HOME | `/usr/lib/jvm/java-17-openjdk-amd64` |

The configured name must exactly match the name used in the Jenkinsfile.

## 15.2 Configure Maven

Add Maven:

| Setting | Value      |
| ------- | ---------- |
| Name    | `Maven3.9` |
| Version | 3.9.9      |

Jenkins may manage several JDK and Maven versions for different projects. 

---

# 16. Store Nexus Credentials in Jenkins

Navigate to:

```text
Manage Jenkins → Credentials → System → Global credentials
```

Add:

| Setting     | Value                  |
| ----------- | ---------------------- |
| Kind        | Username with password |
| Username    | Nexus username         |
| Password    | Nexus password         |
| ID          | `Nexus-login`          |
| Description | Nexus Repository Login |

The credential ID will be referenced in the Jenkinsfile.

---

# 17. Create the Jenkins Pipeline Job

From the Jenkins dashboard, select:

```text
New Item
```

Configure:

| Setting | Value                  |
| ------- | ---------------------- |
| Name    | `vprofile-ci-pipeline` |
| Type    | Pipeline               |

Under the Pipeline section:

| Setting        | Value                    |
| -------------- | ------------------------ |
| Definition     | Pipeline script from SCM |
| SCM            | Git                      |
| Repository URL | SSH URL of your fork     |
| Branch         | `*/jenkins-ci`           |
| Script path    | `Jenkinsfile`            |

---

# 18. Add GitHub Credentials to Jenkins

Under the pipeline repository configuration, add a credential:

| Setting     | Value                                  |
| ----------- | -------------------------------------- |
| Kind        | SSH Username with Private Key          |
| Username    | `git`                                  |
| ID          | `git-login`                            |
| Private key | Contents of the private GitHub SSH key |

Do not use the `.pub` file. Jenkins requires the private key.

Select this credential for the pipeline.

---

# 19. Configure the Maven Settings File

The project should contain a custom Maven `settings.xml`.

Configure the Nexus group repository as a Maven mirror.

A simplified example is:

```xml
<settings>
    <mirrors>
        <mirror>
            <id>nexus</id>
            <name>Nexus Maven Group</name>
            <url>http://${NEXUS_IP}:${NEXUS_PORT}/repository/${NEXUS_GROUP_REPO}/</url>
            <mirrorOf>*</mirrorOf>
        </mirror>
    </mirrors>

    <servers>
        <server>
            <id>nexus</id>
            <username>${NEXUS_USERNAME}</username>
            <password>${NEXUS_PASSWORD}</password>
        </server>
    </servers>
</settings>
```

The exact server IDs in `settings.xml` must match those expected by the project configuration.

---

# 20. Create the Initial Jenkinsfile

Create a file named:

```text
Jenkinsfile
```

Start with:

```groovy
pipeline {
    agent any

    tools {
        jdk 'JDK17'
        maven 'Maven3.9'
    }

    environment {
        NEXUS_IP = '<nexus-private-ip>'
        NEXUS_PORT = '8081'
        NEXUS_GROUP_REPO = 'vprofile-maven-group'
        NEXUS_RELEASE_REPO = 'vprofile-release'
        NEXUS_SNAPSHOT_REPO = 'vprofile-snapshot'
        NEXUS_CREDENTIAL_ID = 'Nexus-login'
    }

    stages {
        stage('Build') {
            steps {
                sh 'mvn clean install -s settings.xml -DskipTests'
            }
        }
    }
}
```

Use the Nexus EC2 private IP because Jenkins and Nexus communicate within the same AWS network. 

Commit and push:

```bash
git add Jenkinsfile settings.xml
git commit -m "Add initial Jenkins build pipeline"
git push origin jenkins-ci
```

Run the Jenkins pipeline manually.

---

# 21. Resolve GitHub Host Key Verification

The first build may fail with:

```text
Host key verification failed
```

This means the Jenkins user has not accepted GitHub's SSH host fingerprint.

Connect to Jenkins:

```bash
ssh -i /path/to/key.pem ubuntu@<jenkins-public-ip>
```

Switch to the Jenkins user:

```bash
sudo su - jenkins
```

Run:

```bash
ssh -T git@github.com
```

Accept the fingerprint when prompted.

GitHub's fingerprint will be stored in:

```text
~/.ssh/known_hosts
```

Run the pipeline again.

---

# 22. Configure the GitHub Webhook

Open your GitHub repository and navigate to:

```text
Settings → Webhooks → Add webhook
```

Configure:

| Setting      | Value                                             |
| ------------ | ------------------------------------------------- |
| Payload URL  | `http://<jenkins-public-ip>:8080/github-webhook/` |
| Content type | `application/json`                                |
| Events       | Push events                                       |

The trailing slash in `/github-webhook/` is important.

In Jenkins, open the pipeline configuration and enable:

```text
GitHub hook trigger for GITScm polling
```

Save the configuration. 

Whenever possible, assign Jenkins a stable DNS name or Elastic IP. Otherwise, update the webhook when the EC2 public IP changes.

---

# 23. Add Unit Test and Checkstyle Stages

Extend the Jenkinsfile:

```groovy
stage('Unit Test') {
    steps {
        sh 'mvn test -s settings.xml'
    }
}

stage('Checkstyle Analysis') {
    steps {
        sh 'mvn checkstyle:checkstyle -s settings.xml'
    }
}
```

Ensure every Maven command includes:

```text
-s settings.xml
```

Otherwise Maven may bypass Nexus and download dependencies directly from the internet.

---

# 24. Archive the WAR Artifact

Add a pipeline-level `post` block:

```groovy
post {
    success {
        archiveArtifacts artifacts: '**/*.war', fingerprint: true
    }
}
```

The generated application artifact should be located under:

```text
target/
```

For example:

```text
target/vprofile-v2.war
```

---

# 25. Configure the SonarQube Scanner in Jenkins

Navigate to:

```text
Manage Jenkins → Tools
```

Add:

| Setting | Value                          |
| ------- | ------------------------------ |
| Tool    | SonarQube Scanner              |
| Name    | `SonarScanner`                 |
| Version | Appropriate compatible version |

Next, generate a SonarQube token:

```text
SonarQube → My Account → Security → Generate Token
```

Copy the token.

In Jenkins, add a credential:

| Setting | Value           |
| ------- | --------------- |
| Kind    | Secret text     |
| Secret  | SonarQube token |
| ID      | `SonarToken`    |

Navigate to:

```text
Manage Jenkins → System → SonarQube Servers
```

Configure:

| Setting              | Value                           |
| -------------------- | ------------------------------- |
| Name                 | `SonarServer`                   |
| Server URL           | `http://<sonarqube-private-ip>` |
| Authentication token | `SonarToken`                    |

Because Nginx listens on port 80, the URL does not require port `9000`. 

---

# 26. Add the SonarQube Analysis Stage

Add environment values:

```groovy
environment {
    SONAR_SERVER = 'SonarServer'
    SONAR_SCANNER = 'SonarScanner'
}
```

Add the analysis stage:

```groovy
stage('SonarQube Analysis') {
    environment {
        SCANNER_HOME = tool "${SONAR_SCANNER}"
    }

    steps {
        withSonarQubeEnv("${SONAR_SERVER}") {
            sh '''
                ${SCANNER_HOME}/bin/sonar-scanner \
                -Dsonar.projectKey=vprofile \
                -Dsonar.projectName=vprofile \
                -Dsonar.projectVersion=1.0 \
                -Dsonar.sources=src/ \
                -Dsonar.java.binaries=target/classes \
                -Dsonar.junit.reportPaths=target/surefire-reports \
                -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml
            '''
        }
    }
}
```

Adjust report paths to match the files generated by your Maven project.

Commit and push the changes. The webhook should trigger the pipeline automatically.

Verify that the project appears in the SonarQube dashboard.

---

# 27. Create a SonarQube Quality Gate

In SonarQube, navigate to:

```text
Quality Gates → Create
```

Create:

```text
VProfile-QG
```

For testing, add a condition such as:

| Metric | Operator     | Threshold |
| ------ | ------------ | --------- |
| Bugs   | Greater than | 25        |

Assign the Quality Gate to the VProfile project:

```text
Project Settings → Quality Gate
```

---

# 28. Configure the SonarQube Webhook

Navigate to:

```text
Project Settings → Webhooks
```

Create:

| Setting | Value                                                 |
| ------- | ----------------------------------------------------- |
| Name    | Jenkins Webhook                                       |
| URL     | `http://<jenkins-private-ip>:8080/sonarqube-webhook/` |

The private IP is appropriate because Jenkins and SonarQube are in the same VPC. 

---

# 29. Add the Quality Gate Stage

Add:

```groovy
stage('Quality Gate') {
    steps {
        timeout(time: 1, unit: 'HOURS') {
            waitForQualityGate abortPipeline: true
        }
    }
}
```

When the Quality Gate fails, Jenkins should report:

```text
Pipeline aborted due to Quality Gate failure
```

Confirm that the failure is caused by the Quality Gate rather than invalid pipeline syntax.

Raise the threshold or correct the source code, then rerun the pipeline and verify that it passes.

---

# 30. Configure Build Timestamp

Navigate to:

```text
Manage Jenkins → System → Build Timestamp
```

Use a pattern such as:

```text
yy-MM-dd_HH-mm
```

The timestamp will be combined with the Jenkins build number to create unique artifact versions.

---

# 31. Upload the Artifact to Nexus

Add a stage after the Quality Gate:

```groovy
stage('Upload Artifact') {
    steps {
        nexusArtifactUploader(
            nexusVersion: 'nexus3',
            protocol: 'http',
            nexusUrl: "${NEXUS_IP}:${NEXUS_PORT}",
            groupId: 'com.vprofile',
            version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}",
            repository: "${NEXUS_RELEASE_REPO}",
            credentialsId: "${NEXUS_CREDENTIAL_ID}",
            artifacts: [
                [
                    artifactId: 'vproapp',
                    classifier: '',
                    file: 'target/vprofile-v2.war',
                    type: 'war'
                ]
            ]
        )
    }
}
```

Ensure that:

* The Nexus URL is correct.
* The repository name exactly matches Nexus.
* The credential ID is correct.
* The WAR file path exists.
* Variables are correctly quoted.

Commit and push the Jenkinsfile. 

Verify the uploaded artifact under:

```text
Nexus → Browse → vprofile-release
```

Each execution should create a uniquely versioned artifact.

---

# 32. Configure Slack

Create a Slack workspace and a dedicated channel such as:

```text
#jenkins-ci
```

Install the Jenkins CI Slack application into the workspace and authorize it for the notification channel.

Copy the generated integration token.

---

# 33. Store the Slack Token in Jenkins

Navigate to:

```text
Manage Jenkins → Credentials
```

Add:

| Setting     | Value                   |
| ----------- | ----------------------- |
| Kind        | Secret text             |
| Secret      | Slack token             |
| ID          | `SlackToken`            |
| Description | Slack Integration Token |

Navigate to:

```text
Manage Jenkins → System → Slack
```

Configure:

| Setting         | Value                |
| --------------- | -------------------- |
| Workspace       | Slack workspace name |
| Credential      | `SlackToken`         |
| Default channel | `#jenkins-ci`        |

Select **Test Connection** and confirm that the test succeeds. 

---

# 34. Add Slack Notifications to the Jenkinsfile

Define a build result colour map before the pipeline block:

```groovy
def COLOR_MAP = [
    'SUCCESS': 'good',
    'FAILURE': 'danger',
    'UNSTABLE': 'warning',
    'ABORTED': '#808080'
]
```

Add a pipeline-level `post` section:

```groovy
post {
    always {
        script {
            def result = currentBuild.currentResult

            slackSend(
                channel: '#jenkins-ci',
                color: COLOR_MAP[result] ?: 'warning',
                message: """
${result}: Job '${env.JOB_NAME}' Build #${env.BUILD_NUMBER}
More information: ${env.BUILD_URL}
"""
            )
        }
    }
}
```

This sends:

* A green message for successful builds.
* A red message for failed builds.
* The job name.
* The build number.
* A direct link to the Jenkins build.

---

# 35. Complete Jenkinsfile Example

```groovy
def COLOR_MAP = [
    'SUCCESS': 'good',
    'FAILURE': 'danger',
    'UNSTABLE': 'warning',
    'ABORTED': '#808080'
]

pipeline {
    agent any

    tools {
        jdk 'JDK17'
        maven 'Maven3.9'
    }

    environment {
        NEXUS_IP = '<nexus-private-ip>'
        NEXUS_PORT = '8081'
        NEXUS_GROUP_REPO = 'vprofile-maven-group'
        NEXUS_RELEASE_REPO = 'vprofile-release'
        NEXUS_SNAPSHOT_REPO = 'vprofile-snapshot'
        NEXUS_CREDENTIAL_ID = 'Nexus-login'

        SONAR_SERVER = 'SonarServer'
        SONAR_SCANNER = 'SonarScanner'
    }

    stages {
        stage('Build') {
            steps {
                sh 'mvn clean install -s settings.xml -DskipTests'
            }
        }

        stage('Unit Test') {
            steps {
                sh 'mvn test -s settings.xml'
            }
        }

        stage('Checkstyle Analysis') {
            steps {
                sh 'mvn checkstyle:checkstyle -s settings.xml'
            }
        }

        stage('SonarQube Analysis') {
            environment {
                SCANNER_HOME = tool "${SONAR_SCANNER}"
            }

            steps {
                withSonarQubeEnv("${SONAR_SERVER}") {
                    sh '''
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=vprofile \
                        -Dsonar.projectName=vprofile \
                        -Dsonar.projectVersion=1.0 \
                        -Dsonar.sources=src/ \
                        -Dsonar.java.binaries=target/classes \
                        -Dsonar.junit.reportPaths=target/surefire-reports \
                        -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Upload Artifact') {
            steps {
                nexusArtifactUploader(
                    nexusVersion: 'nexus3',
                    protocol: 'http',
                    nexusUrl: "${NEXUS_IP}:${NEXUS_PORT}",
                    groupId: 'com.vprofile',
                    version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}",
                    repository: "${NEXUS_RELEASE_REPO}",
                    credentialsId: "${NEXUS_CREDENTIAL_ID}",
                    artifacts: [
                        [
                            artifactId: 'vproapp',
                            classifier: '',
                            file: 'target/vprofile-v2.war',
                            type: 'war'
                        ]
                    ]
                )
            }
        }
    }

    post {
        success {
            archiveArtifacts artifacts: '**/*.war', fingerprint: true
        }

        always {
            script {
                def result = currentBuild.currentResult

                slackSend(
                    channel: '#jenkins-ci',
                    color: COLOR_MAP[currentBuild.currentResult],
                    message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URL}"
                )
            }
        }
    }
}
```

Modify file paths, repository names, credentials and tool identifiers to match your environment.

---

# 36. Validate the Complete Pipeline

Push a source code change to the `jenkins-ci` branch.

Confirm that:

1. GitHub successfully sends the webhook.
2. Jenkins starts automatically.
3. Jenkins retrieves the source code.
4. Maven downloads dependencies through Nexus.
5. The application builds successfully.
6. Unit tests run.
7. Checkstyle produces a report.
8. SonarQube analysis completes.
9. The Quality Gate result is returned to Jenkins.
10. The WAR file is archived.
11. The artifact is uploaded to Nexus.
12. Slack receives the final build notification.

---

# 37. Test Failure Handling

Introduce a controlled pipeline failure, such as an invalid Maven command.

Commit and push the change.

Verify that:

* Jenkins reports the failure.
* Later stages do not continue.
* Slack sends a red failure notification.
* The notification contains a link to the Jenkins build logs.

Correct the error and confirm that the subsequent build succeeds.

---

# 38. Common Troubleshooting Issues

## Jenkins cannot access GitHub

Check:

* Repository SSH URL.
* Jenkins Git credentials.
* GitHub public key registration.
* Jenkins user's `known_hosts` file.
* Selected Git branch.

## GitHub webhook fails

Check:

* Jenkins public IP.
* Port `8080` security group rule.
* `/github-webhook/` path.
* Jenkins availability.
* GitHub recent webhook delivery logs.

## Maven bypasses Nexus

Ensure every Maven command includes:

```text
-s settings.xml
```

Also verify the Nexus mirror URL and repository names.

## Quality Gate waits indefinitely

Check:

* SonarQube webhook URL.
* Jenkins private IP.
* Jenkins port `8080` access from SonarQube.
* SonarQube server configuration in Jenkins.
* SonarQube token.

## Nexus upload fails

Check:

* Nexus credential ID.
* Release repository name.
* Nexus URL and port.
* WAR file path.
* Quotation of Jenkins environment variables.

## Slack connection fails

Check:

* Workspace name.
* Channel name.
* Slack token.
* Jenkins CI Slack app installation.
* Credential selection in Jenkins.

---

# 39. Security Improvements for Production

The project is suitable as a hands-on CI implementation, but a production deployment should include:

* HTTPS for Jenkins, Nexus and SonarQube.
* Strong administrator passwords.
* Least-privilege service accounts.
* Restricted security group access.
* Secrets stored only in Jenkins Credentials.
* An Application Load Balancer or reverse proxy.
* DNS records rather than temporary public IP addresses.
* Private subnets for Nexus and SonarQube.
* Regular EBS snapshots and application backups.
* Centralised logging and monitoring.
* Automated infrastructure provisioning using Terraform or CloudFormation.
* Dedicated Jenkins agents rather than running all builds on the controller.
* IAM roles instead of long-lived AWS access keys.

---

# 40. Cost Management and Cleanup

When the project is not in use:

1. Stop the Jenkins, Nexus and SonarQube instances.
2. Confirm that no unnecessary Elastic IPs, load balancers or snapshots remain.
3. Retain Jenkins if it will be reused in subsequent projects.
4. Terminate unused instances only after backing up required configuration.
5. Delete unused EBS volumes when they are no longer needed.

Stopping an instance prevents compute usage, but storage and some network resources may continue to incur charges.

---

# 41. Project Outcome

This project delivers a complete enterprise-style Continuous Integration pipeline that:

* Uses Pipeline as Code.
* Responds automatically to GitHub changes.
* Builds and tests a Java application.
* Controls dependencies through Nexus.
* Performs static and SonarQube code analysis.
* Enforces an automated Quality Gate.
* Produces immutable, versioned artifacts.
* Publishes validated artifacts to Nexus.
* Provides real-time build feedback through Slack.

The resulting pipeline establishes a strong foundation for Continuous Delivery, where the validated Nexus artifact can be deployed automatically to development, testing, staging and production environments.

