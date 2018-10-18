pipeline {
  agent {
    label 'golang'
  }
  environment {
    APP_NAME = "catalogue"
    ARTEFACT_ID = "sockshop/" + "${env.APP_NAME}"
    VERSION = readFile 'version'
    TAG = "10.31.240.247:5000/library/${env.ARTEFACT_ID}"
    TAG_DEV = "${env.TAG}:dev"
//    TAG_DEV = "${env.TAG}-${env.VERSION}-${env.BUILD_NUMBER}"
    TAG_STAGING = "${env.TAG}-${env.VERSION}"
  }
  stages {
    stage('Go build') {
      steps {
        checkout scm
        container('golang') {
          sh 'go version && glide -v'
          sh 'mkdir -p src/github.com/dynatrace-sockshop/catalogue/'

          sh 'cp -R ./api src/github.com/dynatrace-sockshop/catalogue/'
          sh 'cp -R ./main.go src/github.com/dynatrace-sockshop/catalogue/'
          sh 'cp -R ./glide.* src/github.com/dynatrace-sockshop/catalogue/'

          sh 'export GOPATH=$PWD && cd src/github.com/dynatrace-sockshop/catalogue && glide install && go build -a -ldflags -linkmode=external -installsuffix cgo -o $GOPATH/catalogue main.go'
        }
      }
    }
    stage('Docker build') {
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
        }
      }
      steps {
        container('docker') {
          echo "branch_name=${env.BRANCH_NAME}"

          sh "docker build -t ${env.TAG_DEV} ."
        }
      }
    }
    stage('Docker push to registry'){
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
        }
      }
      steps {
        container('docker') {
          sh "docker push ${env.TAG_DEV}"
        }
      }
    }
    stage('Deploy to dev namespace') {
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
        }
      }
      steps {
        container('kubectl') {
          sh "kubectl -n dev apply -f manifest/catalogue.yml"
        }
      }
    }
    stage('Run health check in dev') {
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
        }
      }
      steps {
        sleep 60

        build job: "jmeter-tests",
          parameters: [
            string(name: 'SCRIPT_NAME', value: 'basiccheck.jmx'),
            string(name: 'SERVER_URL', value: "${env.APP_NAME}.dev"),
            string(name: 'SERVER_PORT', value: '80'),
            string(name: 'CHECK_PATH', value: '/health'),
            string(name: 'VUCount', value: '1'),
            string(name: 'LoopCount', value: '1'),
            string(name: 'DT_LTN', value: "HealthCheck_${BUILD_NUMBER}"),
            string(name: 'FUNC_VALIDATION', value: 'yes'),
            string(name: 'AVG_RT_VALIDATION', value: '0'),
            string(name: 'RETRY_ON_ERROR', value: 'yes')
          ]
      }
    }
    stage('Run functional check in dev') {
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
        }
      }
      steps {
        build job: "jmeter-tests",
          parameters: [
            string(name: 'SCRIPT_NAME', value: "${env.APP_NAME}_load.jmx"),
            string(name: 'SERVER_URL', value: "${env.APP_NAME}.dev"),
            string(name: 'SERVER_PORT', value: '80'),
            string(name: 'CHECK_PATH', value: '/health'),
            string(name: 'VUCount', value: '1'),
            string(name: 'LoopCount', value: '1'),
            string(name: 'DT_LTN', value: "FuncCheck_${BUILD_NUMBER}"),
            string(name: 'FUNC_VALIDATION', value: 'yes'),
            string(name: 'AVG_RT_VALIDATION', value: '0')
          ]
      }
    }
    stage('Mark artifact for staging namespace') {
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*'
        }
      }
      steps {
        container('docker'){
          sh "docker tag ${env.TAG_DEV} ${env.TAG_STAGING}"
        }
      }
    }
    stage('Deploy to staging') {
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*'
        }
      }
      agent {
        label 'git'
      }
      steps {
        echo "update sockshop deployment yaml for staging -> github webhook triggers deployment to staging"
        echo "apply sockshop deployment yaml to staging environment"
      }
    }
  }
}
