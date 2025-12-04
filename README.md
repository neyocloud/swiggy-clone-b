
## CI/CD Pipeline with GitHub Actions (Terraform, SonarQube, Trivy, Slack, EKS)


Introduction



<img width="1454" height="802" alt="image" src="https://github.com/user-attachments/assets/e6a518f0-42f7-4eb0-b91b-89b76dc05613" />






This project demonstrates a full CI/CD DevOps pipeline that builds, analyzes, secures, and deploys a containerized application using GitHub Actions. The pipeline is designed to showcase DevOps best practices end-to-end, from infrastructure provisioning to continuous deployment on Kubernetes. Key components include:

Infrastructure as Code (Terraform): Automating creation of an AWS EC2 instance to host a self-hosted GitHub Actions runner, Docker engine, and a SonarQube server.

Code Quality Analysis (SonarQube): Integrating SonarQube for static code analysis to improve code quality (catch bugs, code smells, etc.).

Security Scanning (Trivy): Performing security scans on source code (files and dependencies) as well as container images to detect vulnerabilities, misconfigurations, and secrets early.

Containerization (Docker): Building a Docker image of the application and pushing it to a registry (e.g. Docker Hub).

Continuous Deployment to Kubernetes (EKS): Deploying the Docker image to an AWS EKS (Elastic Kubernetes Service) cluster using kubectl/eksctl, demonstrating Kubernetes deployment automation.

Notifications (Slack): Sending Slack notifications on pipeline results for real-time feedback to the team.



**The goal is to implement a robust pipeline that ensures code quality and security at each stage (using SonarQube and Trivy scans) and automates deployment to a production-like environment (AWS EKS). This end-to-end setup highlights strong DevOps skills in infrastructure provisioning, CI/CD automation, security (DevSecOps), and cloud orchestration, which would be clear and impressive to recruiters and interviewers.*

Creating the EC2 Infrastructure with Terraform (Self-Hosted Runner Host)
To have more control over the CI environment, I used a self-hosted GitHub Actions runner on AWS. I provisioned an EC2 instance using Terraform as my runner host (which also runs Docker and SonarQube). Terraform allowed me to define this infrastructure as code, ensuring repeatability and consistency. Key steps include:

Terraform Configuration: I wrote Terraform configs to define an EC2 instance (Ubuntu 20.04) with the required security group rules. For example, I opened port 22 for SSH, port 9000 for SonarQube’s web UI, and any needed ports for runner communication (GitHub runners use outbound HTTPS).

Applying Terraform: Using Termius (or any terminal), I ran Terraform commands to create the resources:


```
terraform init            # Initialize Terraform and AWS provider
terraform plan
terraform apply -auto-approve   # Create EC2 instance and related infrastructure

```

After apply, Terraform outputs the new EC2’s public IP.

I used that IP to SSH into the server.

Instance Setup: Once the EC2 was up, I SSH’d in and installed the necessary tools: Docker (to run SonarQube and to build images), and any dependencies required for the runner. For example:



<img width="1504" height="628" alt="image" src="https://github.com/user-attachments/assets/a8eac559-6cb8-4af1-ad2b-e8975c6f0968" />



I then launched a SonarQube container on this host and ensured it was accessible on port 9000:


```

docker run -d --name sonarqube -p 9000:9000 sonarqube:lts

```

For a production-grade setup, SonarQube would use an external database and more memory, but for this pipeline demo the above is sufficient.



Running CI on GitHub-Hosted Runner (ubuntu-latest)

With the EC2 server up and running Docker and SonarQube, the next step was to run our CI pipeline using a GitHub-hosted runner instead of a self-hosted one. This means the workflow executes on GitHub’s infrastructure (ubuntu-latest), and connects to the tools we set up on EC2 over the network.

Using a GitHub-Hosted Runner: In our GitHub Actions workflow, i didn’t add or register any self-hosted runner. Instead, I kept the default GitHub runner by specifying ubuntu-latest in the YAML. For example:

```
jobs:
build-scan:
runs-on: ubuntu-latest # GitHub-hosted runner
# ...
```


I also configured required credentials as encrypted secret keys in the repository (GitHub Secrets) to securely support the CI/CD pipeline.

<img width="3024" height="1964" alt="image" src="https://github.com/user-attachments/assets/453177be-9fb3-4ded-8aef-37a5c8c9d2ed" />



Workflow Execution: Once pushed, GitHub automatically provisioned an ubuntu-latest runner for every workflow run. The pipeline handled build, test, and analysis from that environment without needing any runner setup on the EC2 instance.

SonarQube Scan Connection: Since the runner was not on the EC2 server, it couldn’t reach SonarQube via localhost. So the Sonar scanner in the workflow connected to SonarQube using the EC2 server’s accessible URL/IP (rather than a local address). This allowed the analysis results to be sent to the SonarQube instance running on EC2.


<img width="3024" height="1964" alt="image" src="https://github.com/user-attachments/assets/744dcc74-043e-488d-a5bd-68386dd4aa05" />




Overall, this approach avoided the extra step of configuring a self-hosted runner and kept the CI setup simple, while still letting us run builds and push code quality reports to SonarQube.



## Security Scanning of Source Code with Trivy (FS Scan)

Security is integrated early in the pipeline using Trivy. This is an open-source security scanner by Aqua Security. We first perform a file system scan (Trivy FS) on the source repository to catch any known vulnerabilities or misconfigurations in code and dependencies before building the application. This is part of “shift-left” security in CI (DevSecOps)


What is Trivy FS Scans: Trivy’s filesystem scan can detect vulnerabilities in open-source packages, config issues in infrastructure-as-code (like Terraform or Kubernetes manifests), and even secret exposures in the codebase


By including this scan, we ensure that things like outdated libraries or insecure configurations are flagged early, preventing them from reaching production



Running Trivy in the Workflow: We integrated Trivy via a GitHub Action. after code checkout, we added:



```
- name: Trivy Security Scan (FS)
  uses: aquasecurity/trivy-action@v0.30.0
  with:
    scan-type: fs            # file system scan on repository
    scan-ref: '.'            # scan current repository directory
    severity: CRITICAL,HIGH,MEDIUM,LOW
    scanners: vuln,secret,misconfig
    exit-code: 0             # don't fail the job on findings (optional)
    format: table
```


This action downloads Trivy and scans the code (all folders) for vulnerabilities, secrets, and misconfigurations


I allowed the job to continue even if issues are found (exit-code: 0), so that the pipeline can proceed (you might instead fail the build on high-severity issues in a stricter scenario). The results are logged in the workflow. For instance, Trivy would list any CVEs in our application dependencies, any AWS credential leaks or hardcoded secrets, and any insecure configurations (like a Terraform file with an open security group). This automated scan helps “identify vulnerabilities before they make it to production”



Trivy FS Example: If running manually in a terminal, the equivalent would be:
```
trivy fs --scanners vuln,secret,config --severity CRITICAL,HIGH,MEDIUM .

```

The output would list findings for each category. Integrating this into CI ensures every code change is scrutinized for security issues as part of the build process.



## Docker Image Build and Push (CI Pipeline)

After code is analyzed and cleared, the pipeline builds a Docker image for the application and pushes it to a container registry. Containerizing the app ensures consistency between environments and is required for deployment to EKS.




Building the Docker Image: In the GitHub Actions workflow, I used Docker to build the image from our Dockerfile. Since we have Docker installed, we can either run Docker commands directly or use the Docker Buildx GitHub Action. We opted to run the commands directly for transparency. For example, our workflow has a step:


```
- name: Build Docker Image
  run: docker build -t myapp:latest 
```

This instructs Docker to build an image tagged myapp:latest from the repository’s Dockerfile. (If needed, I could also build multiple tags, e.g., commit SHA tag for versioning.)

Authenticating to Registry: I chose Docker Hub as the container registry (for simplicity and because our EKS cluster can pull public images from Docker Hub easily). We created a Docker Hub repository and added our Docker Hub credentials (username and an access token) as GitHub secrets. These secrets (DOCKER_USERNAME and DOCKERHUB_TOKEN) allow the workflow to log in to Docker Hub
docs.docker.com


 The login step in the workflow uses the Docker CLI or a GitHub Action:


```
- name: Login to Docker Hub
  run: echo "${{ secrets.DOCKERHUB_TOKEN }}" | docker login -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin
```

This logs in the Docker CLI to Docker Hub using my saved credentials (without exposing them in logs).

Pushing the Image: Once logged in, I pushed the image to the Docker Hub repository. I tagged the image with the repo name, then push:


<img width="3024" height="1964" alt="image" src="https://github.com/user-attachments/assets/9bcc35c8-3d39-4940-b4a0-0946e04653fb" />



```
- name: Push Image to Registry
  run: |
    docker tag myapp:latest ${{ secrets.DOCKER_USERNAME }}/myapp:latest
    docker push ${{ secrets.DOCKER_USERNAME }}/myapp:latest
```


This publishes the Docker image to Docker Hub (e.g., dockerhubuser/myapp:latest). In my case, the workflow waited briefly for Docker Hub to process the new image tag (ensuring it was available for the next step).

By the end of this stage, my CI pipeline had produced a versioned Docker image of the application and stored it in a registry, ready for deployment. Using GitHub Actions to automate the build and push ensured that every commit to the repo resulted in a fresh container image, eliminating “it works on my machine” issues.

Container Image Vulnerability Scanning with Trivy

After pushing the Docker image, my pipeline performed a Trivy scan on the image to detect vulnerabilities in the image’s OS packages or libraries. This added another layer of security by scanning the final artifact that would run in production.

**Why Image Scanning:** Even if my source dependencies were clean, vulnerabilities could still be introduced via the base image (e.g., OS packages). Trivy’s image scan checks for known CVEs in the image layers. This helps me catch critical issues in the container (for example, a critical CVE in the base Linux distribution) before deployment.

**Trivy Image Scan in Workflow:** I again used the Aqua Security action, but in image mode. The step in YAML looks like:


```
- name: Trivy Image Scan
  uses: aquasecurity/trivy-action@v0.30.0
  with:
    scan-type: image
    image-ref: '${{ secrets.DOCKER_USERNAME }}/myapp:latest'
    severity: CRITICAL,HIGH,MEDIUM
    format: table
    exit-code: 0   # don't fail pipeline on vulnerabilities (for demo)
```


This pulls the newly built image from Docker Hub and scans it for vulnerabilities


The scan results (a table of any found CVEs with severity) are printed in the job log. In a stricter pipeline, one could choose to fail the build if high/critical vulnerabilities are found, but here we report them while allowing deployment to proceed (the goal is to demonstrate the scan itself).



**Sample Trivy Image Output:** Your Trivy might output something like:



myapp:latest (alpine 3.18)
============================================
Total: 2 (HIGH: 1, CRITICAL: 1)

CRITICAL CVE-2023-XXXX – openssl – Fixed in 1.1.1t – Vulnerability details...
HIGH     CVE-2022-YYYY – libcurl – Fixed in 7.85.0 – Vulnerability details...




This tells me what to patch or update. By having this step, I demonstrate a proactive stance on container security: any issues in the image are immediately known and can be addressed before or after deployment as needed.


Slack Notifications for CI/CD Feedback


To stay informed of my pipeline results, I integrated Slack notifications into my GitHub Actions workflow. This means that whenever a build or deployment succeeds or fails, a message is sent to my specified Slack channel.


<img width="3024" height="1964" alt="image" src="https://github.com/user-attachments/assets/b27eb00c-1650-4cc0-9269-d9c114fa9bd9" />




Why Slack Integration: Teams often “live” in Slack, so sending CI/CD alerts there ensures fast visibility into the pipeline status


Immediate failure alerts can significantly reduce time to response and fix, improving DevOps feedback loops. It encourages collaboration – everyone knows when a deployment happens or if something breaks, without having to check the GitHub UI constantly.


<img width="3024" height="1964" alt="image" src="https://github.com/user-attachments/assets/1e794495-8730-425d-9960-fb0fcf58b1b8" />



Slack App Setup: I created a Slack app incoming webhook for our workspace. The simplest method is using an Incoming Webhook URL (which posts to a channel). Alternatively, one can use Slack’s Bot token and the Slack API. 



In my case, I used the official Slack GitHub Action 

Workflow Step for Slack: At the end of the workflow (in the deploy job), we added a step that runs regardless of success or failure:


```
- name: Notify Slack
  if: always()
  uses: slackapi/slack-github-action@v1.24.0
  with:
    channel-id: ${{ secrets.SLACK_CHANNEL_ID }}
    slack-message: |
      Deployment *${{ job.status }}* for `${{ github.repository }}@${{ github.ref }}` 
      - Job: Deploy 
      - Image: ${{ secrets.DOCKER_USERNAME }}/myapp:latest
  env:
    SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
```



This uses Slack’s official action to post a message to the channel with details. We include the job status (success or failure) and some context like repo, branch, etc.



The if: always() ensures the Slack step runs even if earlier steps fail, so we get notified on both success and failure. For example, a successful run would post a green-check message, whereas a failure would alert with a red-x and logs link. Slack notifications mean no one misses a broken build – it's an essential part of a robust CI/CD pipeline.



### The first result 


<img width="3024" height="1964" alt="image" src="https://github.com/user-attachments/assets/f10e453e-e1db-476c-a9ed-39863fdaced3" />





### Creating an EKS Cluster with eksctl

For the deployment environment, I made use Amazon Elastic Kubernetes Service (EKS) to host our application in a Kubernetes cluster. I provisioned a new EKS cluster using eksctl, which is a handy CLI to create EKS clusters quickly.

eksctl Cluster Creation: We ran the following command from our terminal to create a cluster (the EC2 runner or any machine with AWS CLI/eksctl configured could run this, but we did it locally for setup):


```
eksctl create cluster --name=virtualtechbox-cluster --region=ap-south-1 --nodes=2 --node-type=t3.small
```

This command creates a managed EKS cluster named “virtualtechbox-cluster” in the specified region with two worker nodes. It sets up the control plane, node group, and default configuration. The process takes several minutes. After completion, eksctl automatically updates the local Kubeconfig (usually ~/.kube/config) with the new cluster’s access credentials




AWS Credentials: We ensured that the AWS credentials used for cluster creation have the necessary permissions (IAM role or user with EKS and EC2 permissions). In our pipeline, we also need AWS credentials to deploy to EKS. We stored an IAM user’s access key and secret (with appropriate EKS/Kubernetes permissions) in GitHub Secrets (AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY). The pipeline uses these for authenticating to AWS.

kubectl and Config: The cluster’s credentials from eksctl can be used in the GitHub Actions environment. We either exported the kubeconfig as an artifact or re-generated it in the workflow. A simpler approach in Actions is to use the AWS CLI to re-create kubeconfig on the runner, which we’ll do in the next section.

(Note: For a real production scenario, one might use infrastructure-as-code (Terraform or CloudFormation) for EKS as well, but eksctl is sufficient and faster for this demonstration.)




### Deploying to EKS via GitHub Actions

With the EKS cluster ready, the second job of our GitHub Actions workflow handles deploying the latest Docker image to the cluster. This job runs after the build/scan job, and only if the first part succeeds. It uses AWS credentials and kubectl to apply Kubernetes manifests.

Job Dependencies: Our workflow is configured so that the “Deploy” job depends on the “Build-Analyze-Scan” job. In YAML, we set needs: [build-scan] for the deploy job. This ensures we only deploy if the build, analysis, and scans were successful.

Setting up kubectl: First, the job installs or ensures kubectl is available. We used the azure/setup-kubectl@v2 action (a GitHub-maintained action) to install a specific kubectl version compatible with our EKS version

 
 For example:
```
- name: Install kubectl
  uses: azure/setup-kubectl@v2
  with:
    version: 'v1.27.0'   # match EKS cluster's Kubernetes version
```

AWS Credentials in Workflow: Next, I configured AWS credentials in the job environment. I used the AWS Actions module:


```
- name: Configure AWS Credentials
  uses: aws-actions/configure-aws-credentials@v2
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: us-east-1
```

This exports the AWS creds so that AWS CLI and kubectl (with AWS IAM Auth) can function in subsequent steps



Updating Kubeconfig: To allow kubectl to talk to my EKS cluster, I updated the Kubernetes config for the cluster. I did this by running:


```
- name: Configure kubeconfig for EKS
  run: aws eks update-kubeconfig --name virtualtechbox-cluster --region us-east-1
```

This command fetches the cluster’s endpoint and credentials and merges them into ~/.kube/config 

 
 Essentially, it makes sure kubectl commands know which cluster to operate on.

Deploying the Application: I have Kubernetes manifest files in my repository (e.g., deployment.yaml and service.yaml) describing how to run my container in EKS. The deployment manifest uses the Docker image I pushed earlier, and I apply these manifests:


```
- name: Deploy to EKS
  run: |
    kubectl apply -f k8s/deployment.yaml 
    kubectl apply -f k8s/service.yaml
```

This creates/updates the Deployment and Service in the cluster


Kubernetes then pulls the Docker image from Docker Hub and starts the pods on the EKS worker nodes.



Slack Notification: (As described above in Slack section) The final step in this job posts a Slack message about the deployment status.

GitHub Actions pipeline – “Build-Analyze-Scan” job (self-hosted runner) succeeded, including steps for code checkout, SonarQube analysis, Trivy scan, Docker build/push, and Trivy image scan.


<img width="3024" height="1964" alt="image" src="https://github.com/user-attachments/assets/65f2e5b5-bbd2-4b1e-9f8e-d7fc86a76552" />



GitHub Actions pipeline – “Deploy” job succeeded, showing AWS credentials setup, kubectl config, deployment to EKS, and Slack notification steps.


<img width="3024" height="1964" alt="image" src="https://github.com/user-attachments/assets/8be8ba6b-b365-4aac-9b85-803fb2a962f7" />




In the screenshots above, you can see my two workflow jobs in GitHub Actions. The first job runs on ubuntu-latest and performs the build, analysis, and scanning steps. The second job then deploys to AWS EKS and sends a Slack notification. Both jobs succeeded, which confirms that my pipeline worked end-to-end.

Pipeline Validation and Verification

After the pipeline ran, I performed a few checks to validate that everything worked as expected:

**GitHub Actions Logs:** I reviewed the workflow run in GitHub Actions (as shown in the screenshots) to ensure every step completed successfully. The logs confirmed that my SonarQube analysis was successful (the scanner reported results to my SonarQube server), my Trivy scans completed (with findings printed where applicable), the Docker image built and pushed without errors, and the Kubernetes deployment was applied. Seeing green checkmarks across all steps and jobs showed me that the CI/CD pipeline executed correctly.

**SonarQube Dashboard:** I logged into SonarQube and checked my project dashboard. The latest analysis from the pipeline run was visible. I could see metrics like code smells, bugs, vulnerabilities, and other quality indicators, which gave me confidence in the stability and maintainability of my code. With SonarQube integrated, any new issue introduced by a commit would appear here immediately.

<img width="1226" height="732" alt="image" src="https://github.com/user-attachments/assets/472f4fab-01b6-438d-b798-10fec36ebf7a" />



Kubernetes Resources: i used kubectl to verify the application is running in the EKS cluster......


```
kubectl get pods
kubectl get svc
```

These commands showed an active Deployment (e.g., swiggy-app deployment with the desired number of pods running) and a Service exposing it.






### Live Demo (EKS LoadBalancer)

I deployed this application on AWS EKS and exposed it using a Kubernetes **LoadBalancer** Service.  



<img width="3024" height="1964" alt="image" src="https://github.com/user-attachments/assets/3d7e115c-63dd-4802-b1ac-b97b7b1f8e0d" />




This confirms the deployment part of the pipeline succeeded and the new Docker image is running on EKS.

**Slack Message:** I checked Slack and saw the notification from my pipeline. The message confirmed a successful deployment and included commit information. This real-time confirmation helped me instantly know the pipeline status. If anything had failed, Slack would have alerted me immediately with the failure status, and I could jump straight into the GitHub Actions logs using the link.

By cross-verifying in SonarQube, my EKS cluster, and Slack, I confirmed that every integration in my pipeline (code analysis, security scanning, deployment, and notifications) worked correctly. My pipeline didn’t just run without errors — it achieved the exact outcomes I wanted: strong confidence in code and image quality, plus automated deployment end-to-end.
### Cleanup and Teardown



After validating the pipeline, I cleaned up the resources to avoid ongoing costs and keep the environment tidy:

Kubernetes Deployments: I deleted the deployed application from the EKS cluster (especially since this was a demo deployment). Using kubectl:


```
kubectl delete deployment swiggy-app
kubectl delete service swiggy-app
```

(Replace swiggy-app with your deployment/service names.) This scales down and removes the app from the cluster.

EKS Cluster: Since the cluster was created for demonstration, I tore it down to avoid AWS charges. I used eksctl to delete the cluster:

```
eksctl delete cluster --name=virtualtechbox-cluster --region=us-east-1
```


<img width="2310" height="1274" alt="image" src="https://github.com/user-attachments/assets/a3cdd819-5a0f-4090-a6eb-a827f3cba7b7" />




This command deletes the EKS control plane and all associated resources (like node groups, networking, etc.). It may take a few minutes to complete. The eksctl output shows me the progress of deleting stacks and confirms when everything is fully deleted.



I revoked any temporary credentials or tokens if needed (for example, I can keep the SonarQube token or Slack token for future use, but if this was a one-off setup, I remove the project or regenerate the tokens). I can also clean up my Docker Hub repository by deleting the test image tag if I want to.



After these steps, my AWS environment returned to baseline (no cluster, no app running, no EC2), which helped me avoid unnecessary costs. My repository remains fully configured, so I can re-provision and rerun the pipeline anytime. Including cleanup in the documentation highlights my responsible cloud usage and shows thoroughness in my DevOps workflow.



**In conclusion, this project’s README documents how I set up a comprehensive CI/CD pipeline integrating Terraform, GitHub Actions, SonarQube, Trivy, Slack, Docker Hub, and AWS EKS. My pipeline automates code quality checks, security scans, Docker builds, and Kubernetes deployment, reflecting a production-like DevOps lifecycle. By following this approach, I can achieve rapid and safe software delivery with immediate feedback at every stage. This project demonstrates my proficiency across modern DevOps tools and reinforces the value of automating quality and security into CI/CD (DevSecOps) for reliable software delivery.**












