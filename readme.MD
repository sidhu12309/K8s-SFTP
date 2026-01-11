
Below is a **minimal, clean Helm chart** for your **StatefulSet + EBS + HTTPS (ALB) file server on EKS**.

This is **production-correct**, parameterized, and easy to maintain.

---

# 📦 Helm Chart Structure

```
file-server/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── storageclass.yaml
    ├── statefulset.yaml
    ├── service.yaml
    └── ingress.yaml
```

---

# 1️⃣ `Chart.yaml`

```yaml
apiVersion: v2
name: file-server
description: EKS File Server using StatefulSet + EBS + HTTPS
type: application
version: 0.1.0
appVersion: "1.0"
```

---

# 2️⃣ `values.yaml`

```yaml
replicaCount: 1

image:
  repository: drakkan/sftpgo
  tag: latest
  pullPolicy: IfNotPresent

service:
  port: 80
  targetPort: 8080

storage:
  size: 50Gi
  storageClass: ebs-gp3

ingress:
  enabled: true
  className: alb
  host: file-server.example.com
  certificateArn: arn:aws:acm:us-east-1:ACCOUNT_ID:certificate/CERT_ID
  scheme: internet-facing

securityContext:
  fsGroup: 1000
```

---

# 3️⃣ `templates/storageclass.yaml`

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: {{ .Values.storage.storageClass }}
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3
  fsType: ext4
```

---

# 4️⃣ `templates/statefulset.yaml`

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: {{ .Release.Name }}
spec:
  serviceName: {{ .Release.Name }}
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      securityContext:
        fsGroup: {{ .Values.securityContext.fsGroup }}
      containers:
      - name: sftpgo
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - containerPort: {{ .Values.service.targetPort }}
        env:
        - name: SFTPGO_HTTPD__BINDINGS__0__PORT
          value: "{{ .Values.service.targetPort }}"
        - name: SFTPGO_DATA_PROVIDER__DRIVER
          value: "local"
        - name: SFTPGO_DATA_PROVIDER__LOCAL__BASE_PATH
          value: "/srv/sftpgo"
        volumeMounts:
        - name: data
          mountPath: /srv/sftpgo
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes:
        - ReadWriteOnce
      storageClassName: {{ .Values.storage.storageClass }}
      resources:
        requests:
          storage: {{ .Values.storage.size }}
```

📌 This creates:

```
PVC: data-<release-name>-0
EBS volume attached to pod
```

---

# 5️⃣ `templates/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}
spec:
  selector:
    app: {{ .Release.Name }}
  ports:
  - port: {{ .Values.service.port }}
    targetPort: {{ .Values.service.targetPort }}
```

---

# 6️⃣ `templates/ingress.yaml`

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Release.Name }}
  annotations:
    kubernetes.io/ingress.class: {{ .Values.ingress.className }}
    alb.ingress.kubernetes.io/scheme: {{ .Values.ingress.scheme }}
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/certificate-arn: {{ .Values.ingress.certificateArn }}
spec:
  rules:
  - host: {{ .Values.ingress.host }}
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: {{ .Release.Name }}
            port:
              number: {{ .Values.service.port }}
{{- end }}
```

---

# 🚀 Install the Chart

```bash
helm install file-server ./file-server
```

Upgrade later:

```bash
helm upgrade file-server ./file-server
```

---

# 🔐 Access

* URL:

```
https://file-server.example.com
```

* Login via **SFTPGo Web UI**
* Users stored on **EBS volume**
* Each user isolated:

```
/srv/sftpgo/<username>
```

---

#  Important Reminders (EBS Reality)

* `replicaCount` **must remain 1**
* No HA
* Single AZ
* Use **snapshots** for backup

---

#  TL;DR

This Helm chart gives you:

*  StatefulSet (correct for EBS)
*  Dynamic EBS provisioning
*  HTTPS (443) via ALB + ACM
*  User/password file access
*  Clean Helm templating


