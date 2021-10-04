pipeline {
  agent any
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
            curl -sL https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 -o /usr/bin/cfssl
            chmod a+x /usr/bin/cfssl
          }

          if (!fileExists("/usr/bin/cfssljson")) {
            curl -sL https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 -o /usr/bin/cfssljson
            chmod a+x /usr/bin/cfssljson
          }

          if (!fileExists("/usr/bin/cfssl-certinfo")) {
            curl -sL https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64 -o /usr/bin/cfssl-certinfo
            chmod a+x /usr/bin/cfssl-certinfo
          }
        }
      }
    }

    stage('Prepare kubernetes consul') {
      steps {
        sh 'git clone https://github.com/kelseyhightower/consul-on-kubernetes.git .consul-on-kubernetes'
        sh 'cd .consul-on-kubernetes; mkdir -p $HOME/$TARGET_ENV/ca; cfssl gencert -initca ca/ca-csr.json | cfssljson -bare ca'
        sh 'cfssl gencert -ca=$HOME/$TARGET_ENV/ca/ca.pem -ca-key=$HOME/$TARGET_ENV/ca/ca-key.pem -config=ca/ca-config.json -profile=default ca/consul-csr.json | cfssljson -bare consul'
        sh(returnStdout: true, script: '''
          GOSSIP_ENCRYPTION_KEY=`consul keygen`
          kubectl create secret generic consul --from-literal="gossip-encryption-key=$GOSSIP_ENCRYPTION_KEY" --from-file=$HOME/$TARGET_ENV/ca/ca.pem --from-file=$HOME/$TARGET_ENV/ca/consul.pem --from-file=$HOME/$TARGET_ENV/ca/consul-key.pem
        '''.stripIndent())
        sh 'kubectl create configmap consul --from-file=configs/server.json'
      }
    }

    stage('Deploy to target') {
      when {
        expression { DEPLOY_TARGET == 'true' }
      }
      steps {
        sh 'sed -i "s/consul.internal-devops.development.npool.top/consul.internal-devops.$TARGET_ENV.npool.top/g" 01-ingress.yaml'
        sh 'cd /etc/kubeasz; ./ezctl checkout $TARGET_ENV'
        sh 'kubectl apply -k .'
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
