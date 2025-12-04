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
    REGISTRY_NAME = 'harbor.beckn.locavora.org'
    IMAGE_NAME = 'locavora-public/ondc-buyer-app-py-protocol'
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

    // LOCAVORA - attention on doit utiliser les apos (') pour que les variables d'environnement ne soient pas interprétées
    //  Warning: A secret was passed to "sh" using Groovy String interpolation, which is insecure.
		//  Affected argument(s) used the following variable(s): [xxx]
	  //  See https://jenkins.io/redirect/groovy-string-interpolation for details.
    stage('Build with Buildah using Dockerfile in provided repo') {
      steps {
        container('buildah') {
          // LOCAVORA - STORAGE_DRIVER=vfs needed if fuse not supported in kernel (lsmod | grep fuse)
          sh 'cd webserver && buildah build -t $REGISTRY_NAME/$IMAGE_NAME:0.1 .'
        }
      }
    }
    stage('Login to Harbor registry') {
      steps {
        container('buildah') {
          sh 'echo $IMAGE_REGISTRY_CREDS_PSW | buildah login -u $IMAGE_REGISTRY_CREDS_USR --password-stdin $REGISTRY_NAME'
        }
      }
    }
    stage('tag image') {
      steps {
        container('buildah') {
          sh 'buildah tag $REGISTRY_NAME/$IMAGE_NAME:0.1 $REGISTRY_NAME/$IMAGE_NAME:latest'
        }
      }
    }
    stage('push image') {
      steps {
        container('buildah') {
          sh 'buildah push $REGISTRY_NAME/$IMAGE_NAME:0.1'
          sh 'buildah push $REGISTRY_NAME/$IMAGE_NAME:latest'
        }
      }
    }
  }
  post {
    always {
      container('buildah') {
        sh 'buildah logout $REGISTRY_NAME'
      }
    }
  }
}