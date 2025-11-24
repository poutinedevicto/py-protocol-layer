
def REGISTRY_NAME = "harbor.beckn.locavora.org"
def IMAGE_NAME = "locavora-public/ondc-buyer-app-py-protocol"

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
    resources:
      requests:
        ephemeral-storage: 3Gi
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
          sh 'cd webserver && STORAGE_DRIVER=vfs buildah build -t REGISTRY_NAME/IMAGE_NAME:0.1 .'
        }
      }
    }
    stage('Login to Harbor registry') {
      steps {
        container('buildah') {
          sh 'cd webserver && (echo $IMAGE_REGISTRY_CREDS_PSW | STORAGE_DRIVER=vfs buildah login -u $IMAGE_REGISTRY_CREDS_USR --password-stdin REGISTRY_NAME)'
        }
      }
    }
    stage('tag image') {
      steps {
        container('buildah') {
          sh 'cd webserver && STORAGE_DRIVER=vfs buildah tag REGISTRY_NAME/IMAGE_NAME:0.1 REGISTRY_NAME/IMAGE_NAME:latest'
        }
      }
    }
    stage('push image') {
      steps {
        container('buildah') {
          sh 'cd webserver && STORAGE_DRIVER=vfs buildah push REGISTRY_NAME/IMAGE_NAME:0.1'
          sh 'cd webserver && STORAGE_DRIVER=vfs buildah push REGISTRY_NAME/IMAGE_NAME:latest'
        }
      }
    }
  }
  post {
    always {
      container('buildah') {
        sh 'STORAGE_DRIVER=vfs buildah logout REGISTRY_NAME'
      }
    }
  }
}