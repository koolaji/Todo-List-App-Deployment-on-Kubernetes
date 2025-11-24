# استقرار برنامه لیست کارها در کوبرنتیس

این مخزن شامل کد منبع و فایل‌های پیکربندی کوبرنتیس برای استقرار یک برنامه لیست کارها با بک‌اند Flask، فرانت‌اند React، و پایگاه داده MySQL است.

## فهرست مطالب

- [پیش‌نیازها](#پیش‌نیازها)
- [نصب و راه‌اندازی Kind](#نصب-و-راه‌اندازی-kind)
- [نصب و پیکربندی Ingress در Kind](#نصب-و-پیکربندی-ingress-در-kind)
- [شروع کار](#شروع-کار)
- [پیکربندی](#پیکربندی)
- [پاکسازی](#پاکسازی)

## پیش‌نیازها

قبل از شروع، اطمینان حاصل کنید که پیش‌نیازهای زیر را نصب کرده‌اید:

- **Docker**: برای ساخت ایمیج‌های کانتینر و اجرای Kind به Docker نیاز دارید. می‌توانید آن را از [اینجا](https://www.docker.com/get-started) دانلود کنید.

- **kubectl**: ابزار خط فرمان کوبرنتیس `kubectl` را نصب کنید. دستورالعمل‌های نصب را می‌توانید [اینجا](https://kubernetes.io/docs/tasks/tools/install-kubectl/) پیدا کنید.

- **Kind**: برای راه‌اندازی یک کلاستر کوبرنتیس محلی از Kind (Kubernetes in Docker) استفاده خواهیم کرد.

## نصب و راه‌اندازی Kind

1. **نصب Kind**:

   برای لینوکس یا macOS:
   ```bash
   curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
   # یا برای macOS:
   # curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-darwin-amd64
   chmod +x ./kind
   sudo mv ./kind /usr/local/bin/kind
   ```

   برای ویندوز (با استفاده از Chocolatey):
   ```bash
   choco install kind
   ```

2. **ایجاد کلاستر Kind با پشتیبانی از Ingress**:

   ابتدا یک فایل پیکربندی برای کلاستر Kind ایجاد کنید:
   ```bash
   cat > kind-config.yaml << EOF
   kind: Cluster
   apiVersion: kind.x-k8s.io/v1alpha4
   nodes:
   - role: control-plane
     kubeadmConfigPatches:
     - |
       kind: InitConfiguration
       nodeRegistration:
         kubeletExtraArgs:
           node-labels: "ingress-ready=true"
     extraPortMappings:
     - containerPort: 80
       hostPort: 80
       protocol: TCP
     - containerPort: 443
       hostPort: 443
       protocol: TCP
   EOF
   ```

   سپس با استفاده از این پیکربندی، کلاستر را ایجاد کنید:
   ```bash
   kind create cluster --config kind-config.yaml --name todo-app
   ```

3. **بررسی وضعیت کلاستر**:
   ```bash
   kubectl get nodes
   ```
   
   خروجی باید چیزی شبیه به این باشد:
   ```
   NAME                    STATUS   ROLES           AGE     VERSION
   todo-app-control-plane   Ready    control-plane   2m32s   v1.27.3
   ```

## نصب و پیکربندی Ingress در Kind

1. **نصب NGINX Ingress Controller**:
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
   ```

2. **منتظر بمانید تا Ingress Controller آماده شود**:
   ```bash
   kubectl wait --namespace ingress-nginx \
     --for=condition=ready pod \
     --selector=app.kubernetes.io/component=controller \
     --timeout=90s
   ```

3. **تست Ingress Controller**:
   برای اطمینان از کارکرد صحیح Ingress، می‌توانید یک سرویس تست ایجاد کنید:
   ```bash
   kubectl apply -f - <<EOF
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: test-app
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: test-app
     template:
       metadata:
         labels:
           app: test-app
       spec:
         containers:
         - name: test-app
           image: nginx:latest
           ports:
           - containerPort: 80
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: test-app
   spec:
     ports:
     - port: 80
       targetPort: 80
     selector:
       app: test-app
   ---
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: test-app
   spec:
     rules:
     - host: test-app.local
       http:
         paths:
         - path: /
           pathType: Prefix
           backend:
             service:
               name: test-app
               port:
                 number: 80
   EOF
   ```

4. **تنظیم فایل hosts**:
   اضافه کردن خط زیر به فایل `/etc/hosts` (لینوکس/macOS) یا `C:\Windows\System32\drivers\etc\hosts` (ویندوز):
   ```
   127.0.0.1 test-app.local to-do-app.local
   ```

5. **تست Ingress**:
   در مرورگر خود، آدرس `http://test-app.local` را باز کنید. باید صفحه خوش‌آمدگویی Nginx را مشاهده کنید.

## شروع کار

1. **کلون کردن این مخزن:**
   ```bash
   git clone https://github.com/qxf2/Todo-List-App-Deployment-on-Kubernetes.git
   cd Todo-List-App-Deployment-on-Kubernetes
   ```

2. **ایجاد یک سیکرت برای پایگاه داده MySQL:**
   ```bash
   kubectl create secret generic db-secret --from-literal=MYSQL_DATABASE=todo_database --from-literal=MYSQL_ROOT_PASSWORD=password123 --from-literal=DATABASE_USER=root
   ```

3. **استقرار MySQL:**
   ```bash
   kubectl apply -f kubernetes/mysql-deployment.yaml
   kubectl apply -f kubernetes/mysql-service.yaml
   ```
   
   منتظر بمانید تا پاد MySQL آماده شود:
   ```bash
   kubectl wait --for=condition=ready pod --selector=app=mysql --timeout=120s
   ```

4. **ورود به پاد MySQL و ایجاد جدول در پایگاه داده:**
   ```bash
   # نام پاد MySQL را پیدا کنید
   POD_NAME=$(kubectl get pods -l app=mysql -o jsonpath='{.items[0].metadata.name}')
   
   # وارد پاد شوید و جدول را ایجاد کنید
   kubectl exec -it $POD_NAME -- mysql -u root -ppassword123 -e "
   USE todo_database;
   CREATE TABLE IF NOT EXISTS task (
        id INT AUTO_INCREMENT PRIMARY KEY,
        title VARCHAR(100) NOT NULL,
        done BOOLEAN DEFAULT FALSE
   );
   INSERT INTO task (title, done) VALUES ('Task 1', false), ('Task 2', true);
   SELECT * FROM task;
   "
   ```

5. **استقرار ConfigMap:**
   ```bash
   kubectl apply -f kubernetes/todo-configmap.yml
   ```

6. **آماده‌سازی و استقرار بک‌اند:**
   ```bash
   # ساخت ایمیج Docker بک‌اند
   cd backend/
   docker build -t todo-backend:latest .
   
   # بارگذاری ایمیج به کلاستر Kind
   kind load docker-image todo-backend:latest --name todo-app
   
   # استقرار بک‌اند
   cd ..
   kubectl apply -f kubernetes/backend-deployment.yaml
   kubectl apply -f kubernetes/backend-service.yaml
   kubectl apply -f kubernetes/backend-ingress.yaml
   
   # منتظر بمانید تا پاد بک‌اند آماده شود
   kubectl wait --for=condition=ready pod --selector=app=backend --timeout=60s
   ```

7. **آماده‌سازی و استقرار فرانت‌اند:**
   ```bash
   # ساخت ایمیج Docker فرانت‌اند
   cd frontend/
   docker build -t todo-frontend-react:latest .
   
   # بارگذاری ایمیج به کلاستر Kind
   kind load docker-image todo-frontend-react:latest --name todo-app
   
   # استقرار فرانت‌اند
   cd ..
   kubectl apply -f kubernetes/frontend-deployment.yaml
   kubectl apply -f kubernetes/frontend-service.yaml
   kubectl apply -f kubernetes/frontend-ingress.yaml
   
   # منتظر بمانید تا پاد فرانت‌اند آماده شود
   kubectl wait --for=condition=ready pod --selector=app=frontend --timeout=60s
   ```

8. **دسترسی به برنامه:**
   برنامه Todo List اکنون در آدرس‌های زیر در دسترس است:
   - فرانت‌اند: `http://to-do-app.local`
   - API بک‌اند: `http://to-do-app.local/tasks`

   می‌توانید با دستور زیر وضعیت همه منابع را بررسی کنید:
   ```bash
   kubectl get all
   ```

## پیکربندی

- **پیکربندی MySQL**: می‌توانید پیکربندی MySQL را با اصلاح فایل `kubernetes/mysql-deployment.yaml` سفارشی کنید.
- **پیکربندی بک‌اند**: بک‌اند Flask را با اصلاح `kubernetes/backend-deployment.yaml` پیکربندی کنید.
- **پیکربندی فرانت‌اند**: فرانت‌اند React را با اصلاح `kubernetes/frontend-deployment.yaml` پیکربندی کنید.

## پاکسازی

برای حذف منابع مستقر شده:
```bash
kubectl delete -f kubernetes/
```

برای حذف کلاستر Kind:
```bash
kind delete cluster --name todo-app
```