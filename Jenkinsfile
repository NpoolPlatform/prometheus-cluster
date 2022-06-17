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

     stage('Config mysql user') {
      when {
        expression { DEPLOY_TARGET == 'true' }
      }
      steps {
        sh (returnStdout: false, script: '''
            PASSWORD=`kubectl get secret --namespace "kube-system" mysql-password-secret -o jsonpath="{.data.rootpassword}" | base64 --decode`
            kubectl exec --namespace kube-system mysql-0 -- mysql -h 127.0.0.1 -uroot -p$PASSWORD -P3306 -e "
            CREATE USER IF NOT EXISTS 'mysql_exporter'@'%.mysql-exporter.monitor.svc.cluster.local' IDENTIFIED BY '$MYSQL_EXPORTER_PASSWORD';
            ALTER USER 'mysql_exporter'@'%.mysql-exporter.monitor.svc.cluster.local' WITH  MAX_QUERIES_PER_HOUR $MAX_QUERIES_PER_HOUR MAX_CONNECTIONS_PER_HOUR $MAX_CONNECTIONS_PER_HOUR;
            GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'mysql_exporter'@'%.mysql-exporter.monitor.svc.cluster.local';"

            MYSQL_EXPORTER_PASSWORD=$MYSQL_EXPORTER_PASSWORD envsubst < ./mysql-export/mysql-password-secret.yaml > ./mysql-export/.mysql-password-secret.yaml
            kubectl apply -f mysql-export/.mysql-password-secret.yaml
        '''.stripIndent())
      }
    }
    stage('Deploy prometheus cluster') {
      when {
        expression { DEPLOY_TARGET == 'true' }
      }
      steps {
        sh 'helm repo add prometheus-community https://prometheus-community.github.io/helm-charts'
        sh 'GRAFANA_PASSWORD=$GRAFANA_PASSWORD TARGET_ENV=$TARGET_ENV WECHAT_CORP_ID=$WECHAT_CORP_ID WECHAT_AGENT_ID=$WECHAT_AGENT_ID WECHAT_API_SECRET=$WECHAT_API_SECRET STORAGE_CLASS_NAME=$STORAGE_CLASS_NAME NODE_SELECTOR_LABEL_KEY=$NODE_SELECTOR_LABEL_KEY NODE_SELECTOR_LABEL_VALUE=$NODE_SELECTOR_LABEL_VALUE envsubst < values.yaml > .values.yaml'
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
        sh 'NODE_SELECTOR_LABEL_KEY=$NODE_SELECTOR_LABEL_KEY NODE_SELECTOR_LABEL_VALUE=$NODE_SELECTOR_LABEL_VALUE envsubst < ./redis-exporter/values.yaml > ./redis-exporter/.values.yaml'
        sh 'helm repo add prometheus-community https://prometheus-community.github.io/helm-charts'
        sh 'helm upgrade prometheus-redis-exporter -f ./redis-exporter/.values.yaml --namespace monitor ./redis-exporter/prometheus-redis-exporter || helm install prometheus-redis-exporter -f ./redis-exporter/.values.yaml --namespace monitor ./redis-exporter/prometheus-redis-exporter'
      }
    }

    stage('Deploy mysql exporter cluster') {
      when {
        expression { DEPLOY_TARGET == 'true' }
      }
      steps {
        sh 'NODE_SELECTOR_LABEL_KEY=$NODE_SELECTOR_LABEL_KEY NODE_SELECTOR_LABEL_VALUE=$NODE_SELECTOR_LABEL_VALUE envsubst < ./mysql-export/values.yaml > ./mysql-export/.values.yaml'
        sh 'helm repo add prometheus-community https://prometheus-community.github.io/helm-charts'
        sh 'helm upgrade prometheus-mysql-exporter -f ./mysql-export/.values.yaml --namespace monitor ./mysql-export/prometheus-mysql-exporter || helm install prometheus-mysql-exporter -f ./mysql-export/.values.yaml --namespace monitor ./mysql-export/prometheus-mysql-exporter'
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