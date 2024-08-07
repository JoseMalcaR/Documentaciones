# Comandos Para Listar y Eliminar
# 👋, Lukas Medina y Jose Malca
¡Bienvenido!

## Índice

- [Listar y Eliminar Recursos en GCP](#listar-y-eliminar-recursos-en-gcp)
- [Ejemplo Deploy.yaml](#se-deben-modificar-los-siguentes-datos)
- [Ejemplo Ingress.yaml](#se-deben-modificar-los-siguentes-datos-1)
- [Ejemplo Services.yaml ](#se-deben-modificar-los-siguentes-datos-2)
- [Workflow para subir imagenes(Docker) a GCP a travez del Artifactory Registry](#workflow-para-subir-imagenesdocker-a-gcp-a-travez-del-artifactory-registry)
- [Workflow para subir una pagina web a Firebase Hosting](#workflow-para-subir-una-pagina-web-a-firebase-hosting)
- [Instalar NGINX y Cert-manager para los sitios seguros de cada microservicio](#verificar-el-contexto-actual-de-kubectl)


## Listar y Eliminar Recursos en GCP

```bash
# Listar Servicios
kubectl get svc

# Listar Implementaciones (Deployments)
kubectl get deployments

# Listar Ingresos
kubectl get ing


# Eliminar Servicio
kubectl delete service <nombre-del-servicio>

# Eliminar Implementación (Deployment)
kubectl delete deployment <nombre-de-la-implementacion>

# Eliminar Ingreso
kubectl delete ingress <nombre-del-ingreso>

```

# Ejemplo de Deployment.yaml

## Se deben modificar los siguentes datos 
- name:backend-market
- app:backend-market
- name:backend-market
- image:{IMAGE_TAG}
- containerPort: 8080

```bash

apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-market
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend-market
  template:
    metadata:
      labels:
        app: backend-market
    spec:
      containers:
      - name: backend-market
        image: {IMAGE_TAG}
        ports:
        - containerPort: 8080
        resources:
          # You must specify requests for CPU to autoscale
          # based on CPU utilization
          limits:
            cpu: 50m
            memory: 128Mi
          requests:
            cpu: 50m
            memory: 128Mi
```


# Ejemplo de Ingress.yaml

## Se deben modificar los siguentes datos 
- name: backend-market-ingress
- host:backend-market.tssw.cl
- name: backend-market-ingress
- hosts:backend-market.tssw.cl
- secretName:backend-market-ingress-secret


```bash

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: backend-market-ingress
  namespace: default
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    acme.cert-manager.io/http01-edit-in-place: "true"
spec: 
  rules:
  - host: backend-market.tssw.cl
    http:
      paths:
      - backend:
          service:
            name: backend-market-svc
            port:
              number: 80
        pathType: Prefix
        path: /
  tls:
  - hosts:
    - backend-market.tssw.cl
    secretName: backend-market-ingress-secret

```

# Ejemplo de Service.yaml

## Se deben modificar los siguentes datos 

- name:backend-market-svc
- app:backend-market
- name: backend-market-svc
- targetPort:8080



```bash

apiVersion: v1
kind: Service
metadata:
  name: backend-market-svc
spec:
  selector:
    app: backend-market
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: ClusterIP

```

# Workflow para subir imagenes(Docker) a GCP a travez del Artifactory Registry

Definir los secretos de Github en cada repositorio, estos datos son de gcp.

```bash
name: Build and Push Go Image to Google Cloud Platform
on:
  push:
    branches: [main]

env:
  IMAGE_TAG: ${{ github.sha }}

jobs:
  build-push-gcr:
    name: Build and Push to GCP
    runs-on: ubuntu-latest
    steps:
    - name: Checkout
      uses: actions/checkout@v4


    - name: Authenticate with gcloud CLI
      uses: google-github-actions/auth@v1
      with:
        credentials_json: ${{ secrets.SERVICE_ACCOUNT_KEY }}

    - name: Setup gcloud CLI
      uses: google-github-actions/setup-gcloud@v1
      with:
        project_id: ${{ secrets.PROJECT_ID }}
        
    - name: Login to GAR
      uses: docker/login-action@v3
      with:
          registry: ${{ secrets.GKE_ZONE }}-docker.pkg.dev
          username: _json_key
          password: ${{ secrets.SERVICE_ACCOUNT_KEY }}

    - name: Build Docker Image
      run: docker build -t ${{ secrets.REGISTRY_URL }}/${{ secrets.PROJECT_ID }}/images/${{ secrets.IMAGE_NAME }}:${{ github.sha }} .

  
    - name: Configure Docker Client
      run: |-
        gcloud auth configure-docker --quiet
        gcloud auth configure-docker ${{ secrets.REGISTRY_URL }} --quiet

    - name: Push Docker Image to Artifact Registry
      run: |
          docker push ${{ secrets.REGISTRY_URL }}/${{ secrets.PROJECT_ID }}/images/${{ secrets.IMAGE_NAME }}:${{ github.sha }}
          
    - name: Install Google Cloud SDK
      uses: google-github-actions/setup-gcloud@v0.2.0
      with:
          version: 'latest'
          project_id: ${{ secrets.PROJECT_ID }}
          service_account_key: ${{ secrets.SERVICE_ACCOUNT_KEY }}

    - name: Install gke-gcloud-auth-plugin
      run: gcloud components install gke-gcloud-auth-plugin
      
    - name: Get GKE Credentials
      run: |-
        gcloud container clusters get-credentials ${{ secrets.GKE_CLUSTER }} --zone ${{ secrets.GKE_ZONE }} --project ${{ secrets.PROJECT_ID }}

    - name: Set Image Deployment YAML
      run: sed -i "s~{IMAGE_TAG}~${{ secrets.REGISTRY_URL }}/${{ secrets.PROJECT_ID }}/images/${{ secrets.IMAGE_NAME }}:${{ github.sha }}~" deploy/k8s/deploy.yaml
    
    - name: Apply Kubernetes Deploy YAML
      run: kubectl apply -f ./deploy/k8s/deploy.yaml

    - name: Apply Kubernetes Ingress YAML
      run: kubectl apply -f ./deploy/k8s/ingress.yaml

    - name: Apply Kubernetes Services YAML
      run: kubectl apply -f ./deploy/k8s/services.yaml

```

# Workflow para subir una pagina web a Firebase Hosting
Importante crear archivos .firebaserc y firebase.json

```bash
name: Deploy to Firebase Hosting

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Set up Node.js
        uses: actions/setup-node@v2
        with:
          node-version: '20'  # Ajusta a una versión compatible con Firebase CLI

      - name: Install dependencies
        run: npm install

      - name: Build
        run: npm run build  # Ajusta según tu configuración de construcción

      - name: Deploy to Firebase Hosting
        uses: FirebaseExtended/action-hosting-deploy@v0
        with:
          repoToken: ${{ secrets.GITHUB_TOKEN }}
          firebaseServiceAccount: ${{ secrets.FIREBASE_SERVICE_ACCOUNT_TSS_1S2024 }}
          projectId: tss-1s2024
          channelId: live  # Ajusta el ID del canal según tu configuración
        env:
          FIREBASE_CLI_EXPERIMENTS: webframeworks

```

# Instalar NGINX y Cert-manager para los sitios seguros de cada microservicio.
## Verificar el contexto actual de kubectl

```bash

kubectl config current-context


```


## Configurar el entorno de kubectl para apuntar al clúster

```bash

gcloud container clusters get-credentials k8s-tisw-api --zone=southamerica-west1

```

## Instalar NGINX

https://platform9.com/learn/v1.0/tutorials/nginix-controller-helm


## Configurar e instalar Cert-manager 

```bash

helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.13.2 \
  --set installCRDs=true \
  --set global.leaderElection.namespace=cert-manager

```

## Configura certificados para cert-manager


```bash

apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: jmalca@utem.cl
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx


```



## Failed to download "jetstack/cert-manager" Se debe instalar manualmente jetstack repo

```bash

helm repo add jetstack https://charts.jetstack.io
helm repo update

```


