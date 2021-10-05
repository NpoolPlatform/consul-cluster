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

    stage('Check cfssl tools') {
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
        }
      }
    }

    stage('Prepare kubernetes consul') {
      steps {
        sh(returnStdout: true, script: '''
          cd k8s

          set +e
          kubectl get secret | grep consul
          rc=$?
          set -e

          if [ ! $rc -eq 0 ]; then
            cfssl gencert -initca ca/ca-csr.json | cfssljson -bare ca

            mv ca.pem $HOME/.consul/$TARGET_ENV/ca
            mv ca-key.pem $HOME/.consul/$TARGET_ENV/ca
            mv ca.csr $HOME/.consul/$TARGET_ENV/ca

            cfssl gencert -ca=$HOME/.consul/$TARGET_ENV/ca/ca.pem -ca-key=$HOME/.consul/$TARGET_ENV/ca/ca-key.pem -config=ca/ca-config.json -profile=default ca/consul-csr.json | cfssljson -bare consul

            mv consul.pem $HOME/.consul/$TARGET_ENV/ca
            mv consul-key.pem $HOME/.consul/$TARGET_ENV/ca

            GOSSIP_ENCRYPTION_KEY=`consul keygen`
            kubectl create secret generic consul --from-literal="gossip-encryption-key=$GOSSIP_ENCRYPTION_KEY" --from-file=$HOME/.consul/$TARGET_ENV/ca/ca.pem --from-file=$HOME/.consul/$TARGET_ENV/ca/consul.pem --from-file=$HOME/.consul/$TARGET_ENV/ca/consul-key.pem

            kubectl delete configmap consul || true
          fi

          set +e
          kubectl get configmap | grep consul
          rc=$?
          set -e

          if [ ! $rc -eq 0 ]; then
            kubectl create configmap consul --from-file=configs/server.json
          fi
        '''.stripIndent())
      }
    }

    stage('Deploy to target') {
      when {
        expression { DEPLOY_TARGET == 'true' }
      }
      steps {
        sh 'sed -i "s/consul.internal-devops.development.npool.top/consul.internal-devops.$TARGET_ENV.npool.top/g" 01-ingress.yaml'
        sh 'cd /etc/kubeasz; ./ezctl checkout $TARGET_ENV'
        sh 'kubectl apply -k k8s'
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
