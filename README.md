# Three-tier-application setup in k8s 
# Three-tier-application Setup in K8s 
-----------------------------------------------------------
## Phase-1 (Basic Requirements for K8s)
1. Create IAM user with AdminAccess 
2. Generate AccessKey and SecretKey for CLI Access
3. Launch Ubuntu Ec2 machine install below tools
 - awscli
 - kubectl
 - eksctl
 - docker
4. setup aws configure 
5. Clone repo https://github.com/thej950/Three-tier-Application-Deployment.git
6. Create Docker image and push to ECR or DockerHub 
- fronted-img
- backend-img
-----------------------------------------------------
## Phase-2 (K8s Setup)
1. setup EKS cluster
- from commands
 ```bash
 eksctl create cluster --name three-tier-cluster --region us-east-1 --node-type t2.medium --nodes-min 2 --nodes-max 2 
 aws eks update-kubeconfig --region us-east-1 --name three-tier-cluster 
 kubectl get nodes 
 ```
 - it will take 15 min to create k8s cluster 
2. Setup workspace Namespace
 - from below commands
 ```bash 
 kubectl create namespace workshop 
 kubectl config set-context --current --namespace workshop 
 ```
3. setup frontend
 - create deployment for fronted 
 - modify manifest files inside k8s_manifests directory according to images setups 
 - run below commands in k8s_manifets directory for fronted
 ```bash
 kubectl apply -f frontend-deployment.yaml 
 kubectl apply -f frontend-service.yaml 
 ```
4. setup backend
 - these commands for backend
 ```bash
 kubectl apply -f backend-deployment.yaml 
 kubectl apply -f backend-service.yaml 
 ```
 - to view pods 
 ```bash
 kubectl get pods -n workshop
 ```

 > Note: Upto here Two tier is complete 

5. Setup Database tier 
 - Locate mango directory that containes deployment, service, screts manifests files 
 ```bash
 kubectl apply -f . 
 kubectl get all 
 ```

> Note: Now all tiers are setup 
-----------------------------------------------------
## Phase-3 (ALB and Ingress)
1. Create IAM policy for ALB Controller

 - Below command will download aws IAM policy for ALB
 ```bash
 curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json 
 ```
 -  create IAM policy in aws account 
 - from below commands
 ```bash
 aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json 
 ```
2. Associate IAM OIDC Provider (If Not Already Done)  
 - AWS Load Balancer Controller is required to manage ALB in EKS.
 - This command apply the load balancer policy to your eks cluster so that your eks cluster is working with your load balancer according to the policy
 ```bash
 eksctl utils associate-iam-oidc-provider --region=us-east-1 --cluster=three-tier-cluster --approve 
 ```
3. SETUP IAM Service Account to EKS 

  - Using Below command create and attach service account to your cluster that you cluster is allowed to work with load balancer service 
	
  ```bash
  eksctl create iamserviceaccount \
    --cluster=three-tier-cluster \
    --namespace=kube-system \
    --name=aws-load-balancer-controller \
    --role-name AmazonEKSLoadBalancerControllerRole \
    --attach-policy-arn=arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
    --approve --region=us-east-1
  ```
 > Note: Replace <ACCOUNT_ID> with your AWS account ID.
	
 > Note:  All the policies are attached lets deploy the load balancer 
4. Deploy AWS Load Balancer Controller Using Helm	
 - For this we have to install helm→Helm is a special tool that helps you easily carry and manage your software when you’re using Kubernetes, which is like a big playground for running applications. 

  ```bash
  sudo snap install helm --classic 
  ```
 - After this we have to add a particular manifest for load balancer that is pre written by someone on eks repo by using helm 
  ```bash
  helm repo add eks https://aws.github.io/eks-charts 
  ```
 - update the eks repo using helm
  ```bash
  helm repo update
  ```
 - Install the load balancer controller on your eks cluster 
  
  ```bash
	helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
	  -n kube-system \
	  --set clusterName=three-tier-cluster \
	  --set serviceAccount.create=false \
	  --set serviceAccount.name=aws-load-balancer-controller
  ```
 - verify alb controller pods in kube-system 
  ```bash
  kubectl get deployment -n kube-system aws-load-balancer-controller 
  ```
 > Note: Now your Load balancer is working let’s setup Ingress for internal routing 
5. SETUP INGRESS
 - setup ingress for internal routing 
 - Loacte the full_stack_lb.yaml file 
	```bash
	kubectl apply -f full_stack_lb.yaml 
	kubectl get ing -n workshop 
	```
 - output will be 
   ```bash
   NAME     CLASS   HOSTS   ADDRESS                                                                 PORTS   AGE
   mainlb   alb     *       my-alb-dns-name.us-east-1.elb.amazonaws.com   80      2m
   ```

 > Note: Now paste DNS address in Browser 

 > Note: Application setup is completed
6. Desroy Everything
 
 1. Uninstall AWS Load Balancer Controller
  ```bash
  helm uninstall aws-load-balancer-controller -n kube-system
  ```
  - Verify that the ALB controller has been removed:
  ```bash
  kubectl get pods -n kube-system | grep aws-load-balancer-controller
  ```
 2. Delete Ingress & Services
  ```bash
  kubectl delete ingress mainlb -n workshop
  kubectl delete service frontend -n workshop
  kubectl delete service backend -n workshop
  kubectl delete deployment frontend -n workshop
  kubectl delete deployment backend -n workshop
  kubectl delete namespace workshop
  ```
  - Verify everything is deleted:
  ```bash
  kubectl get all -n workshop
  ```
 3. Delete IAM Service Account
  ```bash
  eksctl delete iamserviceaccount \
    --cluster=my-cluster \
    --namespace=kube-system \
    --name=aws-load-balancer-controller \
    --region=us-east-1
  ```
  - Verify deletion
  ```bash
  eksctl get iamserviceaccount --cluster=my-cluster
  ```
 4. Detach and Delete IAM Policy
  ```bash
  aws iam list-entities-for-policy --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy
  ```
  - If it is attached, detach it from the IAM role:
  ```bash
  aws iam detach-role-policy \
    --role-name AmazonEKSLoadBalancerControllerRole \
    --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy
   ```
   - Now delete the IAM policy:
   ```bash
   aws iam delete-policy --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy
   ```
   > Replace <ACCOUNT_ID> with your AWS account ID.
 5. Delete IAM Role for ALB Controller
 	```bash
 	aws iam delete-role --role-name AmazonEKSLoadBalancerControllerRole
 	```
 	- Verify deletion:
 	```bash
 	aws iam list-roles | grep AmazonEKSLoadBalancerControllerRole
 	```
 6. Disassociate IAM OIDC Provider
  - First, get the OIDC Provider URL:
  ```bash
  aws eks describe-cluster --name my-cluster --query "cluster.identity.oidc.issuer" --region us-east-1
  ```
  - It should return something like:
  ```bash
  "https://oidc.eks.us-east-1.amazonaws.com/id/XXXXXXX"
  ```
  - Extract the ID at the end of the URL and delete the OIDC provider:
  ```bash
  aws iam list-open-id-connect-providers
  ```
  - Find the ARN of your OIDC provider and delete it:
  ```bash
  aws iam delete-open-id-connect-provider --open-id-connect-provider-arn arn:aws:iam::<ACCOUNT_ID>:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/XXXXXXX
  ```
 7. Delete the EKS Cluster
  ```bash
  eksctl delete cluster --name=my-cluster --region=us-east-1
  ```
  





	- Run below command on every directory 
	```bash
	kubectl delete -f . 
	```
	- Delete the cluster and the stack of your cloud formation 
	```bash
	eksctl delete cluster --name three-tier-cluster --region us-east-1 
	aws cloudformation delete-stack --stack-name eksctl-three-tier-cluster-cluster 
	```
	- verify in the cloud formation cosole for eks stacks deleted or not 


  
-----------------------------------------------------
## ALB and Ingress

  
 
 6. Verify the Ingress and ALB
   - Check Ingress
   ```bash
   kubectl get ingress -n workshop
   ```
   - output will be 
   ```bash
   NAME     CLASS   HOSTS   ADDRESS                                                                 PORTS   AGE
   mainlb   alb     *       my-alb-dns-name.us-east-1.elb.amazonaws.com   80      2m
   ```


## **6️⃣ Troubleshooting Common Issues**
| Issue | Solution |
|--------|------------|
| ALB is not created | Check the AWS Load Balancer Controller logs: `kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller` |
| No DNS in `kubectl get ingress` | Ensure the IAM role has the correct permissions (`AWSLoadBalancerControllerIAMPolicy` is attached) |
| "403 Access Denied" in `kubectl describe ingress` | Check IAM role permissions and OIDC provider |
| ALB shows unhealthy targets | Ensure the backend services are running (`kubectl get pods -n workshop`) |

---

1. Install AWS Load Balancer Controller 
2. Deploy frontend and backend services
3. Create an ALB Ingress resource 
4. Verify ALB & DNS are created 

--- 

