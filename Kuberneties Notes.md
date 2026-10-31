# Kubernetes Storage, ConfigMaps, Secrets, Autoscaling, Ingress & AWS Secrets Manager

---

# 1. Persistent Volumes (PV) and Persistent Volume Claims (PVC)

## Manifest for Static EFS PV

**Prerequisites:**
- Create EFS first in the same cluster VPC.
- In the EFS Security Group, allow NFS port `2049` from the Worker Node Security Group.
- Make sure `nfs-utils` (RedHat) or `nfs-common` (Ubuntu/Debian) is available on worker nodes.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: efs-pv

spec:
  capacity:
    storage: 10Gi

  persistentVolumeReclaimPolicy: Retain

  # To create a Static EFS PV, use an empty StorageClass name.
  storageClassName: ""

  mountOptions:
    - nfsvers=4.1

  accessModes:
    - ReadWriteMany

  nfs:
    server: fs-075fd30f2aec4a0a1.efs.ap-south-1.amazonaws.com
    path: "/"
```

## Manifest for EBS PV

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ebs-pv

spec:
  capacity:
    storage: 20Gi

  accessModes:
    - ReadWriteOnce

  persistentVolumeReclaimPolicy: Retain

  awsElasticBlockStore:
    volumeID: vol-0663994ce090fc119
    fsType: ext4
```

## Manifest for Claiming PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-pvc

spec:
  accessModes:
    - ReadWriteMany

  storageClassName: ""

  resources:
    requests:
      storage: 5Gi
```

## Manifest for Claiming PVC with StorageClass (Dynamic Provisioning)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-pvc

spec:
  accessModes:
    - ReadWriteOnce

  storageClassName: gp2

  resources:
    requests:
      storage: 10Gi
```

## Without Dynamic Provisioning

```text
Admin
  |
  v
Create PV manually
  |
  v
Create PVC
  |
  v
Pod
```

## With Dynamic Provisioning

```text
User
  |
  v
Create PVC --> Specify StorageClassName & Requested Storage
  |
  v
StorageClass
  |
  v
CSI Driver (EBS/EFS) --> Install using Helm or GitHub manifests
  |
  v
Create Actual Storage (EBS/EFS)
  |
  v
Create PV
  |
  v
PVC Binds
  |
  v
Pod
```

---

# 2. StorageClass

## StorageClass for EBS

**For creating your own StorageClass, the AWS EBS CSI Driver must be installed and configured.**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc

provisioner: ebs.csi.aws.com

volumeBindingMode: WaitForFirstConsumer

parameters:
  type: gp3
  fsType: ext4

reclaimPolicy: Delete

allowVolumeExpansion: true
```

### Why is WaitForFirstConsumer important for EBS?

It delays volume provisioning until a Pod is created and the scheduler selects a suitable node.

EBS volumes are AZ-specific, so the volume must be created in the same Availability Zone as the selected worker node.

```text
PVC Created
    |
    v
Wait
    |
    v
Pod Created
    |
    v
Scheduler Chooses Suitable Node/AZ
    |
    v
EBS Created in That AZ
    |
    v
Pod Uses EBS
```

## StorageClass for EFS

**Prerequisites generally include:**

- EFS filesystem
- EFS mount targets
- Network connectivity
- Security Group allowing NFS TCP port 2049
- EFS CSI Driver
- Appropriate IAM permissions for the EFS CSI Driver

Then StorageClass can dynamically provision EFS-backed storage.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc

provisioner: efs.csi.aws.com

parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-053f655e0992e8615
  directoryPerms: "700"
  gidRangeStart: "1000"
  gidRangeEnd: "2000"
  basePath: "/dynamic_provisioning"

reclaimPolicy: Delete
```

---

# 3. Mounting PVC in Pod

## Deployment with EBS PVC

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-ebs

spec:
  replicas: 1

  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      labels:
        app: nginx

    spec:
      containers:
        - name: nginx
          image: nginx

          volumeMounts:
            - name: v1
              mountPath: /data

      volumes:
        - name: v1
          persistentVolumeClaim:
            claimName: ebs-pvc
```

## HostPath Volume

```yaml
volumeMounts:
  - name: host-data
    mountPath: /data

volumes:
  - name: host-data
    hostPath:
      path: /data
      type: DirectoryOrCreate
```

## EmptyDir Volume

```yaml
volumeMounts:
  - name: shared-data
    mountPath: /data

volumes:
  - name: shared-data
    emptyDir: {}
```

## EmptyDir with Temporary Storage on RAM

```yaml
volumes:
  - name: cache
    emptyDir:
      medium: Memory
```

---

# 4. StatefulSet with VolumeClaimTemplates

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nginx

spec:
  serviceName: nginx
  replicas: 3

  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      labels:
        app: nginx

    spec:
      containers:
        - name: nginx
          image: nginx

          volumeMounts:
            - name: v1
              mountPath: /data

  volumeClaimTemplates:
    - metadata:
        name: v1

      spec:
        storageClassName: gp2

        accessModes:
          - ReadWriteOnce

        resources:
          requests:
            storage: 1Gi
```

---

# 5. Deployment Manifest with Volume Mount and Resources

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ekart

spec:
  replicas: 2

  selector:
    matchLabels:
      app: ekart

  template:
    metadata:
      labels:
        app: ekart

    spec:
      containers:
        - name: ekart
          image: surajkhanderao/ekart:v1

          ports:
            - containerPort: 8070

          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "1"
              memory: "1Gi"

          startupProbe:
            httpGet:
              path: /
              port: 8070
            periodSeconds: 5
            failureThreshold: 12

          readinessProbe:
            httpGet:
              path: /
              port: 8070
            periodSeconds: 5

          livenessProbe:
            httpGet:
              path: /
              port: 8070
            periodSeconds: 5

          volumeMounts:
            - mountPath: /data
              name: pod-volume

      volumes:
        - name: pod-volume
          hostPath:
            path: /data
            type: DirectoryOrCreate
```

---

# 6. ConfigMap & Secret

## ConfigMap

### Manifest for Passing ENV Variables

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config

data:
  APP_ENV: "production"
  LOG_LEVEL: "INFO"
```

### Passing Environment Variables

You can pass individual environment variables or all variables at once.

#### Single ENV

```yaml
env:
  - name: APP_ENV
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: APP_ENV

  - name: LOG_LEVEL
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: LOG_LEVEL
```

#### Passing All ENV Variables at Once

This will make each ConfigMap key available as one environment variable inside the container.

```yaml
envFrom:
  - configMapRef:
      name: app-config

  - secretRef:
      name: app-secrets
```

`app-secrets` should be the name of the Secret you created.

### Passing a File Inside the Container Using Volume Mount

#### Create Files Using ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config

data:
  application.properties: |
    server.port=8080
    logging.level.root=INFO
```

#### Mount That File as a Volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-demo

spec:
  containers:
    - name: app
      image: nginx

      volumeMounts:
        - name: config-volume
          mountPath: /etc/config

  volumes:
    - name: config-volume
      configMap:
        name: app-config

    - name: secret-volume
      secret:
        secretName: db-secret
```

### Volumes + Items (If One File Contains Multiple Keys)

```yaml
volumes:
  - name: config-volume
    configMap:
      name: app-config
      items:
        - key: application.properties
          path: application.properties
```

## Secret Manifest

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret

type: Opaque  # For generic credentials like UserID/Password

stringData:
  DB_USERNAME: "admin"
  DB_PASSWORD: "mypassword"
```

---

# 7. HPA (Horizontal Pod Autoscaler)

HPA automatically changes the number of Pod replicas based on observed metrics such as CPU, memory, custom metrics, or external metrics.

For CPU and memory resource metrics, HPA obtains the metrics through the Kubernetes Metrics API, commonly provided by Metrics Server.

HPA compares the current metric against the configured target and updates the target workload's replica count within the configured minimum and maximum.

### Difference Between HPA, VPA and Cluster Autoscaler

- **HPA:** Scales Pods horizontally by increasing or decreasing the number of replicas.
- **VPA:** Adjusts Pod resource requests vertically.
- **Cluster Autoscaler:** Changes the number of worker nodes.

### How HPA Works

```text
              Kubelet
                 |
                 | Resource Usage
                 v
           Metrics Server
                 |
                 v
             Metrics API
                 |
                 v
                HPA
                 |
           Compares with
               Target
                 |
                 v
             Deployment
                 |
                 v
             ReplicaSet
                 |
                 v
          More / Fewer Pods
```

## HPA Manifest

### Install Metrics Server First

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### Command

```bash
kubectl autoscale deployment nginx-deployment \
  --cpu-percent=70 \
  --min=2 \
  --max=10
```

### Manifest

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler

metadata:
  name: nginx-hpa

spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deployment

  minReplicas: 2
  maxReplicas: 10

  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70

    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 70

  # To avoid flapping
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0

    scaleDown:
      stabilizationWindowSeconds: 300
```

## HPA with AverageValue

```yaml
metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: AverageValue
        averageValue: 500m
```

---

# 8. VPA (Vertical Pod Autoscaler)

## How It Works

```text
VPA Recommender
      |
      v
Observes Usage
      |
      v
Calculates Resource Recommendation
      |
      v
VPA Updater
      |
      v
May Evict Pods When Needed
      |
      v
VPA Admission Controller
      |
      v
New Pod Receives Updated Resources
```

## Installation

```bash
git clone https://github.com/kubernetes/autoscaler.git

cd autoscaler/vertical-pod-autoscaler

./hack/vpa-up.sh
```

## VPA Manifest

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler

metadata:
  name: nginx-vpa

spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deployment

  updatePolicy:
    updateMode: "Auto"

  resourcePolicy:
    containerPolicies:
      - containerName: nginx

        minAllowed:
          cpu: 100m
          memory: 128Mi

        maxAllowed:
          cpu: 1
          memory: 1Gi
```

---

# 9. Ingress

## AWS Load Balancer Controller

### First Install Helm

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4

chmod 700 get_helm.sh

./get_helm.sh
```

### Attach Role to Worker Nodes

Attach the required IAM permissions to the appropriate IAM role used by the AWS Load Balancer Controller.

### Install EKS Charts Repository

```bash
helm repo add eks https://aws.github.io/eks-charts

helm repo update
```

### Install AWS Load Balancer Controller

```bash
helm install aws-load-balancer-controller \
  eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=mycluster.example.com
```

Now you can create Ingress resources using AWS Load Balancer Controller.

---

## NGINX Ingress Controller

### Install Helm

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3

chmod 700 get_helm.sh

./get_helm.sh
```

### Install NGINX Ingress Controller

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

helm repo update

helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace
```

### Alternatively, Directly Apply NGINX Controller Manifest

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/cloud/deploy.yaml
```

## Manifest for NGINX (Regex-Based Path Routing)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-web-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$2

spec:
  ingressClassName: nginx

  rules:
    - http:
        paths:
          - path: /nginx(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: nginx-service
                port:
                  number: 80

          - path: /httpd(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: httpd-service
                port:
                  number: 80

          - path: /(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: default-service
                port:
                  number: 80
```

## Simple NGINX Ingress Manifest (Prefix-Based Routing)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-web-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/rewrite-target: /

spec:
  ingressClassName: nginx

  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: ekart-svc
                port:
                  number: 80
```

## Ingress with HTTPS

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-web-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /

spec:
  ingressClassName: nginx

  tls:
    - hosts:
        - www.surajdevops.cv
      secretName: bankapp-tls

  rules:
    - host: www.surajdevops.cv
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: ekart-svc
                port:
                  number: 80
```

## ClusterIssuer Manifest

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod

spec:
  acme:
    email: npfhfjamvryncglqjg@vtmpj.net
    server: https://acme-v02.api.letsencrypt.org/directory

    privateKeySecretRef:
      name: letsencrypt-prod

    solvers:
      - http01:
          ingress:
            ingressClassName: nginx
```

## Important Notes

- `pathType` tells Kubernetes how the path should be interpreted.
- `use-regex: "true"` tells the NGINX Ingress Controller to use regex matching.
- `rewrite-target: /$2` changes the URL before forwarding the request.
- For regex-based paths with the NGINX Ingress Controller, `ImplementationSpecific` is the appropriate choice.

### Useful Commands

```bash
kubectl get pods -n ingress-nginx

kubectl get svc -n ingress-nginx

kubectl get ing
```

---

# 10. AWS Secrets Manager for Storing and Managing DB/App Secrets

## Flow
```
AWS Secrets Manager: stores credentials.

EKS Pod Identity: provides secure AWS authentication.

External Secrets Operator (ESO): retrieves credentials from AWS.

Kubernetes Secret: makes credentials available to your application.

Deployment: consumes credentials using envFrom.

```

## What to Do

1. Create MySQL credentials in AWS Secrets Manager.
2. Create an IAM policy allowing access to that particular AWS Secret.
3. Create an IAM Role with EKS Pod Identity as the trusted entity.
4. Attach the restricted-permission IAM policy to that role.
5. Create a namespace and install External Secrets Operator (ESO).
6. Use the ESO ServiceAccount.
7. Install the EKS Pod Identity Agent add-on.
8. Create a Pod Identity Association between the IAM Role and ESO ServiceAccount.
9. Create SecretStore manifest.
10. Create ExternalSecret manifest.
11. Verify the automatically generated Kubernetes Secret.
12. Connect BankApp Deployment using `envFrom`.

## Custom Policy with Restricted Access for IAM Role

**Policy Name: ESOSecretsManagerPolicy**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "YOUR_SECRET_ARN"
    }
  ]
}
```

---

## Install External Secrets Operator in EKS

### Add External Secrets Operator Repository

```bash
helm repo add external-secrets https://charts.external-secrets.io

helm repo update
```

### Install ESO

```bash
kubectl create namespace external-secrets

helm install external-secrets \
  external-secrets/external-secrets \
  -n external-secrets
```

## How ESO Communicates with AWS Secrets Manager

```text
EKS Cluster: sk-cluster
          |
          v
Namespace: external-secrets
          |
          v
Service Account: external-secrets
          |
          v
IAM Role: ESOSecretsManagerRole
          |
          v
IAM Policy: ESOSecretsManagerPolicy
          |
          v
AWS Secrets Manager
```

---

## Install Pod Identity Agent

```bash
aws eks create-addon \
  --cluster-name sk-cluster \
  --addon-name eks-pod-identity-agent \
  --region eu-west-1
```

**Note:** The ServiceAccount is created by default in the ESO Helm deployment. No need to create it separately.

## Attach Role to That ServiceAccount

```bash
aws eks create-pod-identity-association \
  --cluster-name sk-cluster \
  --namespace external-secrets \
  --service-account external-secrets \
  --role-arn YOUR_ROLE_ARN \
  --region eu-west-1
```

### Verify

```bash
aws eks list-pod-identity-associations \
  --cluster-name sk-cluster \
  --region eu-west-1
```

### Helm Creates the Following Resources for ESO

```text
Namespace: external-secrets

ServiceAccount: external-secrets

Deployment: external-secrets
```

---

## Create SecretStore

Now we need to tell ESO where our external secrets are stored. For that, we need to create a SecretStore.

### What Does SecretStore Do?

It tells ESO:

- Which provider to connect to: AWS.
- Which service to use: Secrets Manager.
- Which AWS region to access: eu-west-1.

### Create secretstore.yaml

```yaml
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: aws-secret-store
  namespace: webapps

spec:
  provider:
    aws:
      service: SecretsManager
      region: eu-west-1
```

### Verify

```bash
kubectl get secretstore -n webapps
```

---

## Create ExternalSecret

It tells ESO exactly which credentials to retrieve and what Kubernetes Secret to create.

### externalsecret.yaml

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: bankapp-external-secret
  namespace: webapps

spec:
  refreshInterval: 1h

  secretStoreRef:
    name: aws-secret-store
    kind: SecretStore

  target:
    name: bankapp-db-secret
    creationPolicy: Owner

  data:
    - secretKey: DB_USER
      remoteRef:
        key: bankapp/mysql
        property: username

    - secretKey: DB_PASSWORD
      remoteRef:
        key: bankapp/mysql
        property: password
```

### What Does It Do?

It communicates with SecretStore and retrieves the secrets from AWS Secrets Manager.

It then creates a Kubernetes Secret (`bankapp-db-secret`) and stores all credentials specified in the `data` section.

This automatically creates a Kubernetes Secret for you and stores the database username and password, which you can call inside your application YAML to connect to the database.

## Connect BankApp Deployment Using envFrom

```yaml
spec:
  containers:
    - name: bankapp
      image: adijaiswal/bankapp:v6

      envFrom:
        - secretRef:
            name: bankapp-db-secret
```

### Verify Environment Variable

```bash
kubectl exec -n webapps POD_NAME -- printenv DB_USER
```

## Important Notes

### Secret Rotation

- ESO refreshes the Kubernetes Secret periodically.
- However, environment variables inside an already-running container do not automatically change.
- You need to restart the Pod after a credential update, or implement an automated rollout mechanism.

### Security

ESO synchronizes credentials into Kubernetes Secrets, so Kubernetes RBAC and encryption at rest still matter.

## Verify the Complete Integration

```bash
kubectl get secretstore -n webapps

kubectl get externalsecret -n webapps

kubectl get secret bankapp-db-secret -n webapps
```

---

# End of Kubernetes Notes
