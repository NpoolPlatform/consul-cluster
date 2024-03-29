pipeline {
  agent any
  tools {
    go 'go'
  }
  environment {
    GOPROXY = 'https://goproxy.cn,direct'
  }
  stages {
    stage('Clone consul cluster') {
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

    stage('Deploy consul with helm to development') {
      when {
        expression { DEPLOY_TARGET == 'true' }
        expression { TARGET_ENV == 'development' }
      }
      steps {
        sh 'helm repo add hashicorp https://helm.releases.hashicorp.com'
        sh 'helm upgrade consul hashicorp/consul --values values.yaml --namespace kube-system --version "0.41.0" --set server.storage=4Gi,global.name=consul,client.enabled=false,server.replicas=1,server.bootstrapExpect=1,dns.enabled=false || helm install consul hashicorp/consul --values values.yaml --namespace kube-system --version "0.41.0" --set server.storage=4Gi,global.name=consul,client.enabled=false,server.replicas=1,server.bootstrapExpect=1,dns.enabled=false'
      }
    }

    stage('Deploy consul with helm to production or testing') {
      when {
        expression { DEPLOY_TARGET == 'true' }
        anyOf {
          expression { TARGET_ENV != 'development' }
        }
      }
      steps {
        sh 'helm repo add hashicorp https://helm.releases.hashicorp.com'
        sh 'helm upgrade consul hashicorp/consul --values values.yaml --namespace kube-system --version "0.41.0" --set server.storage=4Gi,global.name=consul,client.enabled=false,server.replicas=1,server.bootstrapExpect=1,dns.enabled=false || helm install consul hashicorp/consul --values values.yaml --namespace kube-system --version "0.41.0" --set server.storage=4Gi,global.name=consul,client.enabled=false,server.replicas=1,server.bootstrapExpect=1,dns.enabled=false'
        // sh 'helm install consul hashicorp/consul --namespace kube-system --set server.storage=10Gi,global.name=consul,client.enabled=false,dns.enabled=false'
      }
    }

    stage('Deploy ingress to target') {
      when {
        expression { DEPLOY_TARGET == 'true' }
      }
      steps {
        sh 'sed -i "s/consul.development.npool.top/consul.$TARGET_ENV.npool.top/g" 01-traefik-vpn-ingress.yaml'
        sh 'kubectl apply -f 01-traefik-vpn-ingress.yaml'
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
