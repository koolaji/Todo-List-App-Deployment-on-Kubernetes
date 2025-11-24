# تسک: تبدیل مانیفست‌های کوبرنتیس برنامه Todo به چارت Helm

## هدف
هدف از این تسک، تبدیل مانیفست‌های کوبرنتیس موجود برای برنامه Todo به یک چارت Helm است تا مدیریت و استقرار برنامه راحت‌تر شود.

## پیش‌نیازها
- نصب Helm روی سیستم محلی
- آشنایی با مفاهیم Helm و ساختار چارت‌ها
- دسترسی به مخزن کد برنامه Todo با مانیفست‌های کوبرنتیس

## گام‌ها

### 1. ایجاد ساختار اولیه چارت Helm
```bash
helm create todo-app
```
ساختار پوشه‌های زیر ایجاد خواهد شد:
```
todo-app/
├── .helmignore
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── NOTES.txt
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   └── ...
└── charts/
```

### 2. پاکسازی فایل‌های اولیه
فایل‌های پیش‌فرض موجود در پوشه `templates/` را حذف کنید و فایل `values.yaml` را خالی کنید.

### 3. تعریف فایل Chart.yaml
اطلاعات چارت را در فایل `Chart.yaml` تعریف کنید:
```yaml
apiVersion: v2
name: todo-app
description: A Helm chart for Todo List application with React, Flask, and MySQL
type: application
version: 0.1.0
appVersion: "1.0.0"
```

### 4. تعریف فایل values.yaml
پارامترهای قابل تنظیم را در `values.yaml` تعریف کنید:

```yaml
# پیکربندی عمومی
nameOverride: ""
fullnameOverride: ""

# پیکربندی MySQL
mysql:
  image: mysql:latest
  imagePullPolicy: Never
  databaseName: todo_database
  rootPassword: changeme
  persistence:
    enabled: true
    size: 1Gi
  service:
    port: 3306

# پیکربندی بک‌اند
backend:
  image: todo-backend:latest
  imagePullPolicy: Never
  replicaCount: 1
  service:
    type: NodePort
    port: 5000
  ingress:
    enabled: true
    host: "to-do-app.local"
    path: "/tasks"
    pathType: "Prefix"

# پیکربندی فرانت‌اند
frontend:
  image: todo-frontend-react:latest
  imagePullPolicy: Never
  replicaCount: 1
  service:
    type: NodePort
    port: 80
  ingress:
    enabled: true
    host: "to-do-app.local"
    path: "/"
    pathType: "Prefix"
```

### 5. تبدیل مانیفست‌های موجود به تمپلیت‌های Helm

#### الف) تبدیل MySQL StatefulSet
فایل `templates/mysql-statefulset.yaml` را با محتوای زیر ایجاد کنید:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: {{ include "todo-app.fullname" . }}-mysql
  labels:
    {{- include "todo-app.labels" . | nindent 4 }}
    app.kubernetes.io/component: mysql
spec:
  serviceName: {{ include "todo-app.fullname" . }}-mysql
  replicas: 1
  selector:
    matchLabels:
      {{- include "todo-app.selectorLabels" . | nindent 6 }}
      app.kubernetes.io/component: mysql
  template:
    metadata:
      labels:
        {{- include "todo-app.selectorLabels" . | nindent 8 }}
        app.kubernetes.io/component: mysql
    spec:
      containers:
        - name: mysql
          image: {{ .Values.mysql.image }}
          imagePullPolicy: {{ .Values.mysql.imagePullPolicy }}
          env:
            - name: MYSQL_DATABASE
              value: {{ .Values.mysql.databaseName }}
            - name: MYSQL_ROOT_PASSWORD
              value: {{ .Values.mysql.rootPassword }}
          ports:
            - containerPort: 3306
          volumeMounts:
            - name: mysql-persistent-storage
              mountPath: /var/lib/mysql
  volumeClaimTemplates:
    - metadata:
        name: mysql-persistent-storage
      spec:
        accessModes: [ "ReadWriteOnce" ]
        resources:
          requests:
            storage: {{ .Values.mysql.persistence.size }}
```

#### ب) تبدیل MySQL Service
فایل `templates/mysql-service.yaml` را با محتوای زیر ایجاد کنید:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "todo-app.fullname" . }}-mysql
  labels:
    {{- include "todo-app.labels" . | nindent 4 }}
    app.kubernetes.io/component: mysql
spec:
  selector:
    {{- include "todo-app.selectorLabels" . | nindent 4 }}
    app.kubernetes.io/component: mysql
  ports:
    - protocol: TCP
      port: {{ .Values.mysql.service.port }}
      targetPort: 3306
```

#### ج) تبدیل ConfigMap
فایل `templates/configmap.yaml` را با محتوای زیر ایجاد کنید:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "todo-app.fullname" . }}-config
  labels:
    {{- include "todo-app.labels" . | nindent 4 }}
spec:
  data:
    DATABASE_HOST: {{ include "todo-app.fullname" . }}-mysql
```

#### د) تبدیل Secret
فایل `templates/secret.yaml` را با محتوای زیر ایجاد کنید:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "todo-app.fullname" . }}-db-secret
  labels:
    {{- include "todo-app.labels" . | nindent 4 }}
type: Opaque
stringData:
  MYSQL_DATABASE: {{ .Values.mysql.databaseName }}
  MYSQL_ROOT_PASSWORD: {{ .Values.mysql.rootPassword }}
  DATABASE_USER: root
```

#### ه) تبدیل Backend Deployment
فایل `templates/backend-deployment.yaml` را با محتوای زیر ایجاد کنید:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "todo-app.fullname" . }}-backend
  labels:
    {{- include "todo-app.labels" . | nindent 4 }}
    app.kubernetes.io/component: backend
spec:
  replicas: {{ .Values.backend.replicaCount }}
  selector:
    matchLabels:
      {{- include "todo-app.selectorLabels" . | nindent 6 }}
      app.kubernetes.io/component: backend
  template:
    metadata:
      labels:
        {{- include "todo-app.selectorLabels" . | nindent 8 }}
        app.kubernetes.io/component: backend
    spec:
      containers:
        - name: backend
          image: {{ .Values.backend.image }}
          imagePullPolicy: {{ .Values.backend.imagePullPolicy }}
          ports:
            - containerPort: {{ .Values.backend.service.port }}
          envFrom:
            - configMapRef:
                name: {{ include "todo-app.fullname" . }}-config
            - secretRef:
                name: {{ include "todo-app.fullname" . }}-db-secret
```

#### و) تبدیل Backend Service
فایل `templates/backend-service.yaml` را با محتوای زیر ایجاد کنید:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "todo-app.fullname" . }}-backend
  labels:
    {{- include "todo-app.labels" . | nindent 4 }}
    app.kubernetes.io/component: backend
spec:
  selector:
    {{- include "todo-app.selectorLabels" . | nindent 4 }}
    app.kubernetes.io/component: backend
  ports:
    - protocol: TCP
      port: {{ .Values.backend.service.port }}
      targetPort: {{ .Values.backend.service.port }}
  type: {{ .Values.backend.service.type }}
```

#### ز) تبدیل Frontend Deployment
فایل `templates/frontend-deployment.yaml` را با محتوای زیر ایجاد کنید:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "todo-app.fullname" . }}-frontend
  labels:
    {{- include "todo-app.labels" . | nindent 4 }}
    app.kubernetes.io/component: frontend
spec:
  replicas: {{ .Values.frontend.replicaCount }}
  selector:
    matchLabels:
      {{- include "todo-app.selectorLabels" . | nindent 6 }}
      app.kubernetes.io/component: frontend
  template:
    metadata:
      labels:
        {{- include "todo-app.selectorLabels" . | nindent 8 }}
        app.kubernetes.io/component: frontend
    spec:
      containers:
        - name: frontend
          image: {{ .Values.frontend.image }}
          imagePullPolicy: {{ .Values.frontend.imagePullPolicy }}
          ports:
            - containerPort: 80
```

#### ح) تبدیل Frontend Service
فایل `templates/frontend-service.yaml` را با محتوای زیر ایجاد کنید:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "todo-app.fullname" . }}-frontend
  labels:
    {{- include "todo-app.labels" . | nindent 4 }}
    app.kubernetes.io/component: frontend
spec:
  selector:
    {{- include "todo-app.selectorLabels" . | nindent 4 }}
    app.kubernetes.io/component: frontend
  ports:
    - protocol: TCP
      port: {{ .Values.frontend.service.port }}
      targetPort: 80
  type: {{ .Values.frontend.service.type }}
```

#### ط) تبدیل Ingress
فایل `templates/ingress.yaml` را با محتوای زیر ایجاد کنید:

```yaml
{{- if or .Values.backend.ingress.enabled .Values.frontend.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "todo-app.fullname" . }}-ingress
  labels:
    {{- include "todo-app.labels" . | nindent 4 }}
spec:
  rules:
    {{- if .Values.backend.ingress.enabled }}
    - host: {{ .Values.backend.ingress.host }}
      http:
        paths:
          - path: {{ .Values.backend.ingress.path }}
            pathType: {{ .Values.backend.ingress.pathType }}
            backend:
              service:
                name: {{ include "todo-app.fullname" . }}-backend
                port:
                  number: {{ .Values.backend.service.port }}
    {{- end }}
    {{- if .Values.frontend.ingress.enabled }}
    - host: {{ .Values.frontend.ingress.host }}
      http:
        paths:
          - path: {{ .Values.frontend.ingress.path }}
            pathType: {{ .Values.frontend.ingress.pathType }}
            backend:
              service:
                name: {{ include "todo-app.fullname" . }}-frontend
                port:
                  number: {{ .Values.frontend.service.port }}
    {{- end }}
{{- end }}
```

### 6. ایجاد Helpers
در فایل `templates/_helpers.tpl`، توابع کمکی زیر را تعریف کنید:

```
{{/*
Expand the name of the chart.
*/}}
{{- define "todo-app.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
We truncate at 63 chars because some Kubernetes name fields are limited to this (by the DNS naming spec).
If release name contains chart name it will be used as a full name.
*/}}
{{- define "todo-app.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Create chart name and version as used by the chart label.
*/}}
{{- define "todo-app.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "todo-app.labels" -}}
helm.sh/chart: {{ include "todo-app.chart" . }}
{{ include "todo-app.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "todo-app.selectorLabels" -}}
app.kubernetes.io/name: {{ include "todo-app.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

### 7. ایجاد NOTES.txt
فایل `templates/NOTES.txt` را با محتوای زیر ایجاد کنید:

```
برنامه Todo List با موفقیت نصب شد!

برای دسترسی به برنامه:

1. پیکربندی فایل hosts سیستم خود:
   {{ .Values.frontend.ingress.host }} باید به آدرس IP کلاستر شما اشاره کند.

2. دسترسی به فرانت‌اند:
   http://{{ .Values.frontend.ingress.host }}

3. دسترسی به API بک‌اند:
   http://{{ .Values.backend.ingress.host }}{{ .Values.backend.ingress.path }}

برای دیدن وضعیت پاد‌ها:
  kubectl get pods --namespace {{ .Release.Namespace }}

برای دیدن وضعیت سرویس‌ها:
  kubectl get svc --namespace {{ .Release.Namespace }}
```

### 8. تست و اصلاح
چارت را با دستور زیر تست کنید:
```bash
helm install --dry-run --debug todo-release ./todo-app
```

### 9. نصب چارت
پس از اطمینان از صحت چارت، آن را نصب کنید:
```bash
helm install todo-release ./todo-app
```

### 10. بررسی نهایی
پس از نصب چارت، وضعیت برنامه را بررسی کنید:
```bash
kubectl get all
```

و با مراجعه به آدرس تعریف شده در مرورگر، برنامه Todo List را مشاهده کنید.