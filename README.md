# Spring Dashboard - Spring Boot + Docker + Kubernetes Kustomize POC

This single repository contains the Spring Boot application, Docker build files, and Kustomize manifests for DEV, QA, UAT, and PROD.

## 1. Prerequisites
- Java 17 and Maven 3.8+
- Docker
- kubectl connected to the intended Kubernetes cluster
- A Docker Hub account and permission to push images

## 2. Build and run locally with Maven
```bash
mvn clean package
java -jar target/spring-dashboard-1.0.0.jar
```
Open `http://localhost:8081`. Check:
- `http://localhost:8081/api/status`
- `http://localhost:8081/actuator/health`

## 3. Build and push Docker image
Replace `DOCKERHUB_USERNAME` below with your Docker Hub username.
```bash
sudo docker build -t DOCKERHUB_USERNAME/spring-dashboard:v1 .
sudo docker login
sudo docker push DOCKERHUB_USERNAME/spring-dashboard:v1
```

## 4. Configure Kustomize image
Replace `DOCKERHUB_USERNAME` in:
- `k8s/base/deployment.yaml`
- `k8s/overlays/dev/kustomization.yaml`
- `k8s/overlays/qa/kustomization.yaml`
- `k8s/overlays/uat/kustomization.yaml`
- `k8s/overlays/prod/kustomization.yaml`

The image name in each overlay's `images.name` must match the image name in the base Deployment.

## 5. Render and inspect before deploying
```bash
kubectl kustomize k8s/overlays/dev
kubectl diff -k k8s/overlays/dev
```
Rendering and diff do not deploy resources.

## 6. Deploy DEV
The namespaces `dev`, `qa`, `uat`, and `prod` are expected to exist already. Check first:
```bash
kubectl get namespace dev qa uat prod
kubectl apply -k k8s/overlays/dev
kubectl rollout status deployment/spring-dashboard -n dev
kubectl get pods,svc -n dev -l app=spring-dashboard
```

Test through port-forward:
```bash
kubectl port-forward -n dev service/spring-dashboard 8081:80
```
Then open `http://localhost:8081`.

## 7. Deploy other environments only when ready
```bash
kubectl diff -k k8s/overlays/qa
kubectl apply -k k8s/overlays/qa

kubectl diff -k k8s/overlays/uat
kubectl apply -k k8s/overlays/uat

kubectl diff -k k8s/overlays/prod
kubectl apply -k k8s/overlays/prod
```

## Notes
- This is a learning POC. Review resource requests/limits and cluster capacity before applying.
- Do not deploy to QA/UAT/PROD until DEV has been validated and you have approval.
- The Service is `ClusterIP`; it is not exposed publicly.
- `APP_ENV` and `LOG_LEVEL` are supplied as environment variables. The sample app does not currently change behavior based on them.
- The ConfigMap generator creates a content-hashed ConfigMap name and Kustomize rewrites the Deployment reference. Changing its literals changes the pod template and triggers a rollout on apply.
- Use a new immutable image tag for each build (for example `v2`) rather than overwriting `v1` in a shared environment.
- `kubectl kustomize` renders manifests; `kubectl apply -k` deploys them.
