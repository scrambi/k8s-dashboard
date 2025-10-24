pipeline {
  agent any

  options {
    timestamps()
    ansiColor('xterm')
    disableConcurrentBuilds()
  }

  environment {
    NS_DASH   = 'kubernetes-dashboard'
    NS_MLB    = 'metallb-system'
    NS_ING    = 'ingress-nginx'
    REL_DASH  = 'kubernetes-dashboard'
    TG_CHAT   = '180424264'
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

            echo "[wait] metallb-controller rollout..."
            kubectl -n $NS_MLB rollout status deploy/metallb-controller --timeout=180s

            echo "[wait] metallb-webhook-service endpoints (served by controller)..."
            for i in {1..90}; do
              EP=$(kubectl -n $NS_MLB get endpoints metallb-webhook-service -o jsonpath='{.subsets[0].addresses[0].ip}' 2>/dev/null || true)
              PORT=$(kubectl -n $NS_MLB get endpoints metallb-webhook-service -o jsonpath='{.subsets[0].ports[0].port}' 2>/dev/null || true)
              if [ -n "$EP" ] && [ "$PORT" = "9443" ]; then
                echo "[ok] endpoints ready: $EP:$PORT"
                break
              fi
              echo "[wait] no endpoints yet..."
              sleep 2
            done

            sleep 2
            echo "[apply] metallb-pool.yaml (with retries)"
            n=0
            until kubectl apply -f metallb/metallb-pool.yaml; do
              n=$((n+1))
              if [ $n -ge 5 ]; then
                echo "ERROR: failed to apply metallb-pool.yaml after ${n} attempts"
                kubectl -n $NS_MLB get pods -o wide || true
                kubectl -n $NS_MLB get svc metallb-webhook-service -o wide || true
                kubectl -n $NS_MLB describe deploy metallb-controller || true
                kubectl -n $NS_MLB logs deploy/metallb-controller --all-containers --tail=200 || true
                exit 1
              fi
              echo "[retry $n/5] waiting for webhook admission to be ready..."
              sleep 3
            done
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

    stage('Install/Update Kubernetes Dashboard (with robust retries)') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
          sh '''
            set -e

            echo "[try#1] helm repo add/update (10 retries)..."
            ok=""
            for i in {1..10}; do
              if helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard && helm repo update; then
                ok=1; break
              fi
              echo "[retry $i/10] repo add/update failed, sleep 3s..."
              sleep 3
            done

            if [ -n "$ok" ]; then
              echo "[install] via repo alias kubernetes-dashboard/kubernetes-dashboard (10 retries)"
              for i in {1..10}; do
                if helm upgrade --install $REL_DASH kubernetes-dashboard/kubernetes-dashboard \
                  -n $NS_DASH --create-namespace \
                  -f dashboard/values.yaml; then
                  break
                fi
                echo "[retry $i/10] helm upgrade failed, sleep 3s..."
                sleep 3
              done
            else
              echo "[try#2] falling back to --repo (no repo add), 10 retries..."
              ok2=""
              for i in {1..10}; do
                if helm upgrade --install $REL_DASH kubernetes-dashboard \
                  --repo https://kubernetes.github.io/dashboard \
                  -n $NS_DASH --create-namespace \
                  -f dashboard/values.yaml; then
                  ok2=1; break
                fi
                echo "[retry $i/10] helm upgrade --repo failed, sleep 3s..."
                sleep 3
              done

              if [ -z "$ok2" ]; then
                echo "[try#3] final fallback: helm pull + local install"
                rm -f kubernetes-dashboard-*.tgz || true
                for i in {1..10}; do
                  if helm pull kubernetes-dashboard --repo https://kubernetes.github.io/dashboard; then
                    CHART=$(ls -1 kubernetes-dashboard-*.tgz | head -n1 || true)
                    if [ -n "$CHART" ]; then
                      if helm upgrade --install $REL_DASH "$CHART" \
                        -n $NS_DASH --create-namespace \
                        -f dashboard/values.yaml; then
                        break
                      fi
                    fi
                  fi
                  echo "[retry $i/10] pull/install failed, sleep 3s..."
                  sleep 3
                done
              fi
            fi

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
