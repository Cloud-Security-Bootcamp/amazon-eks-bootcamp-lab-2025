# Lab Guide 1 - Building a Amazon EKS Lab with a Vulnerable Java Pod

## Objective
In this lab, you’ll create an Amazon EKS cluster, deploy a pod running a vulnerable Java application (`ghcr.io/datadog/vulnerable-java-application`), expose it with a load balancer, and clean up. This introduces basic Kubernetes concepts in EKS.

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
     - Click **Create new role** (or select an existing one).
     - In the IAM console, choose **EKS**, attach `AmazonEKSClusterPolicy`, name it `eks-cluster-role`, and create it.
     - Back in EKS, select `eks-cluster-role` from the dropdown.
   - **Networking:**
     - Use the default VPC or pick an existing one.
     - Select at least 2 subnets in different Availability Zones (e.g., `us-east-1a`, `us-east-1b`).
   - Click **Next**, review, and click **Create**. (Takes ~10-15 minutes.)

4. **Verify Cluster Creation**
   - Return to the EKS dashboard.
   - Click `eks-lab-cluster` and confirm its status is `Active`.

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
   - **Instance type:** Choose `t3.medium` (or adjust based on needs).
   - **Desired capacity:** Set to `2` nodes.
   - **Subnets:** Use the same subnets as the cluster.
   - Click **Next**, review, and click **Create**. (Takes ~5-10 minutes.)

3. **Verify Node Group**
   - In the **Compute** tab, confirm `eks-node-group` is `Active` with 2 nodes.

---

## Step 3: Connect to the Cluster Using CloudShell

1. **Open AWS CloudShell**
   - In the AWS Console, click the CloudShell icon (terminal symbol) in the top-right corner.

2. **Update AWS CLI Config**
   - Connect to the cluster:
     ```
     aws eks update-kubeconfig --region us-east-1 --name eks-lab-cluster
     ```
     - Replace `us-east-1` with your region if different.
   - Verify connection:
     ```
     kubectl get nodes
     ```
     - You should see 2 nodes (e.g., `ip-192-168-x-x.ec2.internal`).

---

## Step 4: Deploy the Vulnerable Java Pod

1. **Create the YAML File**
   - In CloudShell, create `vulnerable-app.yaml`:
     ```
     cat <<EOF > vulnerable-app.yaml
     apiVersion: v1
     kind: Pod
     metadata:
       name: vulnerable-java-app
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
     kubectl apply -f vulnerable-app.yaml
     ```

3. **Verify Deployment**
   - Check pods:
     ```
     kubectl get pods
     ```
     - Ensure `vulnerable-java-app` is `Running`.
   - Check service:
     ```
     kubectl get services
     ```
     - Wait for `vulnerable-java-service` to show an `EXTERNAL-IP` (e.g., `a12b34...elb.amazonaws.com`).

4. **Test Access**
   - Copy the `EXTERNAL-IP` and test:
     ```
     curl http://<EXTERNAL-IP>
     ```
     - If successful, you’ll see the app’s response.

---

## Step 5: Clean Up

1. **Delete Resources**
   - Run:
     ```
     kubectl delete -f vulnerable-app.yaml
     ```

2. **(Optional) Delete Cluster and Node Group**
   - In the AWS Console:
     - Go to EKS > `eks-lab-cluster` > **Compute** > Select `eks-node-group` > **Delete**.
     - Then, go to **Clusters** > Select `eks-lab-cluster` > **Delete**.
   - Cleanup takes ~10-15 minutes.

---

## In Case You Need More Help

If the website isn’t loading (e.g., `curl` fails), try these troubleshooting steps:

1. **Check Pod Status**
   - Run:
     ```
     kubectl get pods
     ```
     - If not `Running`, check events:
       ```
       kubectl describe pod vulnerable-java-app
       ```
     - Look for errors like `ImagePullBackOff` (image issue) or `CrashLoopBackOff` (app crashing).

2. **Check Logs**
   - Run:
     ```
     kubectl logs vulnerable-java-app
     ```
     - Look for startup messages (e.g., port used) or errors. If the app uses a port other than 8080 (e.g., 3000), update `vulnerable-app.yaml`:
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
         kubectl delete -f vulnerable-app.yaml
         kubectl apply -f vulnerable-app.yaml
         ```

3. **Test Service Internally**
   - Launch a debug pod:
     ```
     kubectl run -it --rm debug --image=busybox -- sh
     ```
     - Inside, run:
       ```
       wget -O- http://vulnerable-java-service:80
       ```
     - If "Connection refused," the app isn’t listening on 8080. Recheck logs.
     - If pod creation fails (e.g., "AlreadyExists"), delete it:
       ```
       kubectl delete pod debug --force --grace-period=0
       ```

4. **Check Load Balancer**
   - In AWS Console > EC2 > Load Balancers, find the LB for `vulnerable-java-service`.
   - Ensure its security group allows inbound `HTTP` on port 80 (add rule if missing).

---

## Key Takeaways
- You’ve created an EKS cluster and deployed a vulnerable app with a load balancer.
- This lab introduces basic pod and service creation in Kubernetes.
