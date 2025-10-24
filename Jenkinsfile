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
    // если нет DNS — будем открывать по IP; если потом поднимешь DNS, поменяешь на dashboard.lan
    DASH_HOST = 'dashboard.lan'
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Setup kubecontext & helm') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
          sh '''
            set -e
            kubectl version --client=true
            kubectl config current-context || true
            helm version || (curl -sSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash)
          '''
        }
      }
    }

    stage('Install/Update MetalLB') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
          sh """
            set -e
            kubectl create namespace ${NS_MLB} || true
            helm repo add metallb https://metallb.github.io/metallb
            helm repo update
            helm upgrade --install metallb metallb/metallb -n ${NS_MLB}
            # применим твой пул (в репо: metallb/metallb-pool.yaml с диапазоном 192.168.31.248-192.168.31.250)
            kubectl apply -f metallb/metallb-pool.yaml
          """
        }
      }
    }

    stage('Install/Update ingress-nginx (LoadBalancer)') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
          sh """
            set -e
            helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
            helm repo update
            helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \\
              -n ${NS_ING} --create-namespace \\
              --set controller.service.type=LoadBalancer

            # ждём EXTERNAL-IP от MetalLB
            for i in {1..90}; do
              IP=$(kubectl -n ${NS_ING} get svc ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}' 2>/dev/null || true)
              [ -n "$IP" ] && echo "$IP" > ingress_ip.txt && break
              sleep 2
            done
            [ -s ingress_ip.txt ] || { echo "No EXTERNAL-IP from MetalLB"; kubectl -n ${NS_ING} get svc ingress-nginx-controller -o wide; exit 1; }
          """
        }
      }
    }

    stage('Install/Update Kubernetes Dashboard') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
          sh """
            set -e
            helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard
            helm repo update
            helm upgrade --install ${REL_DASH} kubernetes-dashboard/kubernetes-dashboard \\
              -n ${NS_DASH} --create-namespace \\
              -f dashboard/values.yaml

            kubectl apply -f dashboard/rbac-admin.yaml

            kubectl -n ${NS_DASH} rollout status deploy/${REL_DASH}-api --timeout=300s
            kubectl -n ${NS_DASH} rollout status deploy/${REL_DASH}-web --timeout=300s
            kubectl -n ${NS_DASH} rollout status deploy/${REL_DASH}-auth --timeout=300s
          """
        }
      }
    }

    stage('Apply Ingress for Dashboard') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
          sh """
            set -e
            kubectl apply -f ingress/dashboard-ingress.yaml

            # сохраним реальный URL для сообщения:
            IP=$(cat ingress_ip.txt)
            # если DNS не поднимешь — можно будет зайти по http://<IP>
            echo "http://${IP}" > url_ip.txt
            # если позже заведёшь hosts/DNS — будет красиво по имени
            echo "http://${DASH_HOST}" > url_host.txt
          """
        }
      }
    }

    stage('Prepare login token') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
          sh """
            set -e
            kubectl -n ${NS_DASH} create token admin-user > token.txt
          """
        }
      }
    }
  }

  post {
    success {
      withCredentials([string(credentialsId: 'telegram-bot-token', variable: 'TG')]) {
        sh '''
          set -e
          IP=$(cat ingress_ip.txt)
          URL_IP=$(cat url_ip.txt)
          URL_HOST=$(cat url_host.txt)
          TEXT="✅ Dashboard OK: ${JOB_NAME} #${BUILD_NUMBER}\\nIngress IP: ${IP}\\nURL (без DNS): ${URL_IP}\\nURL (c hosts/DNS): ${URL_HOST}\\n\\nToken (первые 40 симв): $(head -c 40 token.txt)..."

          curl -s -X POST -H 'Content-Type: application/json' \
            --data-binary @- "https://api.telegram.org/bot${TG}/sendMessage" <<EOF
{"chat_id":"''' + '${TG_CHAT}' + '''","text":"${TEXT}"}
EOF
        '''
      }
    }
    failure {
      withCredentials([string(credentialsId: 'telegram-bot-token', variable: 'TG')]) {
        sh '''
          curl -s -X POST -H 'Content-Type: application/json' \
            --data-binary @- "https://api.telegram.org/bot${TG}/sendMessage" <<EOF
{"chat_id":"''' + '${TG_CHAT}' + '''","text":"❌ FAILED: ${JOB_NAME} #${BUILD_NUMBER} (см. логи Jenkins)"}
EOF
        ''' || true
      }
    }
  }
}
