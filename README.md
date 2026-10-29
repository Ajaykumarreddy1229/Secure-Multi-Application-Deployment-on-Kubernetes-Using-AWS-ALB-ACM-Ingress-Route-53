# Secure-Multi-Application-Deployment-on-Kubernetes-Using-AWS-ALB-ACM-Ingress-Route-53
# Secure Kubernetes Applications with HTTPS Using AWS ACM, ALB, KOPS, and Route 53

## 1. Project Overview

This project demonstrates how to deploy multiple applications in a Kubernetes cluster and securely expose them to users through HTTPS using AWS services.

We use **KOPS** to manage the Kubernetes cluster, **AWS Certificate Manager (ACM)** to provide an SSL/TLS certificate, **AWS Application Load Balancer (ALB)** to handle incoming traffic, and **Amazon Route 53** to connect a custom domain name.

The project also demonstrates **path-based routing**, where a single Application Load Balancer directs incoming requests to different Kubernetes applications based on the URL path.

We deploy four applications:

1. Internet Banking Application
2. Mobile Banking Application
3. Insurance Application
4. Loans Application

Each application runs as a Kubernetes Deployment with three replicas.

### Project Objective

The main objectives are:

- Deploy multiple applications inside Kubernetes.
- Configure Kubernetes Deployments and Services.
- Implement path-based routing using Kubernetes Ingress.
- Enable HTTPS using AWS ACM.
- Redirect HTTP traffic to HTTPS.
- Use a single ALB for multiple applications.
- Configure DNS using Amazon Route 53.
- Improve application availability using multiple replicas and health checks.

## 2. Technologies Used

| Technology | Purpose |
|---|---|
| AWS EC2 | Hosts Kubernetes worker and control-plane nodes |
| KOPS | Creates and manages the Kubernetes cluster |
| Kubernetes | Manages containerized applications |
| Docker | Packages applications into container images |
| Kubernetes Deployment | Manages application replicas |
| Kubernetes NodePort Service | Exposes applications on Kubernetes nodes |
| Kubernetes Ingress | Defines external routing rules |
| AWS Load Balancer Controller | Creates and manages AWS ALBs |
| AWS ALB | Distributes incoming application traffic |
| AWS ACM | Provides the SSL/TLS certificate |
| Amazon Route 53 | Manages domain DNS records |
| AWS IAM | Provides permissions to AWS resources |
| Helm | Installs Kubernetes applications and controllers |

## 3. AWS Architecture

The architecture follows this flow:

```text
                       USERS
                         |
                         |
                    AMAZON ROUTE 53
                         |
                         |
                    HTTPS PORT 443
                         |
                         v
              AWS APPLICATION LOAD BALANCER
                         |
                   ACM CERTIFICATE
                         |
                         |
               PATH-BASED ROUTING
                         |
       -----------------------------------------
       |            |             |            |
 /ib-service   /mb-service  /insurance-service /loans-service
       |            |             |            |
  ib-service   mb-service   insurance-service loans-service
  (NodePort)   (NodePort)      (NodePort)      (NodePort)
       |            |             |            |
 ib-deployment mb-deployment insurance-deployment loans-deployment
       |            |             |            |
    3 Pods        3 Pods        3 Pods        3 Pods
```

The AWS Load Balancer Controller watches Kubernetes Ingress resources and configures the ALB.

It does not handle application traffic directly.

The ALB receives incoming HTTPS requests and forwards them to the appropriate Kubernetes Service through registered targets.

## 4. How the Project Works

### Step 1: User Accesses the Application

The user opens the application domain:

`https://k8s.ajayclouddevops.online`

The request starts from the user's browser.

### Step 2: Route 53 Resolves the Domain

Amazon Route 53 manages the DNS record for the application.

It resolves the custom domain name to the AWS Application Load Balancer.

### Step 3: Application Load Balancer Receives Traffic

The ALB accepts incoming HTTPS traffic on port 443.

The ACM certificate is attached to the ALB's HTTPS listener.

The ALB uses this certificate to establish a secure TLS connection with the user's browser.

### Step 4: ALB Checks Routing Rules

The ALB evaluates the request URL and selects the appropriate target group.

For example:

| URL Path | Kubernetes Service |
|---|---|
| `/ib-service` | ib-service |
| `/mb-service` | mb-service |
| `/insurance-service` | insurance-service |
| `/loans-service` | loans-service |

### Step 5: Traffic Reaches Kubernetes Services

Each application is exposed using a NodePort Service.

With ALB instance target mode, the ALB registers Kubernetes nodes and forwards requests to the corresponding NodePort.

The Kubernetes Service then directs requests to an available application Pod.

### Step 6: Application Responds

The selected Pod processes the request and returns a response.

The response travels back through the ALB to the user.

## 5. Prerequisites

Before starting, make sure you have:

- An AWS account
- AWS CLI installed and configured
- kubectl installed
- KOPS installed
- Helm installed or ready to install
- A Kubernetes cluster created using KOPS
- A registered domain name
- A Route 53 hosted zone
- Required IAM permissions
- Application Docker images
- Appropriate VPC and subnet configuration

## 6. Verify the KOPS Cluster

First, verify the Kubernetes nodes.

**Command:**

```bash
kubectl get nodes
```

All required nodes should show the `Ready` status.

Also verify the cluster:

```bash
kubectl cluster-info
```

Check existing Pods:

```bash
kubectl get pods -A
```

## 7. Create an SSL/TLS Certificate Using AWS ACM

AWS Certificate Manager helps us provision and manage SSL/TLS certificates.

### Steps

1. Open the AWS Management Console.
2. Search for AWS Certificate Manager.
3. Select the Mumbai Region (`ap-south-1`).
4. Click Request Certificate.
5. Choose Public Certificate.
6. Enter the domain name.
7. Select DNS validation.
8. Add the required validation records in Route 53.
9. Wait until the certificate status becomes Issued.
10. Copy the certificate ARN.

**Example domain:**

```text
*.ajayclouddevops.online
```

This wildcard certificate can secure subdomains such as `k8s.ajayclouddevops.online`.

The ACM certificate must be in the same AWS Region as the ALB.

## 8. Create Kubernetes Deployments

A Kubernetes Deployment manages application Pods.

It helps maintain the required number of replicas and replaces failed Pods.

In this project, we deploy four applications with three replicas each.

### Deployment Details

| Deployment | Replicas |
|---|---:|
| ib-deployment | 3 |
| mb-deployment | 3 |
| insurance-deployment | 3 |
| loans-deployment | 3 |
| **Total** | **12** |

### Example: Insurance Deployment

Create the file:

```bash
vi insurance-deployment.yml
```

Add the following YAML:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: insurance-deployment
  labels:
    app: insurance

spec:
  replicas: 3

  selector:
    matchLabels:
      app: insurance

  template:
    metadata:
      labels:
        app: insurance

    spec:
      containers:
        - name: insurance-container
          image: trainerreyaz/insurance-image:latest

          ports:
            - containerPort: 80

          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 15
            periodSeconds: 10
            failureThreshold: 3

          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 2

          startupProbe:
            httpGet:
              path: /
              port: 80
            failureThreshold: 30
            periodSeconds: 10
```

### Apply Deployment

```bash
kubectl apply -f insurance-deployment.yml
```

### Verify Deployment

```bash
kubectl get deployments
```

```bash
kubectl get pods
```

**Note:** The other three Deployments require their respective container images. The insurance image is the only image specified in the original project notes.

## 9. Configure Kubernetes Health Probes

Kubernetes provides three important health probes.

### Liveness Probe

Checks whether the application is running properly.

If the probe repeatedly fails, Kubernetes restarts the container.

### Readiness Probe

Checks whether the application is ready to receive requests.

If the application is not ready, Kubernetes removes the Pod from the Service's ready endpoints.

### Startup Probe

Allows the application additional time to start.

Until the startup probe succeeds, Kubernetes does not execute the liveness and readiness probes.

These probes help improve application reliability.

## 10. Create Kubernetes NodePort Services

A Kubernetes Service provides a stable way to access application Pods.

In this project, we use NodePort Services because the ALB uses instance target mode.

### Example: Insurance Service

Create the file:

```bash
vi insurance-service.yml
```

Add:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: insurance-service

spec:
  type: NodePort

  selector:
    app: insurance

  ports:
    - port: 80
      targetPort: 80
      nodePort: 31433
```

Apply:

```bash
kubectl apply -f insurance-service.yml
```

Verify:

```bash
kubectl get svc
```

The Service listens on port 80 and exposes NodePort 31433.

The other three applications also require their own NodePort Services with unique NodePort numbers.

## 11. Configure AWS IAM Permissions

The AWS Load Balancer Controller needs IAM permissions to manage AWS networking resources.

These include:

- Application Load Balancers
- Target Groups
- Listeners
- Security Group rules
- Load Balancer configurations

For a KOPS cluster, configure an appropriate IAM role and supported AWS credential mechanism for the controller.

Use the official AWS Load Balancer Controller IAM policy.

Avoid granting full administrator permissions to worker nodes.

## 12. Install Helm

Helm is a Kubernetes package manager.

It helps install and manage Kubernetes applications using Helm charts.

### Download Helm

```bash
curl -fsSL -o get_helm.sh \
https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
```

### Give Permission

```bash
chmod 700 get_helm.sh
```

### Install Helm

```bash
./get_helm.sh
```

### Verify Installation

```bash
helm version
```

## 13. Install AWS Load Balancer Controller

The AWS Load Balancer Controller integrates Kubernetes Ingress with AWS Application Load Balancers.

When we create an Ingress resource, the controller reads the configuration and creates or updates the corresponding AWS ALB.

### Add Helm Repository

```bash
helm repo add eks https://aws.github.io/eks-charts
```

### Update Helm Repository

```bash
helm repo update
```

### Install Controller

The following example assumes a Kubernetes service account named `aws-load-balancer-controller` has already been created and configured with the required AWS IAM permissions.

```bash
helm install aws-load-balancer-controller \
  eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=reyaz.k8s.local \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

Replace `reyaz.k8s.local` with your actual KOPS cluster name.

For KOPS clusters, additional settings may be required for VPC discovery, AWS credentials, networking, and controller compatibility.

### Verify Controller

```bash
kubectl get pods -n kube-system | grep aws-load-balancer
```

**Important:** cert-manager is not required when using an existing ACM certificate directly with an ALB.

## 14. Understand Kubernetes Ingress

Kubernetes Ingress defines rules for routing external HTTP and HTTPS traffic to internal Services.

It supports two important routing methods.

### Host-Based Routing

Traffic is routed based on the domain name.

Example:

- `bank.example.com` → Banking Service
- `insurance.example.com` → Insurance Service

### Path-Based Routing

Traffic is routed based on the URL path.

Example:

- `/ib-service` → Internet Banking
- `/mb-service` → Mobile Banking
- `/insurance-service` → Insurance
- `/loans-service` → Loans

In this project, we use path-based routing.

## 15. Configure Kubernetes Ingress with AWS ALB

Create an Ingress file:

```bash
vi ingress.yml
```

### Ingress YAML

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress

metadata:
  name: banking-ingress

  annotations:
    # Internet-facing ALB
    alb.ingress.kubernetes.io/scheme: internet-facing

    # Register nodes as targets
    alb.ingress.kubernetes.io/target-type: instance

    # HTTP and HTTPS listeners
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80},{"HTTPS":443}]'

    # Redirect HTTP to HTTPS
    alb.ingress.kubernetes.io/ssl-redirect: "443"

    # ACM Certificate
    alb.ingress.kubernetes.io/certificate-arn: "REPLACE_WITH_YOUR_ACM_CERTIFICATE_ARN"

    # Health Check
    alb.ingress.kubernetes.io/healthcheck-path: /
    alb.ingress.kubernetes.io/healthcheck-port: traffic-port

spec:
  ingressClassName: alb

  rules:
    - host: k8s.ajayclouddevops.online

      http:
        paths:
          - path: /ib-service
            pathType: Prefix
            backend:
              service:
                name: ib-service
                port:
                  number: 80

          - path: /mb-service
            pathType: Prefix
            backend:
              service:
                name: mb-service
                port:
                  number: 80

          - path: /insurance-service
            pathType: Prefix
            backend:
              service:
                name: insurance-service
                port:
                  number: 80

          - path: /loans-service
            pathType: Prefix
            backend:
              service:
                name: loans-service
                port:
                  number: 80
```

### Important Configuration

Before applying this YAML:

- Replace the ACM certificate ARN.
- Make sure all four Services exist.
- Make sure each Service has healthy backend Pods.
- Configure suitable ALB subnets and subnet discovery.
- Confirm the AWS Load Balancer Controller is running.

ALB forwards the original URL path by default. Therefore, each backend application must support its configured path, or you must add suitable URL rewrite or application routing configuration.

### Apply Ingress

```bash
kubectl apply -f ingress.yml
```

### Verify Ingress

```bash
kubectl get ingress
```

### Describe Ingress

```bash
kubectl describe ingress banking-ingress
```

## 16. Configure Amazon Route 53

After creating the Ingress, the AWS Load Balancer Controller provisions an ALB.

The ALB has a DNS name.

We use Route 53 to connect our custom domain to the ALB.

### Steps

1. Open Amazon Route 53.
2. Select the hosted zone `ajayclouddevops.online`.
3. Click Create Record.
4. Enter `k8s` as the record name.
5. Select A record.
6. Enable Alias.
7. Select the Application Load Balancer.
8. Save the DNS record.

### Application Domain

```text
k8s.ajayclouddevops.online
```

## 17. Access Applications Using HTTPS

After completing the configuration, the intended application URLs are:

### Internet Banking

```text
https://k8s.ajayclouddevops.online/ib-service
```

### Mobile Banking

```text
https://k8s.ajayclouddevops.online/mb-service
```

### Insurance

```text
https://k8s.ajayclouddevops.online/insurance-service
```

### Loans

```text
https://k8s.ajayclouddevops.online/loans-service
```

These URLs work when the corresponding applications, Services, Ingress rules, ALB, and DNS records are configured correctly.

## 18. Verify HTTPS

Test HTTPS:

```bash
curl -I https://k8s.ajayclouddevops.online/insurance-service
```

Test HTTP-to-HTTPS redirection:

```bash
curl -I http://k8s.ajayclouddevops.online/insurance-service
```

The HTTP request should redirect to HTTPS.

Also verify that the ALB target groups show healthy targets in the AWS Console.

## 19. Troubleshooting Commands

### Check Kubernetes Nodes

```bash
kubectl get nodes
```

### Check All Pods

```bash
kubectl get pods -o wide
```

### Check Deployments

```bash
kubectl get deployments
```

### Check Services

```bash
kubectl get svc
```

### Check Ingress

```bash
kubectl get ingress
```

### Describe Ingress

```bash
kubectl describe ingress banking-ingress
```

### Check Controller Logs

```bash
kubectl logs -n kube-system \
deployment/aws-load-balancer-controller
```

### Common Problems

| Problem | Possible Reason |
|---|---|
| ALB not created | Controller, IAM, or subnet configuration issue |
| HTTPS not working | Invalid or incorrect ACM certificate |
| 503 error | No healthy backend targets |
| 404 error | Incorrect routing path or unsupported application path |
| DNS not resolving | Incorrect Route 53 record |
| Pods not running | Container image or application issue |
| Health checks failing | Incorrect health-check path or port |

## 20. Security Best Practices

- Use HTTPS for public application access.
- Use valid ACM certificates.
- Redirect HTTP requests to HTTPS.
- Follow IAM least-privilege principles.
- Avoid exposing unnecessary NodePorts publicly.
- Configure appropriate security group rules.
- Keep Kubernetes and controller versions updated.
- Use readiness and liveness probes.
- Monitor ALB target group health.
- Use AWS WAF when additional web application protection is required.

**Security Note:** In this architecture, TLS terminates at the ALB, and the ALB-to-Pod connection uses HTTP. For end-to-end encryption, configure HTTPS on the backend as well.

## 21. Key Learning Outcomes

Through this project, we learn:

1. How to manage a Kubernetes cluster using KOPS.
2. How Kubernetes Deployments manage application replicas.
3. How NodePort Services expose applications.
4. How health probes improve reliability.
5. How to install Helm and AWS Load Balancer Controller.
6. How Kubernetes Ingress supports path-based routing.
7. How AWS ACM provides TLS certificates.
8. How ALB handles HTTPS traffic.
9. How Route 53 connects domains to AWS resources.
10. How multiple applications can share a single ALB.

## 22. Conclusion

This project demonstrates how Kubernetes and AWS services work together to securely expose multiple containerized applications through a single Application Load Balancer.

We explored Kubernetes Deployments, NodePort Services, health probes, Ingress routing, AWS Certificate Manager, IAM, Helm, AWS Load Balancer Controller, and Amazon Route 53.

By combining these technologies, we can build a scalable architecture that supports multiple applications, HTTPS connections, and path-based traffic routing.

This project is useful for learning real-world Kubernetes networking and AWS cloud integration.

**DevOps | AWS | Kubernetes | Cloud Engineering | Hands-on Learning**
