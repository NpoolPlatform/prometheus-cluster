pipeline {
  agent any
  environment {
    GOPROXY = 'https://goproxy.cn,direct'
  }
  tools {
    go 'go'
  }
  stages {
    stage('Clone prometheus cluster') {
      steps {
        git(url: scm.userRemoteConfigs[0].url, branch: '$BRANCH_NAME', changelog: true, credentialsId: 'KK-github-key', poll: true)
      }
    }

    stage('Check deps tools') {
      steps {
        script {
          if (!fileExists("/usr/bin/helm")) {
            sh 'mkdir -p $HOME/.helm'
            if (!fileExists("$HOME/.helm/.helm-src")) {
              sh 'git clone https://github.com/helm/helm.git $HOME/.helm/.helm-src'
            }
            sh 'cd $HOME/.helm/.helm-src; git checkout release-3.7; make; cp bin/helm /usr/bin/helm'
            sh 'helm version'
          }
        }
      }
    }

    stage('Switch to current cluster') {
      steps {
        sh 'cd /etc/kubeasz; ./ezctl checkout $TARGET_ENV'
      }
    }

    stage('Deploy prometheus cluster') {
      when {
        expression { DEPLOY_TARGET == 'true' }
      }
      steps {
        sh 'helm repo add prometheus-community https://prometheus-community.github.io/helm-charts'
        sh 'helm repo update'
        sh 'GRAFANA_PASSWORD=$GRAFANA_PASSWORD TARGET_ENV=$TARGET_ENV WECHAT_CORP_ID=$WECHAT_CORP_ID WECHAT_AGENT_ID=$WECHAT_AGENT_ID WECHAT_API_SECRET=$WECHAT_API_SECRET STORAGE_CLASS_NAME=$STORAGE_CLASS_NAME envsubst < values.yaml > .values.yaml'
        sh 'kubectl apply -f ./alertmanager/template/'
        sh 'helm upgrade prometheus -f .values.yaml --namespace monitor ./kube-prometheus-stack || helm install prometheus -f .values.yaml --namespace monitor ./kube-prometheus-stack'
        sh 'kubectl apply -f ./grafana-dashboards/'
        sh 'kubectl apply -f ./prometheus/rule/'
      }
    }

    stage('Deploy redis exporter cluster') {
      when {
        expression { DEPLOY_TARGET == 'true' }
      }
      steps {
        sh 'helm repo add prometheus-community https://prometheus-community.github.io/helm-charts'
        sh 'helm repo add stable https://charts.helm.sh/stable'
        sh 'helm repo update'
        sh 'helm upgrade prometheus-redis-exporter -f ./redis-exporter/values.yaml --namespace monitor ./redis-exporter/prometheus-redis-exporter || helm install prometheus-redis-exporter -f ./redis-exporter/values.yaml --namespace monitor ./redis-exporter/prometheus-redis-exporter'
      }
    }
  }

  post('Report') {
    fixed {
      script {
        sh(script: 'bash $JENKINS_HOME/wechat-templates/send_wxmsg.sh fixed')
     }
      script {
        // env.ForEmailPlugin = env.WORKSPACE
        emailext attachmentsPattern: 'TestResults\\*.trx',
        body: '${FILE,path="$JENKINS_HOME/email-templates/success_email_tmp.html"}',
        mimeType: 'text/html',
        subject: currentBuild.currentResult + " : " + env.JOB_NAME,
        to: '$DEFAULT_RECIPIENTS'
      }
     }
    success {
      script {
        sh(script: 'bash $JENKINS_HOME/wechat-templates/send_wxmsg.sh successful')
     }
      script {
        // env.ForEmailPlugin = env.WORKSPACE
        emailext attachmentsPattern: 'TestResults\\*.trx',
        body: '${FILE,path="$JENKINS_HOME/email-templates/success_email_tmp.html"}',
        mimeType: 'text/html',
        subject: currentBuild.currentResult + " : " + env.JOB_NAME,
        to: '$DEFAULT_RECIPIENTS'
      }
     }
    failure {
      script {
        sh(script: 'bash $JENKINS_HOME/wechat-templates/send_wxmsg.sh failure')
     }
      script {
        // env.ForEmailPlugin = env.WORKSPACE
        emailext attachmentsPattern: 'TestResults\\*.trx',
        body: '${FILE,path="$JENKINS_HOME/email-templates/fail_email_tmp.html"}',
        mimeType: 'text/html',
        subject: currentBuild.currentResult + " : " + env.JOB_NAME,
        to: '$DEFAULT_RECIPIENTS'
      }
     }
    aborted {
      script {
        sh(script: 'bash $JENKINS_HOME/wechat-templates/send_wxmsg.sh aborted')
     }
      script {
        // env.ForEmailPlugin = env.WORKSPACE
        emailext attachmentsPattern: 'TestResults\\*.trx',
        body: '${FILE,path="$JENKINS_HOME/email-templates/fail_email_tmp.html"}',
        mimeType: 'text/html',
        subject: currentBuild.currentResult + " : " + env.JOB_NAME,
        to: '$DEFAULT_RECIPIENTS'
      }
     }
  }
}