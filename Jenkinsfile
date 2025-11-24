pipeline {
  agent {
    // LOCAVORA_TODO buildah agent also defined in beckn-registry Jenkinsfile 
    kubernetes {
      label 'jenkins-agent-buildah-remote-harbor'
      idleMinutes 60 // Keep the Pod alive for 60 minutes after the build
      yaml '''
apiVersion: v1
kind: Pod
metadata:
  name: buildah
spec:
  containers:
  - name: buildah
    image: quay.io/buildah/stable:v1.23.1
    command:
    - cat
    tty: true
    securityContext:
      privileged: true
    volumeMounts:
      - name: varlibcontainers
        mountPath: /var/lib/containers
  volumes:
  - name: varlibcontainers
  restartPolicy: Never
'''   
    }
  }
  options {
    buildDiscarder(logRotator(numToKeepStr: '10'))
    durabilityHint('PERFORMANCE_OPTIMIZED')
    disableConcurrentBuilds()
  }
  environment {
    // Jenkins UI -> Manage Jenkins -> Credentials
    IMAGE_REGISTRY_CREDS=credentials('harbor-locavora-readwrite')
  }
  stages {
    // LOCAVORA DELETE_ME - Dockerfile is a subdir - using cd "" && 
    // stage('Buildah build using webserver subdirectory') {
    //   steps {
    //       dir('webserver') { 
    //           echo "Changed directory to webserver"
    //       }
    //   }
    // }
    stage('Build with Buildah using Dockerfile in provided repo') {
      steps {
        container('buildah') {
          sh 'cd webserver && STORAGE_DRIVER=vfs buildah build -t harbor.beckn.locavora.org/locavora/ondc-buyer-app-py-protocol:0.1 .'
        }
      }
    }
    stage('Login to Harbor registry') {
      steps {
        container('buildah') {
          sh 'cd webserver && (echo $IMAGE_REGISTRY_CREDS_PSW | STORAGE_DRIVER=vfs buildah login -u $IMAGE_REGISTRY_CREDS_USR --password-stdin harbor.beckn.locavora.org)'
        }
      }
    }
    stage('tag image') {
      steps {
        container('buildah') {
          sh 'cd webserver && STORAGE_DRIVER=vfs buildah tag harbor.beckn.locavora.org/locavora/ondc-buyer-app-py-protocol:0.1 harbor.beckn.locavora.org/locavora/ondc-buyer-app-py-protocol:latest'
        }
      }
    }
    stage('push image') {
      steps {
        container('buildah') {
          sh 'cd webserver && STORAGE_DRIVER=vfs buildah push harbor.beckn.locavora.org/locavora/ondc-buyer-app-py-protocol:0.1'
          sh 'cd webserver && STORAGE_DRIVER=vfs buildah push harbor.beckn.locavora.org/locavora/ondc-buyer-app-py-protocol:latest'
        }
      }
    }
  }
  post {
    always {
      container('buildah') {
        sh 'STORAGE_DRIVER=vfs buildah logout harbor.beckn.locavora.org'
      }
    }
  }
}