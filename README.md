
## CI/CD Pipeline with GitHub Actions (Terraform, SonarQube, Trivy, Slack, EKS)


Introduction



<img width="1454" height="802" alt="image" src="https://github.com/user-attachments/assets/e6a518f0-42f7-4eb0-b91b-89b76dc05613" />






This project demonstrates a full CI/CD DevOps pipeline that builds, analyzes, secures, and deploys a containerized application using GitHub Actions. The pipeline is designed to showcase DevOps best practices end-to-end, from infrastructure provisioning to continuous deployment on Kubernetes. Key components include:



Infrastructure as Code (Terraform): Automating creation of an AWS EC2 instance to host a self-hosted GitHub Actions runner, Docker engine, and a SonarQube server.

Code Quality Analysis (SonarQube): Integrating SonarQube for static code analysis to improve code quality (catch bugs, code smells, etc.).

Security Scanning (Trivy): Performing security scans on source code (files and dependencies) as well as container images to detect vulnerabilities, misconfigurations, and secrets early

Containerization (Docker): Building a Docker image of the application and pushing it to a registry (e.g. Docker Hub).

Continuous Deployment to Kubernetes (EKS): Deploying the Docker image to an AWS EKS (Elastic Kubernetes Service) cluster using kubectl/eksctl, demonstrating Kubernetes deployment automation.

Notifications (Slack): Sending Slack notifications on pipeline results for real-time feedback to the team.

**The goal is to implement a robust pipeline that ensures code quality and security at each stage (using SonarQube and Trivy scans) and automates deployment to a production-like environment (AWS EKS). This end-to-end setup highlights strong DevOps skills in infrastructure provisioning, CI/CD automation, security (DevSecOps), and cloud orchestration, which would be clear and impressive to recruiters and interviewers.*




## Creating the EC2 Infrastructure with Terraform (Self-Hosted Runner Host)

To have more control over the CI environment, we use a self-hosted GitHub Actions runner on AWS. We provisioned an EC2 instance using Terraform as our runner host (which also runs Docker and SonarQube). Terraform allows us to define this infrastructure as code, ensuring repeatability and consistency. Key steps include:

Terraform Configuration: We wrote Terraform configs to define an EC2 instance (Ubuntu 20.04) with the required security group rules. For example, we opened port 22 for SSH, port 9000 for SonarQube’s web UI, and any needed ports for runner communication (GitHub runners use outbound HTTPS).

Applying Terraform: Using Termius (or any terminal), we ran Terraform commands to create the resources:


```
terraform init            # Initialize Terraform and AWS provider
terraform plan
terraform apply -auto-approve   # Create EC2 instance and related infrastructure

```

After apply, Terraform outputs the new EC2’s public IP


We use that IP to SSH into the server.

Instance Setup: Once the EC2 is up, we SSH in and install necessary tools: Docker (to run SonarQube and for building images), and any dependencies for the runner. For example:



<img width="1504" height="628" alt="image" src="https://github.com/user-attachments/assets/a8eac559-6cb8-4af1-ad2b-e8975c6f0968" />



We then launch a SonarQube container on this host and ensure it’s accessible on port 9000:


```

docker run -d --name sonarqube -p 9000:9000 sonarqube:lts

```

For a production-grade setup, SonarQube would use an external database and more memory, but for this pipeline demo the above is sufficient.



Running CI on GitHub-Hosted Runner (ubuntu-latest)

With the EC2 server up and running Docker and SonarQube, the next step was to run our CI pipeline using a GitHub-hosted runner instead of a self-hosted one. This means the workflow executes on GitHub’s infrastructure (ubuntu-latest), and connects to the tools we set up on EC2 over the network.

Using a GitHub-Hosted Runner: In our GitHub Actions workflow, we didn’t add or register any self-hosted runner. Instead, we kept the default GitHub runner by specifying ubuntu-latest in the YAML. For example:

```
jobs:
build-scan:
runs-on: ubuntu-latest # GitHub-hosted runner
# ...
```


We also configured required credentials as encrypted secret keys in the repository (GitHub Secrets) to securely support the CI/CD pipeline.

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


We chose to allow the job to continue even if issues are found (exit-code: 0), so that the pipeline can proceed (you might instead fail the build on high-severity issues in a stricter scenario). The results are logged in the workflow. For instance, Trivy would list any CVEs in our application dependencies, any AWS credential leaks or hardcoded secrets, and any insecure configurations (like a Terraform file with an open security group). This automated scan helps “identify vulnerabilities before they make it to production”



Trivy FS Example: If running manually in a terminal, the equivalent would be:
```
trivy fs --scanners vuln,secret,config --severity CRITICAL,HIGH,MEDIUM .

```

The output would list findings for each category. Integrating this into CI ensures every code change is scrutinized for security issues as part of the build process.



## Docker Image Build and Push (CI Pipeline)

After code is analyzed and cleared, the pipeline builds a Docker image for the application and pushes it to a container registry. Containerizing the app ensures consistency between environments and is required for deployment to EKS.




Building the Docker Image: In the GitHub Actions workflow, we use Docker to build the image from our Dockerfile. Since we have Docker installed, we can either run Docker commands directly or use the Docker Buildx GitHub Action. We opted to run the commands directly for transparency. For example, our workflow has a step:


```
- name: Build Docker Image
  run: docker build -t myapp:latest 
```

This instructs Docker to build an image tagged myapp:latest from the repository’s Dockerfile. (If needed, we could also build multiple tags, e.g., commit SHA tag for versioning.)

Authenticating to Registry: We chose Docker Hub as the container registry (for simplicity and because our EKS cluster can pull public images from Docker Hub easily). We created a Docker Hub repository and added our Docker Hub credentials (username and an access token) as GitHub secrets. These secrets (DOCKER_USERNAME and DOCKERHUB_TOKEN) allow the workflow to log in to Docker Hub
docs.docker.com


 The login step in the workflow uses the Docker CLI or a GitHub Action:


```
- name: Login to Docker Hub
  run: echo "${{ secrets.DOCKERHUB_TOKEN }}" | docker login -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin
```

This logs in the Docker CLI to Docker Hub using our saved credentials (without exposing them in logs).

Pushing the Image: Once logged in, we push the image to the Docker Hub repository. We tag the image with the repo name, then push:


<img width="3024" height="1964" alt="image" src="https://github.com/user-attachments/assets/9bcc35c8-3d39-4940-b4a0-0946e04653fb" />



```
- name: Push Image to Registry
  run: |
    docker tag myapp:latest ${{ secrets.DOCKER_USERNAME }}/myapp:latest
    docker push ${{ secrets.DOCKER_USERNAME }}/myapp:latest
```

This publishes the Docker image to Docker Hub (e.g., dockerhubuser/myapp:latest). In our case, the workflow waited briefly for Docker Hub to process the new image tag (ensuring it’s available for the next step).

By the end of this stage, our CI pipeline has produced a versioned Docker image of the application and stored it in a registry, ready for deployment. Using GitHub Actions to automate build and push ensures that any commit to the repo results in a fresh container image, eliminating “it works on my machine” issues.



### Container Image Vulnerability Scanning with Trivy

After pushing the Docker image, the pipeline performs a Trivy scan on the image to detect any vulnerabilities in the image’s OS packages or libraries. This adds another layer of security by scanning the final artifact that will run in production.

Why Image Scanning: Even if our source dependencies were clean, vulnerabilities could be introduced via the base image (e.g., OS packages). Trivy’s image scan checks for known CVEs in the image layers. This helps catch critical issues in the container (for example, a critical CVE in the base Linux distribution) before deployment



Trivy Image Scan in Workflow: We again use the Aqua Security action, but in image mode. The step in YAML looks like:


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



Sample Trivy Image Output: Your Trivy might output something like:



myapp:latest (alpine 3.18)
============================================
Total: 2 (HIGH: 1, CRITICAL: 1)

CRITICAL CVE-2023-XXXX – openssl – Fixed in 1.1.1t – Vulnerability details...
HIGH     CVE-2022-YYYY – libcurl – Fixed in 7.85.0 – Vulnerability details...




This tells us what to patch or update. By having this step, we demonstrate a proactive stance on container security: any issues in the image are immediately known and can be addressed before or after deployment as needed.




Slack Notifications for CI/CD Feedback

To keep the team informed of pipeline results, we integrated Slack notifications into the GitHub Actions workflow. This means that whenever a build/deploy succeeds or fails, a message is sent to a specified Slack channel.



<img width="3024" height="1964" alt="image" src="https://github.com/user-attachments/assets/b27eb00c-1650-4cc0-9269-d9c114fa9bd9" />




Why Slack Integration: Teams often “live” in Slack, so sending CI/CD alerts there ensures fast visibility into the pipeline status


Immediate failure alerts can significantly reduce time to response and fix, improving DevOps feedback loops. It encourages collaboration – everyone knows when a deployment happens or if something breaks, without having to check the GitHub UI constantly.


<img width="3024" height="1964" alt="image" src="https://github.com/user-attachments/assets/1e794495-8730-425d-9960-fb0fcf58b1b8" />



Slack App Setup: We created a Slack app incoming webhook for our workspace. The simplest method is using an Incoming Webhook URL (which posts to a channel). Alternatively, one can use Slack’s Bot token and the Slack API. 



In our case, we used the official Slack GitHub Action 

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

AWS Credentials in Workflow: Next, we configure AWS credentials in the job environment. We use the AWS Actions module:


```
- name: Configure AWS Credentials
  uses: aws-actions/configure-aws-credentials@v2
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: us-east-1
```

This exports the AWS creds so that AWS CLI and kubectl (with AWS IAM Auth) can function in subsequent steps



Updating Kubeconfig: To allow kubectl to talk to our EKS cluster, we update the Kubernetes config for the cluster. We did this by running:


```
- name: Configure kubeconfig for EKS
  run: aws eks update-kubeconfig --name virtualtechbox-cluster --region us-east-1
```

This command fetches the cluster’s endpoint and credentials and merges them into ~/.kube/config on the runner

 
 Essentially, it makes sure kubectl commands know which cluster to operate on.

Deploying the Application: We have Kubernetes manifest files in our repository (e.g., deployment.yaml and service.yaml) describing how to run our container in EKS. The deployment manifest uses the Docker image we pushed earlier. We apply these manifests:


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




In the screenshots above, you can see our two workflow jobs in GitHub Actions. The first job runs on ubuntu latest and performs the build, analysis, and scanning steps. The second job then deploys to AWS EKS and notifies Slack. Both jobs have succeeded, indicating our pipeline worked end-to-end.

Pipeline Validation and Verification

After the pipeline ran, we performed a few checks to validate that everything worked as expected:

GitHub Actions Logs: We reviewed the workflow run in GitHub Actions (as shown in the screenshots) to ensure each step completed. The logs confirmed that SonarQube analysis was successful (the scanner reported results to our SonarQube server), Trivy scans completed (with output of any findings), the Docker image built and pushed without errors, and the Kubernetes deployment was applied. Seeing green checkmarks on all steps and jobs is a good sign that the CI/CD pipeline executed flawlessly.

SonarQube Dashboard: We logged into SonarQube and checked the project dashboard. The latest analysis corresponding to the pipeline run was present. We could see metrics like code coverage (if configured), code smells, vulnerabilities, etc., giving us confidence in code quality. SonarQube integration means any new issue introduced by a commit would be visible here immediately.


<img width="1226" height="732" alt="image" src="https://github.com/user-attachments/assets/472f4fab-01b6-438d-b798-10fec36ebf7a" />



Kubernetes Resources: We used kubectl to verify the application is running in the EKS cluster......


```
kubectl get pods
kubectl get svc
```

These commands showed an active Deployment (e.g., swiggy-app deployment with the desired number of pods running) and a Service exposing it.






We also port-forwarded or hit the Service’s external LoadBalancer (if configured) to ensure the app was reachable. This confirms the deployment part of the pipeline succeeded and the new Docker image is running on EKS.

Slack Message: We checked Slack and found the notification from the pipeline. The message indicated a successful deployment, along with commit info. This real-time confirmation is useful for the team to know the status. In case of a failure, the Slack message would have alerted us immediately with the failure status and we could drill into logs via the GitHub link.

By cross-verifying in SonarQube, the EKS cluster, and Slack, we ensured that each integration in the pipeline (code analysis, security scan, deployment, and notification) worked correctly. The pipeline not only ran without errors, but it also achieved the intended outcomes: high confidence in code & image quality and automated deployment.



### Cleanup and Teardown



After validating the pipeline, we cleaned up the resources to avoid ongoing costs and keep the environment tidy:

Kubernetes Deployments: We deleted the deployed application from the EKS cluster (especially since this was a demo deployment). Using kubectl:


```
kubectl delete deployment swiggy-app
kubectl delete service swiggy-app
```

(Replace swiggy-app with your deployment/service names.) This scales down and removes the app from the cluster.

EKS Cluster: Since the cluster was created for demonstration, we tore it down to avoid AWS charges. We used eksctl to delete the cluster:

```
eksctl delete cluster --name=virtualtechbox-cluster --region=us-east-1
```


<img width="2310" height="1274" alt="image" src="https://github.com/user-attachments/assets/a3cdd819-5a0f-4090-a6eb-a827f3cba7b7" />




This command deletes the EKS control plane and all associated resources (like node groups, networking, etc.). It may take a few minutes to complete. The eksctl output shows the progress of deleting stacks and confirms when everything is deleted




We revoked any temporary credentials or tokens if needed (for example, the SonarQube token or Slack token can be kept for future use, but if this were a one-off, you might remove the project or regenerate tokens). Docker Hub repository can be cleaned if desired by removing the test image tag.



After these steps, the AWS environment was back to baseline (no cluster, no app running, no EC2), and thus we avoid incurring costs. The repository remains configured, so one can re-provision and rerun the pipeline at any time. Cleanup is a best practice to include in documentation to highlight responsible usage of cloud resources and to demonstrate thoroughness in the DevOps workflow.





In conclusion this project’s README has documented how we set up a comprehensive CI/CD pipeline integrating Terraform, GitHub Actions, SonarQube, Trivy, Slack, and AWS EKS. The pipeline automates code quality checks, security scans, Docker builds, and deployment to Kubernetes, reflecting a production-like DevOps lifecycle. By following this approach, teams can achieve rapid and safe code delivery with immediate feedback at each stage. This demonstration not only underscores proficiency in various DevOps tools and practices but also emphasizes the value of automating quality and security into the CI/CD process (DevSecOps) for robust software delivery















