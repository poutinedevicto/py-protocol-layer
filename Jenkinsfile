// LOCAVORA DELETE_ME - older Jenkinsfile using buildah instead of kaniko
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
    image: quay.io/buildah/stable:latest
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
    CACHE_NAME = 'locavora/dockerfile-build-cache'
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
    stage('Login to Harbor registry') {
      steps {
        container('buildah') {
          sh 'echo $IMAGE_REGISTRY_CREDS_PSW | buildah login -u $IMAGE_REGISTRY_CREDS_USR --password-stdin $REGISTRY_NAME'
        }
      }
    }
    stage('Build with Buildah using Dockerfile in provided repo') {
      steps {
        container('buildah') {
          // LOCAVORA - STORAGE_DRIVER=vfs needed if fuse not supported in kernel (lsmod | grep fuse)
          // IMPORTANT --layers to enable layer build caching
          //           --from-cache=$CACHE_NAME to use cache from previous builds
          //           --to-cache=$CACHE_NAME to save cache for future builds
          //     --retry 0 should disable retries to pull non-existen cached layers, but does not 
          //     work as of dec 2025 - see https://www.hostedredmine.com/projects/mutualisation/wiki/JenkinsMaven_dans_k8s#Removing-retries-in-pushing-Image-layer-to-cache
          
          sh 'cd webserver && \
              nice buildah build --layers \
              --cache-from $REGISTRY_NAME/$CACHE_NAME \
              --cache-to $REGISTRY_NAME/$CACHE_NAME \
              -t $REGISTRY_NAME/$IMAGE_NAME:0.1 .'
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