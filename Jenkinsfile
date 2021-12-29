pipeline {
    agent {
        kubernetes {
            defaultContainer 'kaniko'
            yaml """
apiVersion: v1
kind: Pod
metadata:
  name: kaniko
spec:
  volumes:
  - name: shared-data
    emptyDir: {}
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:v1.7.0-debug
    imagePullPolicy: Always
    command:
    - /busybox/cat
    tty: true
    volumeMounts:
    - name: shared-data
      mountPath: /shared-data
  - name: alpine
    image: alpine
    imagePullPolicy: Always
    command:
    - /bin/cat
    tty: true
    volumeMounts:
    - name: shared-data
      mountPath: /shared-data
"""
        }
    }
    environment {
        IMAGE_NAMESPACE="registry.relizahub.com/cea2e96e-a936-4928-99f2-4cea5c0d4e0b-public"
        IMAGE_NAME="mafia-vue"
        RELIZA_API=credentials('RELIZA_API')
    }
    stages {
        stage('Build with Kaniko') {
            steps {
                script {
                    checkout scm
                    container(name: 'alpine') {
                        sh 'apk add git'
                    }
                    env.COMMIT_TIME = container(name: 'alpine') {
                        sh(script: 'git log -1 --date=iso-strict --pretty="%ad"', returnStdout: true).trim()
                    }
                    withReliza(projectId: '28c3735d-a810-4a3f-9e3a-2a43932589b1') {
                        if (env.LATEST_COMMIT) {
                            env.COMMIT_LIST = getCommitListWithLatest()
                        } else {
                            env.COMMIT_LIST = getCommitListNoLatest()
                        }
                        echo 'Here should go env vars'
                        echo env.COMMIT_TIME
                        echo env.LATEST_COMMIT
                        echo env.COMMIT_LIST
                        echo env.GIT_BRANCH
                        echo 'end of env vars'
                        if (!env.LATEST_COMMIT || env.COMMIT_LIST) {
                            try {
                                container(name: 'kaniko', shell: '/busybox/sh') {
                                    withCredentials([file(credentialsId: 'docker-credentials', variable: 'DOCKER_CONFIG_JSON')]) {
                                        withEnv(['PATH+EXTRA=/busybox']) {
                                            sh '''#!/busybox/sh
                                                cp $DOCKER_CONFIG_JSON /kaniko/.docker/config.json
                                                /kaniko/executor --context `pwd` --destination "$IMAGE_NAMESPACE/$IMAGE_NAME:latest" --digest-file=/shared-data/termination-log
                                            '''
                                        }
                                    }
                                }
                            } catch (Exception e) {
                                env.STATUS = 'rejected'
                                echo 'FAILED BUILD: ' + e.toString()
                            }
                            echo 'build completed, before sha256 extraction'
                            env.SHA_256 = container(name: 'alpine') {
                                sh(script: 'cat /shared-data/termination-log', returnStdout: true).trim()
                            }
                            echo 'SHA 256 digest of our container'
                            echo env.SHA_256
                            addRelizaRelease(artId: "$IMAGE_NAMESPACE/$IMAGE_NAME", artType: "Docker")
                        } else {
                            echo 'Repeated build, skipping push'
                        }
                    }
                }
            }
        }
    }
}

String getCommitListNoLatest() {
  if (env.GIT_PREVIOUS_SUCCESSFUL_COMMIT) {
    return container(name: 'alpine') {
        sh(script: 'git log $GIT_PREVIOUS_SUCCESSFUL_COMMIT..$GIT_COMMIT --date=iso-strict --pretty="%H|||%ad|||%s" | base64 -w 0', returnStdout: true).trim()
    }   
  } else {
    return container(name: 'alpine') {
        sh(script: 'git log -1 --date=iso-strict --pretty="%H|||%ad|||%s" | base64 -w 0', returnStdout: true).trim()
    }
  }
}

String getCommitListWithLatest() {
  return container(name: 'alpine') {
      sh(script: 'git log $LATEST_COMMIT..$GIT_COMMIT --date=iso-strict --pretty="%H|||%ad|||%s" | base64 -w 0', returnStdout: true).trim()
  }
}