# Deploy the 2048 Game on AWS EKS

Type each command one at a time and wait for it to finish before typing the next.

---

## Prerequisites

You need 4 tools: **AWS CLI**, **kubectl**, **eksctl** and **helm**.

- **Windows:** install WSL first (open PowerShell as Administrator, run `wsl --install`, restart). Then open **Ubuntu** and follow the Linux steps.
- **Mac:** follow the Mac steps.
- **Linux / Ubuntu:** follow the Linux steps.

### Linux (and Windows WSL)

**AWS CLI**

```bash
sudo apt update && sudo apt install -y unzip curl
```

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
```

```bash
unzip awscliv2.zip
```

```bash
sudo ./aws/install --update
```

**kubectl**

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

```bash
sudo install -m 0755 kubectl /usr/local/bin/kubectl
```

**eksctl**

```bash
curl -sL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz" | tar xz -C /tmp
```

```bash
sudo mv /tmp/eksctl /usr/local/bin
```

**helm**

```bash
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### Mac

Install [Homebrew](https://brew.sh) first if you don't have it, then:

```bash
brew install awscli kubectl eksctl helm
```

### Check all 4 tools

```bash
aws --version
```

```bash
kubectl version --client
```

```bash
eksctl version
```

```bash
helm version
```

Each command should print a version number.

---

## 1. Log in to AWS

First create an access key: AWS Console → **IAM** → **Users** → your user → **Security credentials** → **Create access key** → choose **Command Line Interface (CLI)**. Copy both keys.

```bash
aws configure
```

Enter:

- **Access Key ID:** paste it
- **Secret Access Key:** paste it
- **Default region name:** `ap-south-1`
- **Default output format:** `json`

Check it works:

```bash
aws sts get-caller-identity
```

It should print your account number.

---

## 2. Create the cluster (takes about 20 minutes)

```bash
eksctl create cluster --name demo-cluster --region ap-south-1 --fargate
```

---

## 3. Connect to the cluster

```bash
aws eks update-kubeconfig --name demo-cluster --region ap-south-1
```

---

## 4. Allow the game to run on Fargate

```bash
eksctl create fargateprofile --cluster demo-cluster --region ap-south-1 --name alb-sample-app --namespace game-2048
```

---

## 5. Deploy the game (4 files, one by one)

You need these 4 files: `namespace.yaml`, `deployment.yaml`, `service.yaml`, `ingress.yaml`. Put them in the folder where you run your commands (in CloudShell: **Actions → Upload file**). Then apply them one by one, in this order.

### 5a. Namespace (a separate space for the game)

File: **namespace.yaml**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: game-2048
```

```bash
kubectl apply -f namespace.yaml
```

### 5b. Deployment (runs the game)

File: **deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: game-2048
  name: deployment-2048
spec:
  replicas: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: app-2048
  template:
    metadata:
      labels:
        app.kubernetes.io/name: app-2048
    spec:
      containers:
      - name: app-2048
        image: public.ecr.aws/l6m2t8p7/docker-2048:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 80
```

```bash
kubectl apply -f deployment.yaml
```

```bash
kubectl get pods -n game-2048
```

Wait until both pods show **Running** (takes 1–2 minutes). If they show **Pending**, wait and run the same command again.

### 5c. Service (gives the game pods one stable name)

File: **service.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  namespace: game-2048
  name: service-2048
spec:
  type: NodePort
  selector:
    app.kubernetes.io/name: app-2048
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
```

```bash
kubectl apply -f service.yaml
```

```bash
kubectl get service -n game-2048
```

### 5d. Ingress (the rule that lets the internet reach the game)

File: **ingress.yaml**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  namespace: game-2048
  name: ingress-2048
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: service-2048
              port:
                number: 80
```

```bash
kubectl apply -f ingress.yaml
```

```bash
kubectl get ingress -n game-2048
```

The **ADDRESS** is empty for now. That's normal: steps 6–9 install the controller that creates the load balancer.

---

## 6. Turn on OIDC

```bash
eksctl utils associate-iam-oidc-provider --cluster demo-cluster --region ap-south-1 --approve
```

---

## 7. Create the IAM policy

```bash
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
```

```bash
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json
```

If you get "already exists", that's fine. Just continue.

---

## 8. Create the IAM role

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
```

```bash
eksctl create iamserviceaccount --cluster demo-cluster --region ap-south-1 --namespace kube-system --name aws-load-balancer-controller --role-name AmazonEKSLoadBalancerControllerRole --attach-policy-arn arn:aws:iam::$ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy --approve
```

---

## 9. Install the load balancer controller

```bash
VPC_ID=$(aws eks describe-cluster --name demo-cluster --region ap-south-1 --query "cluster.resourcesVpcConfig.vpcId" --output text)
```

```bash
helm repo add eks https://aws.github.io/eks-charts
```

```bash
helm repo update eks
```

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=demo-cluster --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller --set region=ap-south-1 --set vpcId=$VPC_ID
```

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

Wait until it shows **2/2** (takes 1–2 minutes). If it shows **0/2**, wait and run the same command again.

---

## 10. Get the game link

```bash
kubectl get ingress -n game-2048
```

Copy the **ADDRESS** and open it in your browser as `http://<ADDRESS>`.
If ADDRESS is empty, wait 1 minute and run the command again.
If the page doesn't load, wait 3 minutes and refresh.

---

## 11. Delete everything (don't skip this)

Delete the Ingress first (this removes the load balancer):

```bash
kubectl delete -f ingress.yaml
```

Wait 2 minutes, then delete the rest of the game:

```bash
kubectl delete -f service.yaml
```

```bash
kubectl delete -f deployment.yaml
```

```bash
kubectl delete -f namespace.yaml
```

Then delete the cluster:

```bash
eksctl delete cluster --name demo-cluster --region ap-south-1
```

This takes about 10–15 minutes. Then check in the AWS Console (region Mumbai) that **EC2 → Load Balancers** is empty.

---

## If something goes wrong

| Problem | Fix |
|---|---|
| Pods stay **Pending** | Step 4 was skipped or mistyped. Run step 4 again. |
| `error: the path "xxx.yaml" does not exist` | The file isn't in your current folder. Run `ls` to check, upload it again, or `cd` to the folder that has it. |
| Controller stays **0/2** | Run steps 6 and 8 again, then run the `helm install` line again with `upgrade --install` instead of `install`. |
| ADDRESS stays empty after 5 minutes | Run `kubectl logs -n kube-system deploy/aws-load-balancer-controller` and look for `AccessDenied`. If you see it, steps 6 or 8 were skipped. |
| `cluster already exists` / `stack already exists` | A previous try was left behind. Delete the `eksctl-demo-cluster-...` stack in **CloudFormation**, then retry. |
| Command says `ACCOUNT_ID` or `VPC_ID` is empty | Run that line from step 8 or 9 again. |

---

## Notes

- If you close the terminal, run the `ACCOUNT_ID` or `VPC_ID` line again before the step that uses it.
- No setup on your laptop? You can use **AWS CloudShell** (terminal icon at the top of the AWS Console) instead. It already has AWS CLI and kubectl and is logged in, so you only install eksctl and helm (Linux commands above) and skip step 1.
- If many students share one AWS account, each student uses a different cluster name instead of `demo-cluster` (for example `demo-rahul`) and a different role name in step 8 (for example `AmazonEKSLoadBalancerControllerRole-rahul`).
