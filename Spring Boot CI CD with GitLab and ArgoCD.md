# Spring Boot CI/CD with GitLab and ArgoCD

## 개념
- GitLab CI/CD와 ArgoCD를 활용한 완전한 DevOps 파이프라인
- [[Spring Boot Project Structure]]를 기반으로 한 자동화된 배포
- Kubernetes 클러스터에서의 운영 환경 구성
- [[Enterprise Design Patterns]]을 지원하는 인프라 구조

## 1. 전체 CI/CD 아키텍처
### 파이프라인 개요
```
Developer → GitLab → GitLab CI → Docker Registry → ArgoCD → Kubernetes
    ↓         ↓         ↓           ↓               ↓         ↓
 Git Push  Build &   Container   Image Store   GitOps     Deploy
          Test      Creation                   Sync       to K8s
```

### 주요 구성 요소
- **GitLab**: 소스 코드 관리 및 CI 파이프라인
- **GitLab CI**: 빌드, 테스트, 이미지 생성
- **Docker Registry**: 컨테이너 이미지 저장소
- **ArgoCD**: GitOps 기반 CD 파이프라인
- **Kubernetes**: 컨테이너 오케스트레이션

## 2. GitLab CI 설정
### .gitlab-ci.yml 파일 구성
```yaml
# GitLab CI/CD Pipeline for Spring Boot Application
stages:
  - validate
  - test
  - build
  - security-scan
  - package
  - deploy-dev
  - deploy-staging
  - deploy-prod

variables:
  MAVEN_OPTS: "-Dmaven.repo.local=$CI_PROJECT_DIR/.m2/repository"
  MAVEN_CLI_OPTS: "--batch-mode --errors --fail-at-end --show-version"
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: "/certs"
  IMAGE_NAME: "$CI_REGISTRY_IMAGE"
  HELM_CHART_PATH: "k8s/helm"

# Maven 의존성 캐싱
cache:
  paths:
    - .m2/repository/
    - target/

# 1단계: 코드 품질 검증
validate:
  stage: validate
  image: maven:3.9-openjdk-21
  script:
    - mvn $MAVEN_CLI_OPTS validate
    - mvn $MAVEN_CLI_OPTS compile
  only:
    - merge_requests
    - main
    - develop

# 2단계: 단위 테스트 및 통합 테스트
unit-test:
  stage: test
  image: maven:3.9-openjdk-21
  services:
    - postgres:15
    - redis:7
  variables:
    POSTGRES_DB: testdb
    POSTGRES_USER: testuser
    POSTGRES_PASSWORD: testpass
    SPRING_PROFILES_ACTIVE: test
  script:
    - mvn $MAVEN_CLI_OPTS test
  artifacts:
    reports:
      junit:
        - target/surefire-reports/TEST-*.xml
        - target/failsafe-reports/TEST-*.xml
    paths:
      - target/site/jacoco/
  coverage: '/Total.*?([0-9]{1,3})%/'

integration-test:
  stage: test
  image: maven:3.9-openjdk-21
  services:
    - postgres:15
    - redis:7
    - docker:dind
  variables:
    DOCKER_HOST: tcp://docker:2376
    DOCKER_TLS_CERTDIR: "/certs"
    TESTCONTAINERS_HOST_OVERRIDE: docker
  script:
    - mvn $MAVEN_CLI_OPTS verify -Pintegration-tests
  artifacts:
    reports:
      junit:
        - target/failsafe-reports/TEST-*.xml

# 3단계: SonarQube 코드 품질 분석
sonarqube-check:
  stage: test
  image: maven:3.9-openjdk-21
  variables:
    SONAR_USER_HOME: "${CI_PROJECT_DIR}/.sonar"
    GIT_DEPTH: "0"
  cache:
    key: "${CI_JOB_NAME}"
    paths:
      - .sonar/cache
  script:
    - mvn $MAVEN_CLI_OPTS verify sonar:sonar
      -Dsonar.projectKey=$CI_PROJECT_NAME
      -Dsonar.host.url=$SONAR_HOST_URL
      -Dsonar.login=$SONAR_TOKEN
  allow_failure: true
  only:
    - main
    - develop
    - merge_requests

# 4단계: 애플리케이션 빌드
build:
  stage: build
  image: maven:3.9-openjdk-21
  script:
    - mvn $MAVEN_CLI_OPTS clean package -DskipTests
    - echo "BUILD_VERSION=$(mvn help:evaluate -Dexpression=project.version -q -DforceStdout)" >> build.env
  artifacts:
    paths:
      - target/*.jar
    reports:
      dotenv: build.env
    expire_in: 1 hour
  only:
    - main
    - develop
    - tags

# 5단계: 보안 스캔
security-scan:
  stage: security-scan
  image: owasp/dependency-check
  script:
    - dependency-check.sh 
      --project "$CI_PROJECT_NAME"
      --scan target/
      --format ALL
      --out reports/
  artifacts:
    paths:
      - reports/
    expire_in: 1 week
  allow_failure: true
  only:
    - main
    - develop

# 6단계: Docker 이미지 빌드 및 푸시
docker-build:
  stage: package
  image: docker:24
  services:
    - docker:24-dind
  variables:
    DOCKER_HOST: tcp://docker:2376
    DOCKER_TLS_CERTDIR: "/certs"
  before_script:
    - echo $CI_REGISTRY_PASSWORD | docker login -u $CI_REGISTRY_USER --password-stdin $CI_REGISTRY
  script:
    - |
      if [[ "$CI_COMMIT_BRANCH" == "main" ]]; then
        TAG="latest"
      elif [[ "$CI_COMMIT_TAG" ]]; then
        TAG="$CI_COMMIT_TAG"
      else
        TAG="$CI_COMMIT_SHORT_SHA"
      fi
    - echo "Building Docker image with tag: $TAG"
    - docker build -t $IMAGE_NAME:$TAG .
    - docker tag $IMAGE_NAME:$TAG $IMAGE_NAME:$CI_COMMIT_SHORT_SHA
    - docker push $IMAGE_NAME:$TAG
    - docker push $IMAGE_NAME:$CI_COMMIT_SHORT_SHA
    - echo "IMAGE_TAG=$TAG" >> docker.env
  artifacts:
    reports:
      dotenv: docker.env
  only:
    - main
    - develop
    - tags

# 7단계: 개발 환경 배포
deploy-dev:
  stage: deploy-dev
  image: alpine/helm:latest
  variables:
    ENVIRONMENT: development
    NAMESPACE: ecommerce-dev
  before_script:
    - apk add --no-cache git curl
    - helm repo add bitnami https://charts.bitnami.com/bitnami
    - helm repo update
  script:
    - |
      helm upgrade --install ecommerce-api $HELM_CHART_PATH \
        --namespace $NAMESPACE \
        --create-namespace \
        --set image.repository=$IMAGE_NAME \
        --set image.tag=$IMAGE_TAG \
        --set environment=$ENVIRONMENT \
        --set ingress.hosts[0].host=api-dev.company.com \
        --values k8s/helm/values-dev.yaml \
        --wait \
        --timeout=300s
  environment:
    name: development
    url: https://api-dev.company.com
  only:
    - develop

# 8단계: 스테이징 환경 배포
deploy-staging:
  stage: deploy-staging
  image: alpine/helm:latest
  variables:
    ENVIRONMENT: staging
    NAMESPACE: ecommerce-staging
  before_script:
    - apk add --no-cache git curl
    - helm repo add bitnami https://charts.bitnami.com/bitnami
    - helm repo update
  script:
    - |
      helm upgrade --install ecommerce-api $HELM_CHART_PATH \
        --namespace $NAMESPACE \
        --create-namespace \
        --set image.repository=$IMAGE_NAME \
        --set image.tag=$IMAGE_TAG \
        --set environment=$ENVIRONMENT \
        --set ingress.hosts[0].host=api-staging.company.com \
        --values k8s/helm/values-staging.yaml \
        --wait \
        --timeout=300s
    - sleep 30
    - curl -f https://api-staging.company.com/actuator/health || exit 1
  environment:
    name: staging
    url: https://api-staging.company.com
  when: manual
  only:
    - main

# 9단계: 운영 환경 배포 (ArgoCD 연동)
deploy-prod:
  stage: deploy-prod
  image: alpine/git
  variables:
    ENVIRONMENT: production
    GITOPS_REPO: "git@gitlab.company.com:devops/gitops-configs.git"
  before_script:
    - apk add --no-cache git openssh-client
    - eval $(ssh-agent -s)
    - echo "$DEPLOY_SSH_PRIVATE_KEY" | tr -d '\r' | ssh-add -
    - mkdir -p ~/.ssh
    - chmod 700 ~/.ssh
    - ssh-keyscan gitlab.company.com >> ~/.ssh/known_hosts
    - chmod 644 ~/.ssh/known_hosts
    - git config --global user.email "gitlab-ci@company.com"
    - git config --global user.name "GitLab CI"
  script:
    - git clone $GITOPS_REPO gitops-configs
    - cd gitops-configs
    - |
      # Update Helm values for production
      sed -i "s|tag:.*|tag: $IMAGE_TAG|g" apps/ecommerce-api/values-prod.yaml
      sed -i "s|appVersion:.*|appVersion: $IMAGE_TAG|g" apps/ecommerce-api/Chart.yaml
    - git add .
    - git commit -m "Update ecommerce-api to $IMAGE_TAG" || echo "No changes to commit"
    - git push origin main
  environment:
    name: production
    url: https://api.company.com
  when: manual
  only:
    - tags
```

## 3. Dockerfile 최적화
### 멀티스테이지 빌드를 활용한 효율적인 이미지
```dockerfile
# Multi-stage build for Spring Boot application
FROM maven:3.9-openjdk-21 AS builder

# Set working directory
WORKDIR /app

# Copy pom.xml first to leverage Docker cache
COPY pom.xml .
COPY src ./src

# Build the application
RUN mvn clean package -DskipTests

# Runtime stage
FROM openjdk:21-jdk-slim

# Create non-root user for security
RUN groupadd -r spring && useradd -r -g spring spring

# Install required packages
RUN apt-get update && apt-get install -y \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Set working directory
WORKDIR /app

# Copy the built jar from builder stage
COPY --from=builder /app/target/*.jar app.jar

# Change ownership to spring user
RUN chown -R spring:spring /app

# Switch to non-root user
USER spring

# Expose port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1

# JVM optimization for containers
ENV JAVA_OPTS="-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0 -XX:+UseG1GC"

# Run the application
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

## 4. Kubernetes 매니페스트
### Helm Chart 구조
```
k8s/helm/
├── Chart.yaml
├── values.yaml
├── values-dev.yaml
├── values-staging.yaml
├── values-prod.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── configmap.yaml
    ├── secret.yaml
    ├── hpa.yaml
    ├── servicemonitor.yaml
    └── _helpers.tpl
```

### Helm Chart 템플릿 예시
```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "ecommerce-api.fullname" . }}
  labels:
    {{- include "ecommerce-api.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "ecommerce-api.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
        prometheus.io/scrape: "true"
        prometheus.io/path: "/actuator/prometheus"
        prometheus.io/port: "8080"
      labels:
        {{- include "ecommerce-api.selectorLabels" . | nindent 8 }}
    spec:
      serviceAccountName: {{ include "ecommerce-api.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      containers:
        - name: {{ .Chart.Name }}
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: {{ .Values.environment }}
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: {{ include "ecommerce-api.fullname" . }}-secret
                  key: database-url
            - name: DATABASE_USERNAME
              valueFrom:
                secretKeyRef:
                  name: {{ include "ecommerce-api.fullname" . }}-secret
                  key: database-username
            - name: DATABASE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: {{ include "ecommerce-api.fullname" . }}-secret
                  key: database-password
            - name: REDIS_URL
              valueFrom:
                configMapKeyRef:
                  name: {{ include "ecommerce-api.fullname" . }}-config
                  key: redis-url
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: http
            initialDelaySeconds: 60
            periodSeconds: 30
            timeoutSeconds: 5
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: http
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          volumeMounts:
            - name: config-volume
              mountPath: /app/config
            - name: logs-volume
              mountPath: /app/logs
      volumes:
        - name: config-volume
          configMap:
            name: {{ include "ecommerce-api.fullname" . }}-config
        - name: logs-volume
          emptyDir: {}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}

---
# templates/hpa.yaml
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "ecommerce-api.fullname" . }}
  labels:
    {{- include "ecommerce-api.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "ecommerce-api.fullname" . }}
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
  metrics:
    {{- if .Values.autoscaling.targetCPUUtilizationPercentage }}
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}
    {{- end }}
    {{- if .Values.autoscaling.targetMemoryUtilizationPercentage }}
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetMemoryUtilizationPercentage }}
    {{- end }}
{{- end }}
```

### 환경별 Values 파일
```yaml
# values-prod.yaml
replicaCount: 3

image:
  repository: registry.company.com/ecommerce/api
  pullPolicy: IfNotPresent
  tag: ""

environment: production

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  className: "nginx"
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
  hosts:
    - host: api.company.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: api-company-com-tls
      hosts:
        - api.company.com

resources:
  limits:
    cpu: 1000m
    memory: 2Gi
  requests:
    cpu: 500m
    memory: 1Gi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

nodeSelector: {}

tolerations: []

affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchExpressions:
          - key: app.kubernetes.io/name
            operator: In
            values:
            - ecommerce-api
        topologyKey: kubernetes.io/hostname

# Database configuration
database:
  host: postgres-prod.company.com
  port: 5432
  name: ecommerce_prod
  ssl: true

# Redis configuration  
redis:
  host: redis-prod.company.com
  port: 6379
  ssl: true

# Monitoring
monitoring:
  enabled: true
  prometheus:
    enabled: true
  grafana:
    enabled: true
```

## 5. ArgoCD 설정
### Application 매니페스트
```yaml
# argocd/applications/ecommerce-api-prod.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ecommerce-api-prod
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: ecommerce
  source:
    repoURL: git@gitlab.company.com:devops/gitops-configs.git
    targetRevision: main
    path: apps/ecommerce-api
    helm:
      valueFiles:
        - values-prod.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: ecommerce-prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  revisionHistoryLimit: 10
```

### ArgoCD Project 설정
```yaml
# argocd/projects/ecommerce-project.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: ecommerce
  namespace: argocd
spec:
  description: E-commerce Platform Applications
  sourceRepos:
    - 'git@gitlab.company.com:devops/gitops-configs.git'
    - 'https://charts.bitnami.com/bitnami'
  destinations:
    - namespace: 'ecommerce-*'
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: ''
      kind: Namespace
    - group: 'rbac.authorization.k8s.io'
      kind: ClusterRole
    - group: 'rbac.authorization.k8s.io'
      kind: ClusterRoleBinding
  namespaceResourceWhitelist:
    - group: ''
      kind: '*'
    - group: 'apps'
      kind: '*'
    - group: 'extensions'
      kind: '*'
    - group: 'networking.k8s.io'
      kind: '*'
    - group: 'autoscaling'
      kind: '*'
    - group: 'monitoring.coreos.com'
      kind: '*'
  roles:
    - name: developers
      policies:
        - p, proj:ecommerce:developers, applications, sync, ecommerce/*, allow
        - p, proj:ecommerce:developers, applications, get, ecommerce/*, allow
      groups:
        - company:developers
    - name: devops
      policies:
        - p, proj:ecommerce:devops, applications, *, ecommerce/*, allow
        - p, proj:ecommerce:devops, repositories, *, *, allow
      groups:
        - company:devops
```

## 6. GitOps Repository 구조
### GitOps 설정 저장소 구조
```
gitops-configs/
├── apps/
│   ├── ecommerce-api/
│   │   ├── Chart.yaml
│   │   ├── values-dev.yaml
│   │   ├── values-staging.yaml
│   │   ├── values-prod.yaml
│   │   └── templates/
│   └── ecommerce-frontend/
├── infrastructure/
│   ├── cert-manager/
│   ├── ingress-nginx/
│   ├── monitoring/
│   └── argocd/
├── environments/
│   ├── development/
│   ├── staging/
│   └── production/
└── scripts/
    ├── setup-cluster.sh
    ├── deploy-argocd.sh
    └── bootstrap-apps.sh
```

## 7. 모니터링 및 관찰성
### Prometheus ServiceMonitor
```yaml
# templates/servicemonitor.yaml
{{- if .Values.monitoring.prometheus.enabled }}
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: {{ include "ecommerce-api.fullname" . }}
  labels:
    {{- include "ecommerce-api.labels" . | nindent 4 }}
spec:
  selector:
    matchLabels:
      {{- include "ecommerce-api.selectorLabels" . | nindent 6 }}
  endpoints:
  - port: http
    path: /actuator/prometheus
    interval: 30s
    scrapeTimeout: 10s
{{- end }}
```

### Grafana Dashboard 설정
```json
{
  "dashboard": {
    "title": "Spring Boot Application Metrics",
    "panels": [
      {
        "title": "HTTP Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(http_server_requests_seconds_count{application=\"ecommerce-api\"}[5m])",
            "legendFormat": "{{method}} {{uri}}"
          }
        ]
      },
      {
        "title": "JVM Memory Usage",
        "type": "graph", 
        "targets": [
          {
            "expr": "jvm_memory_used_bytes{application=\"ecommerce-api\"}",
            "legendFormat": "{{area}}"
          }
        ]
      }
    ]
  }
}
```

## 8. 보안 및 정책
### Network Policy
```yaml
# templates/networkpolicy.yaml
{{- if .Values.networkPolicy.enabled }}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: {{ include "ecommerce-api.fullname" . }}
  labels:
    {{- include "ecommerce-api.labels" . | nindent 4 }}
spec:
  podSelector:
    matchLabels:
      {{- include "ecommerce-api.selectorLabels" . | nindent 6 }}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    - namespaceSelector:
        matchLabels:
          name: monitoring
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: database
    ports:
    - protocol: TCP
      port: 5432
  - to:
    - namespaceSelector:
        matchLabels:
          name: redis
    ports:
    - protocol: TCP
      port: 6379
  - to: []
    ports:
    - protocol: TCP
      port: 53
    - protocol: UDP
      port: 53
{{- end }}
```

### Pod Security Policy
```yaml
# templates/podsecuritypolicy.yaml
{{- if .Values.podSecurityPolicy.enabled }}
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: {{ include "ecommerce-api.fullname" . }}
  labels:
    {{- include "ecommerce-api.labels" . | nindent 4 }}
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
    - ALL
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    - 'persistentVolumeClaim'
  runAsUser:
    rule: 'MustRunAsNonRoot'
  seLinux:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
{{- end }}
```

## 9. 운영 스크립트
### 배포 스크립트
```bash
#!/bin/bash
# scripts/deploy.sh

set -e

ENVIRONMENT=$1
IMAGE_TAG=$2

if [ -z "$ENVIRONMENT" ] || [ -z "$IMAGE_TAG" ]; then
    echo "Usage: $0 <environment> <image_tag>"
    echo "Example: $0 production v1.2.3"
    exit 1
fi

echo "Deploying ecommerce-api to $ENVIRONMENT with tag $IMAGE_TAG"

# Update GitOps repository
git clone git@gitlab.company.com:devops/gitops-configs.git /tmp/gitops-configs
cd /tmp/gitops-configs

# Update image tag
sed -i "s|tag:.*|tag: $IMAGE_TAG|g" apps/ecommerce-api/values-${ENVIRONMENT}.yaml

# Commit changes
git add .
git commit -m "Deploy ecommerce-api $IMAGE_TAG to $ENVIRONMENT"
git push origin main

echo "Deployment triggered. Check ArgoCD for sync status."
```

### 헬스체크 스크립트
```bash
#!/bin/bash
# scripts/health-check.sh

ENVIRONMENT=$1
NAMESPACE="ecommerce-${ENVIRONMENT}"

if [ -z "$ENVIRONMENT" ]; then
    echo "Usage: $0 <environment>"
    exit 1
fi

echo "Checking health of ecommerce-api in $ENVIRONMENT environment..."

# Check pod status
kubectl get pods -n $NAMESPACE -l app.kubernetes.io/name=ecommerce-api

# Check service endpoints
kubectl get endpoints -n $NAMESPACE ecommerce-api

# Health check via API
INGRESS_URL=$(kubectl get ingress -n $NAMESPACE ecommerce-api -o jsonpath='{.spec.rules[0].host}')
curl -f "https://${INGRESS_URL}/actuator/health" || echo "Health check failed"

echo "Health check completed."
```

## 관련 노트
- [[Spring Boot Project Structure]]
- [[Enterprise Design Patterns]]
- [[Security Patterns]]
- [[Observability]]
- [[Spring Boot Actuator]]

#cicd #devops #gitlab #argocd #kubernetes #gitops