# k8s_wp_efs_rds_ssl_acm
kubernetes WP deployment with efs, ssl acm 

#### setup in k8s 

### wiht new vpc
eksctl create cluster \  --name demo-cluster \  --region us-east-1 \  --nodegroup-name my-node-group \  --node-type t3.small \  --nodes 1 \  --nodes-min 1 \  --nodes-max 2 \  --managed

### with existing VPC 
eksctl create cluster \
  --name my-cluster \
  --region us-east-1 \
  --subnet-ids subnet-1cb73310,subnet-ed5fc488,subnet-6713e14c,subnet-52c3100b \
  --nodegroup-name my-node \
  --node-type t3.small \
  --nodes 1 \
  --nodes-min 1 \
  --nodes-max 2 \
  --managed

-----------------

Install Helm

curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh


### Create AWS load balancer controller via Helm
Step 1: Create IAM Role using eksctl

curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.2/docs/install/iam_policy.json

aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json

eksctl create iamserviceaccount \
  --cluster=my-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::659328422194:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve

Step 2: Install AWS Load Balancer Controller

helm repo add eks https://aws.github.io/eks-charts
helm repo update eks
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller 
  
Step 3: Verify that the controller is installed

kubectl get deployment -n kube-system aws-load-balancer-controller 


### ingress file 

#commnet here we have mentioned the subnet manually it may be due to i have created the eks with manual subnet 


apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wordpress-efs
    #  namespace: myapplications-ns
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
      #    alb.ingress.kubernetes.io/group.name: frontend
      #alb.ingress.kubernetes.io/group.name: my-alb-group  #Use this to share ALB among multiple ingresses. #CostEffective
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:659328422194:certificate/44ef16c5-9e9f-4e74-9e8a-c56da28ee9f9
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
     # Action to redirect HTTP traffic to HTTPS
     #    alb.ingress.kubernetes.io/actions.ssl-redirect: "443"
    alb.ingress.kubernetes.io/actions.ssl-redirect: '{"Type": "redirect", "RedirectConfig": { "Protocol": "HTTPS", "Port": "443", "StatusCode": "HTTP_301"}}'

    alb.ingress.kubernetes.io/subnets: subnet-1cb73310,subnet-ed5fc488,subnet-6713e14c,subnet-52c3100b
  labels:
    name: wordpress-efs
spec:
  ingressClassName: alb
  rules:
    - host: test.nirajvara.in
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: ssl-redirect
              port:
                name: use-annotation
        - path: /
          pathType: Prefix
          backend:
            service:
              name: wordpress-efs
              port:
                number: 80
