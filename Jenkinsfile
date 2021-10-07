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
          if (!fileExists("/usr/bin/cfssl")) {
            sh 'curl -sL https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 -o /usr/bin/cfssl'
            sh 'chmod a+x /usr/bin/cfssl'
          }

          if (!fileExists("/usr/bin/cfssljson")) {
            sh 'curl -sL https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 -o /usr/bin/cfssljson'
            sh 'chmod a+x /usr/bin/cfssljson'
          }

          if (!fileExists("/usr/bin/cfssl-certinfo")) {
            sh 'curl -sL https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64 -o /usr/bin/cfssl-certinfo'
            sh 'chmod a+x /usr/bin/cfssl-certinfo'
          }

          if (!fileExists("/usr/bin/consul")) {
            sh 'mkdir -p $HOME/.consul'
            if (!fileExists("$HOME/.consul/.consul-src")) {
              sh 'git clone https://github.com/hashicorp/consul.git $HOME/.consul/.consul-src'
            }
            sh 'cd $HOME/.consul/.consul-src; make tools; make dev; cp ./bin/consul /usr/bin/consul'
            sh 'consul -v'
          }

          if (!fileExists("/usr/bin/helm")) {
            sh 'mkdir -p $HOME/.helm'
            if (!fileExists("$HOME/.helm/.helm-src")) {
              sh 'git clone https://github.com/helm/helm.git $HOME/.helm/.helm-src'
            }
            sh 'cd $HOME/.helm/.helm-src; make; cp bin/helm /usr/bin/helm'
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
        sh 'helm install consul hashicorp/consul --namespace kube-system --set server.storage=1Gi,server.storageClass=local-storage,global.name=consul,client.enabled=false,server.replicas=1,server.bootstrapExpect=1'
      }
    }

    stage('Deploy consul with helm to development') {
      when {
        expression { DEPLOY_TARGET == 'true' }
        expression { TARGET_ENV != 'development' }
      }
      steps {
        sh 'helm install consul hashicorp/consul --namespace kube-system --set server.storage=10Gi,server.storageClass=local-storage,global.name=consul,client.enabled=false'
      }
    }

    stage('Deploy ingress to target') {
      when {
        expression { DEPLOY_TARGET == 'true' }
      }
      steps {
        sh 'sed -i "s/consul.internal-devops.development.npool.top/consul.internal-devops.$TARGET_ENV.npool.top/g" 01-ingress.yaml'
        sh 'kubectl apply -f 01-ingress.yaml'
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
