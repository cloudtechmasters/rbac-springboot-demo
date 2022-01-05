# rbac-springboot-demo
**Assumption** : EKS cluster is ready and helm is installed

# Steps:

1.	Install springboot application
2.	Create rbac-user admin and deploy
3.	MAP user to K8S
4.	Test new user
5.	Create role and binding
6.	Verify role and binding

# 1.Install Springboot application.

**# kubectl apply -f springboot-deployment.yaml** 

  deployment.apps/springboot created

**# kubectl apply -f springboot-service.yaml**

  service/springboot created

**# alias k=kubectl**

**# k get all**

     NAME                             READY   STATUS    RESTARTS   AGE
     pod/springboot-db6684d7b-c8b4f   1/1     Running   0          23s
     pod/springboot-db6684d7b-vz2pz   1/1     Running   0          23s
     pod/springboot-db6684d7b-wd58t   1/1     Running   0          23s

     NAME                 TYPE           CLUSTER-IP       EXTERNAL-IP                                                              PORT(S)           AGE
     service/kubernetes   ClusterIP      10.100.0.1       <none>                                                                   443/TCP           55m
     service/springboot   LoadBalancer   10.100.156.235   a3ec204441a054cbd90a56cb3a19dbe9-434268644.us-east-2.elb.amazonaws.com   33333:31170/TCP   12s

     NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
     deployment.apps/springboot   3/3     3            3           23s

     NAME                                   DESIRED   CURRENT   READY   AGE
     replicaset.apps/springboot-db6684d7b   3         3         3       23s

# 2.Create user admin and develop

**Admin will have only view only permissions, he can't patch the deployments.** 
  
     aws iam create-user --user-name deploy
     aws iam create-access-key --user-name deploy | tee /tmp/create_output.json

**Create credential file for admin user, so that we can switch user easily.**

    cat << EoF > admin_creds.sh
    export AWS_SECRET_ACCESS_KEY=$(jq -r .AccessKey.SecretAccessKey /tmp/create_output.json)
    export AWS_ACCESS_KEY_ID=$(jq -r .AccessKey.AccessKeyId /tmp/create_output.json)
    EoF
  
**Deploy role can do all operations, he can patch the deployments.**
  
     aws iam create-user --user-name deploy
     aws iam create-access-key --user-name deploy | tee /tmp/create_output.json

**Create credential file for deploy user, so that we can switch user easily.**

    cat << EoF > deploy_creds.sh
    export AWS_SECRET_ACCESS_KEY=$(jq -r .AccessKey.SecretAccessKey /tmp/create_output.json)
    export AWS_ACCESS_KEY_ID=$(jq -r .AccessKey.AccessKeyId /tmp/create_output.json)
    EoF

# 3.	MAP user to K8S

**Add the user details to aws-auth.yaml and apply it to k8s cluster, so that we can map users to k8s.**

**kubectl get configmap -n kube-system aws-auth -o yaml | egrep -v "creationTimestamp|resourceVersion|selfLink|uid" | sed '/^ annotations:/,+2 d' > aws-auth.yaml**
  
  
    cat aws-auth.yaml 
    apiVersion: v1
    data:
      mapRoles: |
        - groups:
          - system:bootstrappers
          - system:nodes
          rolearn: arn:aws:iam::556952635478:role/eksctl-eksdemo-nodegroup-eksdemo-NodeInstanceRole-AGRI5C4SXGJC
          username: system:node:{{EC2PrivateDNSName}}
    kind: ConfigMap
    metadata:
      name: aws-auth
      namespace: kube-system
    data:
      mapUsers: |
         - userarn: arn:aws:iam::556952635478:user/admin
           username: admin
         - userarn: arn:aws:iam::556952635478:user/deploy
           username: deploy

# 4.	Test new user

**. admin_creds.sh**

**# aws sts get-caller-identity**

   {
       "Account": "556952635478", 
       "UserId": "AIDAYDLHWZBLM4EAWSZZK", 
       "Arn": "arn:aws:iam::556952635478:user/admin"
   }
   
 
**# k get deploy**

  Error from server (Forbidden): deployments.apps is forbidden: User "admin" cannot list resource "deployments" in API group "apps" in the namespace "default"

**Here we are getting error, as we have not assigned any role with the user. Lets create role and role-binding**

# 5.	Create role and binding

    # unset AWS_SECRET_ACCESS_KEY 
    # unset AWS_ACCESS_KEY_ID 

    # k apply -f rbacuser-role-admin.yaml 
    role.rbac.authorization.k8s.io/admin created
    # k apply -f rbacuser-role-deploy.yaml 
    role.rbac.authorization.k8s.io/deployer created
    # k apply -f rbacuser-role-binding-admin.yaml 
    rolebinding.rbac.authorization.k8s.io/read-pods created
    # k apply -f rbacuser-role-binding-deploy.yaml 
    rolebinding.rbac.authorization.k8s.io/patch-pods created

# 6. Verify role and binding

**Scenario 1 : Now admin user can able to access the deployment.**

**. admin_creds.sh** 

    # k get deploy
    NAME         READY   UP-TO-DATE   AVAILABLE   AGE
    springboot   2/2     2            2           92m

**Scenario 2 : Usgin deploy user we can able to scale the deployment but for admin we are getting error as he dont have permission to scale.**
 
**. deploy_creds.sh**  
 
    # kubectl scale --replicas=2 deployment/springboot
   deployment.apps/springboot scaled
  
**. admin_creds.sh** 

    # kubectl scale --replicas=2 deployment/springboot
    Error from server (Forbidden): deployments.apps "springboot" is forbidden: User "admin" cannot patch resource "deployments/scale" in API group "apps" in the namespace "default"

  **Scenario 3 : Using admin we can not get resplicaset info but using deploy we can get it.**
  
    # k get rs
    Error from server (Forbidden): replicasets.apps is forbidden: User "admin" cannot list resource "replicasets" in API group "apps" in the namespace "default"

**. deploy_creds.sh**

    # k get rs
    NAME                   DESIRED   CURRENT   READY   AGE
    springboot-db6684d7b   2         2         2       60m

**Scenario 4 : Both admin/depoly user having permission to list deployment on default namespace only, if we try to list resources from other ns will get error.**

**. admin_creds.sh**

      # aws sts get-caller-identity 
      {
          "Account": "5569****635478", 
          "UserId": "AIDAYDLHW****EAWSZZK", 
          "Arn": "arn:aws:iam::5569****635478:user/admin"
      }
      
      # alias k=kubectl
      # k get deploy
      NAME         READY   UP-TO-DATE   AVAILABLE   AGE
      springboot   2/2     2            2           4h19m
      # k get deploy -n kube-system
      Error from server (Forbidden): deployments.apps is forbidden: User "admin" cannot list resource "deployments" in API group "apps" in the namespace "kube-system"

**. deploy_creds.sh**

      # k get deploy
      NAME         READY   UP-TO-DATE   AVAILABLE   AGE
      springboot   2/2     2            2           4h19m

      # k get deploy -n kube-system
      Error from server (Forbidden): deployments.apps is forbidden: User "deploy" cannot list resource "deployments" in API group "apps" in the namespace "kube-system"
