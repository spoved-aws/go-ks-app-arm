
# Complete DevOps Project

### To Do
1. Naming the application
2. kustomize
3. Managing secrets as they're in plain text currently

### Go Application 
## Local Test - Docker Method

### Initialising for base image
```
alpine:3 base
``` 
### Building OCI artifact using bsf and ko
```
KO_DOCKER_REPO=docker.io/kanukhosla10/go-kubesimplify KO_DEFAULTBASEIMAGE=alpine:3 ko build --bare -t v1 . (change your image names here)
```
### Running using Docker

```
docker run -d --name grafana -p 3000:3000 grafana/grafana
```
```
docker run -d --name prometheus -p 9090:9090 -v $(pwd)/prometheus.yml:/etc/prometheus/prometheus.yml prom/prometheus
```
```
docker run --name local-postgres -e POSTGRES_USER=myuser -e POSTGRES_PASSWORD=mypassword -e POSTGRES_DB=mydb -p 5432:5432 -d postgres
```
```
docker exec -it local-postgres psql -U myuser -d mydb
CREATE TABLE goals (
    id SERIAL PRIMARY KEY,
    goal_name TEXT NOT NULL
);
```
```
docker run -d \
  --platform=linux/amd64 \
  -p 8080:8080 \
  -e DB_USERNAME=myuser \
  -e DB_PASSWORD=mypassword \
  -e DB_HOST=host.docker.internal \
  -e DB_PORT=5432 \
  -e DB_NAME=mydb \
  -e SSL=disable \
 ttl.sh/devops-project-1a3a3957a5f042748486580be307ed8e@sha256:9ae320cdf05700210dd50ebefa6b3cd4a11ca2feaad1946f6715e0ec725bda62
```
## Kubernetes - k3s Cluster

Aiming to save this entire project in the below structure.
<img width="542" alt="image" src="https://github.com/user-attachments/assets/e7e79242-b2bd-42c0-bd31-28e5e0cf154f">


### Manual Installations

#### 1. Cert manager
```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.15.3/cert-manager.yaml
```
#### 2. Install Kube prometheus stack
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack --namespace monitoring --create-namespace

```
#### 3. Getting Grafana login secret for admin user

```
kubectl get secret --namespace monitoring kube-prometheus-stack-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
kubectl port-forward svc/grafana 3000:80 -n monitoring
```

#### 4. Install Traefik ingress gateway for the app and argocd
```
kubectl apply -f k8s-manifests/app_with_traefik/ingress/traefik-resource.yaml
```
NOTE: Trafik resource needs to be deployed only once in the cluster and that takes care of all the applications. This step is not required if ArgoCD has been set already. 

Ref - https://github.com/spoved-aws/go-kubesimplify/blob/main/README-arm64v1.md#install-traefik-ingress-gateway-for-the-app-and-argocd

#### 5. Install Cloudnative postgress DB
```
kubectl apply --server-side -f https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/r
```
The command installs the CloudNativePG Custom Resource Definitions (CRDs) and the necessary controllers.

Here’s why it’s needed:

	1.	CRDs: This command installs the Custom Resource Definitions (CRDs) for CloudNative PostgreSQL. These CRDs allow Kubernetes to understand and manage PostgreSQL clusters as a native resource within your cluster.
	2.	Controller: It also installs the CloudNativePG controller, which is a Kubernetes operator. The controller watches for any CloudNativePG Cluster custom resources you define (like the one you have in your YAML with kind Cluster) and ensures that the necessary PostgreSQL pods and resources (such as StatefulSets, services, etc.) are created and managed according to the spec.

Why you need both:

	•	CRDs define the Cluster resource: Without the CRDs installed, Kubernetes won’t recognize your Cluster kind in the YAML.
	•	The controller reconciles the resources: The controller watches for any changes or creation of CloudNativePG Cluster resources and orchestrates the underlying PostgreSQL pods, PVCs, and services accordingly.

Cluster is installed via Helm Charts + ArgoCD. 


### CI - Jenkins

Pipeline - ```ci-pipeline/Jenkinsfile```

Logic for image tagging - using short SHA to tag the image and use the jinja template to populate the values.yaml file with the short SHA and push the values.yaml file to git.

        stage('Generate deploy manifest from Jinja template') {
            steps {
                sh '''
                  python3 -m venv venv
                  . venv/bin/activate
                  pip install jinja2-cli
                  jinja2 tmpl/values.j2 -o charts/go-ks-app-arm/values.yaml -D image_deploy_tag=sha-${sha_short} --strict
                  deactivate
                '''
            }
        }


      stage('Commit deploy manifest on local repo') {
            steps {
                sh '''
                  git add charts/go-ks-app-arm/values.yaml
                  git commit -s -m "updated image tag"
                '''
            }
        }

#### Stages of the Pipeline
```
1. Checkout
2. Log in to Docker Hub
3. Get git SHA short
4. Build and Publish with ko
5. Generate deploy manifest from Jinja template
6. Configure git for the action
7. Stash unstaged changes
8. Pull latest changes from the remote branch
9. Apply stashed changes
10. Commit deploy manifest on local repo
11. Push deploy manifests to remote repo
```

### CD - HELM + ArgoCD
Application is deployed in the **ksgo-arm** namespace. 

Helm Charts: ```charts/go-ks-app-arm```


### Post Sync of the Application in ArgoCD




