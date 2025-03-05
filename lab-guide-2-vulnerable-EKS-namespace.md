# Lab Guide 2 - Building Development & Production Environments in Amazon EKS with a Vulnerable Java App in Dev & NGINX in Prod

## Objective
In this lab, you’ll create an EKS cluster with `dev`, `test`, and `prod` namespaces. `dev` will host a vulnerable Java app with replicas, `prod` will run NGINX with replicas, and `test` will remain empty. Both `dev` and `prod` will use load balancers to demonstrate resilience and scaling differences.

## Prerequisites
- An AWS account with permissions to create EKS clusters, IAM roles, and VPC resources.
- Basic familiarity with the AWS Console and terminal commands.

## Video Resources
You can watch these YouTube videos for additional guidance:
- [Video Walkthrough: Create Your FREE AWS Account](https://www.youtube.com/watch?v=AoQV-X99haQ)
- [Video Walkthrough: Amazon EKS Cluster Lab Creation](https://www.youtube.com/watch?v=1u3YF9i8PzA)
- [Video Walkthrough: Amazon EKS Cluster Lab CleanUp](https://www.youtube.com/watch?v=iLHybxS_2LU)

---

## Step 1: Create an EKS Cluster Using the AWS Console

1. **Log in to the AWS Management Console**
   - Open your browser and go to `https://console.aws.amazon.com/`.

2. **Navigate to EKS**
   - In the top search bar, type "EKS" and select **Elastic Kubernetes Service**.

3. **Create a Cluster**
   - Click **Create cluster**.
   - **Cluster name:** Enter `eks-lab-cluster`.
   - **Kubernetes version:** Use the default (e.g., 1.29).
   - **Cluster service role:**
     - Click **Create new role**.
     - In IAM, choose **EKS**, attach `AmazonEKSClusterPolicy`, name it `eks-cluster-role`, and create it.
     - Select `eks-cluster-role` in the dropdown.
   - **Networking:**
     - Use the default VPC or select an existing one.
     - Choose at least 2 subnets in different Availability Zones (e.g., `us-east-1a`, `us-east-1b`).
   - Click **Next**, review, and click **Create**. (Takes ~10-15 minutes.)

4. **Verify Cluster Creation**
   - In the EKS dashboard, click `eks-lab-cluster` and confirm its status is `Active`.

---

## Step 2: Add a Node Group Using the AWS Console

1. **Select the Cluster**
   - In the EKS dashboard, click `eks-lab-cluster`.

2. **Add Node Group**
   - Go to the **Compute** tab and click **Add node group**.
   - **Name:** Enter `eks-node-group`.
   - **Node IAM role:**
     - Click **Create new role**.
     - In IAM, select **EC2**, attach `AmazonEKSWorkerNodePolicy`, `AmazonEC2ContainerRegistryReadOnly`, and `AmazonEKS_CNI_Policy`.
     - Name it `eks-node-role` and create it.
     - Select `eks-node-role` in the dropdown.
   - **Instance type:** Choose `t3.medium`.
   - **Desired capacity:** Set to `2` nodes.
   - **Subnets:** Use the same subnets as the cluster.
   - Click **Next**, review, and click **Create**. (Takes ~5-10 minutes.)

3. **Verify Node Group**
   - In the **Compute** tab, ensure `eks-node-group` is `Active` with 2 nodes.

---

## Step 3: Connect to the Cluster Using CloudShell

1. **Open AWS CloudShell**
   - In the AWS Console, click the CloudShell icon in the top-right corner.

2. **Update AWS CLI Config**
   - Connect to the cluster:
     ```
     aws eks update-kubeconfig --region us-east-1 --name eks-lab-cluster
     ```
     - Replace `us-east-1` with your region if different.
   - Verify:
     ```
     kubectl get nodes
     ```
     - Expect 2 nodes listed.

---

## Step 4: Create Namespaces

1. **Create Namespaces**
   - Run:
     ```
     kubectl create namespace dev
     kubectl create namespace test
     kubectl create namespace prod
     ```

2. **Verify**
   - Run:
     ```
     kubectl get namespaces
     ```
     - See `dev`, `test`, and `prod` listed.

---

## Step 5: Deploy Vulnerable Java App in `dev` with Deployment

1. **Create the YAML File**
   - In CloudShell:
     ```
     cat <<EOF > dev-vulnerable-app-deployment.yaml
     apiVersion: apps/v1
     kind: Deployment
     metadata:
       name: vulnerable-java-app
       namespace: dev
     spec:
       replicas: 2
       selector:
         matchLabels:
           app: vulnerable-java
       template:
         metadata:
           labels:
             app: vulnerable-java
         spec:
           containers:
           - name: vulnerable-java-container
             image: ghcr.io/datadog/vulnerable-java-application:latest
             ports:
             - containerPort: 8080
     ---
     apiVersion: v1
     kind: Service
     metadata:
       name: vulnerable-java-service
       namespace: dev
     spec:
       selector:
         app: vulnerable-java
       ports:
       - protocol: TCP
         port: 80
         targetPort: 8080
       type: LoadBalancer
     EOF
     ```

2. **Apply the Configuration**
   - Run:
     ```
     kubectl apply -f dev-vulnerable-app-deployment.yaml
     ```

3. **Verify**
   - Check deployment and pods:
     ```
     kubectl get deployments,pods -n dev
     ```
     - Expect 2 `vulnerable-java-app` pods running.
   - Check service:
     ```
     kubectl get services -n dev
     ```
     - Note the `EXTERNAL-IP`.

4. **Test Access**
   - Run:
     ```
     curl http://<dev-EXTERNAL-IP>
     ```

---

## Step 6: Deploy NGINX in `prod` with Deployment

1. **Create the YAML File**
   - In CloudShell:
     ```
     cat <<EOF > prod-nginx-deployment.yaml
     apiVersion: apps/v1
     kind: Deployment
     metadata:
       name: nginx-app
       namespace: prod
     spec:
       replicas: 2
       selector:
         matchLabels:
           app: nginx
       template:
         metadata:
           labels:
             app: nginx
         spec:
           containers:
           - name: nginx-container
             image: nginx:latest
             ports:
             - containerPort: 80
     ---
     apiVersion: v1
     kind: Service
     metadata:
       name: nginx-service
       namespace: prod
     spec:
       selector:
         app: nginx
       ports:
       - protocol: TCP
         port: 80
         targetPort: 80
       type: LoadBalancer
     EOF
     ```

2. **Apply the Configuration**
   - Run:
     ```
     kubectl apply -f prod-nginx-deployment.yaml
     ```

3. **Verify**
   - Check deployment and pods:
     ```
     kubectl get deployments,pods -n prod
     ```
     - Expect 2 `nginx-app` pods running.
   - Check service:
     ```
     kubectl get services -n prod
     ```
     - Note the `EXTERNAL-IP`.

4. **Test Access**
   - Run:
     ```
     curl http://<prod-EXTERNAL-IP>
     ```
     - Expect the NGINX welcome page.

---

## Step 7: Explore Scaling and Resilience

1. **Scale `prod`**
   - Increase NGINX replicas:
     ```
     kubectl scale deployment nginx-app -n prod --replicas=3
     kubectl get pods -n prod
     ```

2. **Test Resilience**
   - Delete a pod in `prod`:
     ```
     kubectl delete pod <nginx-pod-name> -n prod
     ```
     - Watch it recreate:
       ```
       kubectl get pods -n prod
       ```

---

## Step 8: Clean Up

1. **Delete Resources**
   - Run:
     ```
     kubectl delete -f dev-vulnerable-app-deployment.yaml
     kubectl delete -f prod-nginx-deployment.yaml
     kubectl delete namespace dev test prod
     ```

2. **(Optional) Delete Cluster and Node Group**
   - In the AWS Console:
     - Go to EKS > `eks-lab-cluster` > **Compute** > Select `eks-node-group` > **Delete**.
     - Then, go to **Clusters** > Select `eks-lab-cluster` > **Delete**.
   - Cleanup takes ~10-15 minutes.

---

## In Case You Need More Help

If an app isn’t accessible (e.g., `curl` fails), troubleshoot as follows:

1. **Check Pod Status**
   - Run (for `dev` or `prod`):
     ```
     kubectl get pods -n <namespace>
     ```
     - If not `Running`, check events:
       ```
       kubectl describe pod <pod-name> -n <namespace>
       ```

2. **Check Logs**
   - Run:
     ```
     kubectl logs <pod-name> -n <namespace>
     ```
     - For `dev`, confirm the port (e.g., 8080). If different (e.g., 3000), update YAML:
       ```yaml
       ports:
       - containerPort: 3000
       ---
       ports:
       - protocol: TCP
         port: 80
         targetPort: 3000
       ```
       - Reapply:
         ```
         kubectl delete -f <file>.yaml
         kubectl apply -f <file>.yaml
         ```

3. **Test Service Internally**
   - Launch a debug pod:
     ```
     kubectl run -it --rm debug --image=busybox -- sh
     ```
     - Inside, test:
       ```
       wget -O- http://<service-name>.<namespace>:80
       ```
     - If "Connection refused," check the app’s port. If pod creation fails:
       ```
       kubectl delete pod debug --force --grace-period=0
       ```

4. **Check Load Balancer**
   - In AWS Console > EC2 > Load Balancers, find the LB for the service.
   - Ensure its security group allows inbound `HTTP` on port 80.
   - Check target group health; if `Unhealthy`, verify the app responds on the expected port.

---

## Key Takeaways
- You’ve created isolated namespaces with a vulnerable app in `dev` and NGINX in `prod`.
- Using `Deployment` demonstrates scaling and self-healing, contrasting security implications.
