pipeline {
  agent any

  options {
    timestamps()
    ansiColor('xterm')
    disableConcurrentBuilds()
  }

  environment {
    NS_DASH = 'kubernetes-dashboard'
    NS_MLB  = 'metallb-system'
    NS_ING  = 'ingress-nginx'
    REL_DASH = 'kubernetes-dashboard'
    TG_CHAT = '180424264'
    DASH_HOST = 'dashboard.lan'
    PATH = "${WORKSPACE}/bin:${PATH}"
    HELM_VERSION = 'v3.14.4'   
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Setup kubecontext & helm (no-sudo)') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
          sh '''
            set -e
            kubectl version --client=true
            kubectl config current-context || true

            if ! command -v helm >/dev/null 2>&1; then
              echo "Installing helm ${HELM_VERSION} locally into $WORKSPACE/bin ..."
              mkdir -p "$WORKSPACE/bin"
              OS=linux
              ARCH=amd64
              URL="https://get.helm.sh/helm-${HELM_VERSION}-${OS}-${ARCH}.tar.gz"
              TMP="$(mktemp -d)"
              curl -sSL "$URL" -o "$TMP/helm.tgz"
              tar -xzf "$TMP/helm.tgz" -C "$TMP"
              mv "$TMP/${OS}-${ARCH}/helm" "$WORKSPACE/bin/helm"
              chmod +x "$WORKSPACE/bin/helm"
              rm -rf "$TMP"
              helm version
            else
              echo "helm already present:"
              helm version
            fi
          '''
        }
      }
    }

    stage('Install/Update MetalLB') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
          sh '''
            set -e
            kubectl create namespace $NS_MLB || true
            helm repo add metallb https://metallb.github.io/metallb
            helm repo update
            helm upgrade --install metallb metallb/metallb -n $NS_MLB
            kubectl apply -f metallb/metallb-pool.yaml
          '''
        }
      }
    }

    stage('Install/Update ingress-nginx (LoadBalancer)') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
          sh '''
            set -e
            helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
            helm repo update
            helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
              -n $NS_ING --create-namespace \
              --set controller.service.type=LoadBalancer

            # ждём EXTERNAL-IP от MetalLB
            for i in {1..90}; do
              IP=$(kubectl -n $NS_ING get svc ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}' 2>/dev/null || true)
              if [ -n "$IP" ]; then
                echo "$IP" > ingress_ip.txt
                break
              fi
              sleep 2
            done
            if [ ! -s ingress_ip.txt ]; then
              kubectl -n $NS_ING get svc ingress-nginx-controller -o wide || true
              echo "No EXTERNAL-IP from MetalLB"
              exit 1
            fi
          '''
        }
      }
    }

    stage('Install/Update Kubernetes Dashboard') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
          sh '''
            set -e
            helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard
            helm repo update
            helm upgrade --install $REL_DASH kubernetes-dashboard/kubernetes-dashboard \
              -n $NS_DASH --create-namespace \
              -f dashboard/values.yaml

            kubectl apply -f dashboard/rbac-admin.yaml

            kubectl -n $NS_DASH rollout status deploy/$REL_DASH-api --timeout=300s
            kubectl -n $NS_DASH rollout status deploy/$REL_DASH-web --timeout=300s
            kubectl -n $NS_DASH rollout status deploy/$REL_DASH-auth --timeout=300s
          '''
        }
      }
    }

    stage('Apply Ingress for Dashboard') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
          sh '''
            set -e
            kubectl apply -f ingress/dashboard-ingress.yaml

            IP=$(cat ingress_ip.txt)
            echo "http://${IP}" > url_ip.txt
            echo "http://${DASH_HOST}" > url_host.txt
          '''
        }
      }
    }

    stage('Prepare login token') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
          sh '''
            set -e
            kubectl -n $NS_DASH create token admin-user > token.txt
          '''
        }
      }
    }
  }

  post {
    success {
      withCredentials([string(credentialsId: 'telegram-bot-token', variable: 'TG')]) {
        sh '''
          set +e
          IP=$(cat ingress_ip.txt 2>/dev/null)
          URL_IP=$(cat url_ip.txt 2>/dev/null)
          URL_HOST=$(cat url_host.txt 2>/dev/null)
          TOKEN_SHORT=$(head -c 40 token.txt 2>/dev/null)

          curl -s -X POST -H 'Content-Type: application/json' \
            --data-binary @- "https://api.telegram.org/bot${TG}/sendMessage" <<EOF
{"chat_id":"$TG_CHAT","text":"✅ Dashboard OK: ${JOB_NAME} #${BUILD_NUMBER}\nIngress IP: ${IP}\nURL (без DNS): ${URL_IP}\nURL (с hosts/DNS): ${URL_HOST}\n\nToken (первые 40 симв): ${TOKEN_SHORT}..."}
EOF
        '''
      }
    }
    failure {
      withCredentials([string(credentialsId: 'telegram-bot-token', variable: 'TG')]) {
        sh '''
          set +e
          curl -s -X POST -H 'Content-Type: application/json' \
            --data-binary @- "https://api.telegram.org/bot${TG}/sendMessage" <<EOF
{"chat_id":"$TG_CHAT","text":"❌ FAILED: ${JOB_NAME} #${BUILD_NUMBER} (см. логи Jenkins)"}
EOF
        '''
      }
    }
  }
}
