pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod 
spec:
    containers:
      - name: git
        image: harbor.devops.tanzu-altair.com/jenkins/git:v1
        command:
        - sleep
        args:
        - infinity
      - name: docker-pack
        image: harbor.devops.tanzu-altair.com/jenkins/pack:v1
        resources:
          requests:
            memory: "2Gi"
        command:
        - sleep
        args:
        - infinity
        env:
          - name: DOCKER_HOST
            value: tcp://localhost:2375
      - name: docker-daemon
        image: harbor.devops.tanzu-altair.com/jenkins/daemon:v1
        resources:
          requests:
            memory: "2Gi"
        env:
          - name: DOCKER_TLS_CERTDIR
            value: ""
        securityContext:
            privileged: true
        volumeMounts:
          - name: docker-cache
            mountPath: /var/lib/docker
    volumes:
      - name: docker-cache
        emptyDir: {}
"""
        }
    }
    
    environment {
      // Credentials
		// GitLAB
        GIT_CREDENTIALS_ID = ""
        GIT_URL = "" // http:// 또는 https:// 제거		
        BRANCH ="main"
        
        // Harbor
        HARBOR_CREDENTIALS = credentials('')
        HARBOR_URL = ""
        HARBOR_PROJECT = "apps"

        // Build
        BUILD_LANG = ""
        BUILD_LANG_DIR = ""
        BUILDPACK = "paketobuildpacks/builder:base"
        BUILD_JVM_JDK = "17"
        BUILD_ARG = "-Dcheckstyle.skip -Dmaven.test.skip=true --no-transfer-progress"
        
        // App
        IMAGE_NAME = "spring-petclinic"
        APP_NAMESPACE = "default"
        CLUSTER_01 = "devops"
        CLUSTER_02 = "01-workload"
        CLUSTER_03 = "02-workload"
        DOMAIN = "tanzu-altair.com"
        SERVICE_PORT = "8080"
        TARGET_PORT = "8080"
        
        // Who
        EMAIL = "altair@megazone.com"
        NAME = "Altair"
        //-XX:MaxRAM8g -Xmx4g -XX:+UseParallelGC -XX:GCTimeRatio=4 -XX:AdaptiveSizePolicyWeight=90 -XX:MinHeapFreeRatio=20 -XX:MaxHeapFreeRatio=40
    }
    
    stages {
		stage('00. Time Set') {
            steps {
                container('git') {
                    echo '=== Global Preset ==='
                    script {
                        try {
                            def now = new Date()
                            CREATED = now.format("yyyy-MM-dd:HH.mm.ss", TimeZone.
                            getTimeZone('Asia/Seoul')).replace(':', 'T')
                            NEW_VERSION="${CREATED}"
                            env.preambleResult = true
                            echo "${NEW_VERSION}"
                            } 
                        catch(Exception e) {
                            cleanWs()
                            PrintErrorMessage(e)
                            currentBuild.result = 'FAILURE'
                        }
                    }
                }
            }
        }
        
        stage("01. Source") {
            steps {
                container('git') {
                    echo '=== Git Clone Source Code ==='
				    script {
					    try {
					        git credentialsId: "${GIT_CREDENTIALS_ID}", url: "http://${GIT_URL}", branch: "${BRANCH}", poll: true, changelog: true
					        env.gitcloneResult = true
					    }
					    catch(Exception e) {
                            cleanWs()
                            PrintErrorMessage(e)
                            currentBuild.result = 'FAILURE'
					    }	
				    }
                }
			}
		}
        
        stage('02. Build') {
            steps {
                container('docker-pack') {  
                    script {
                        sh 'pack config default-builder ${BUILDPACK}'
                        sh ('''
                        pack build ${HARBOR_URL}/${HARBOR_PROJECT}/${IMAGE_NAME}:$BUILD_NUMBER --env BP_JVM_VERSION=${BUILD_JVM_JDK} --env BP_GRADLE_BUILD_ARGUMENTS='--no-daemon assemble --parallel' --env JAVA_TOOL_OPTIONS='-Xmx4g -Xms4g' --env BPL_JVM_THREAD_COUNT=100 
                        ''')
                    }
                }
            }
        }

        stage('03. Login') {
            steps {
                container('docker-pack') {  
                    script {
                        sh 'echo $HARBOR_CREDENTIALS_PSW | docker login ${HARBOR_URL} -u $HARBOR_CREDENTIALS_USR --password-stdin'
                    }
                }
            }
        }
        
        stage('04. Harbor') {
            steps {
                container('docker-pack') {  
                    script {
                        sh 'docker push ${HARBOR_URL}/${HARBOR_PROJECT}/${IMAGE_NAME}:$BUILD_NUMBER'
                    }
                }
            }
        }
        
        stage('05. YAML') {
            steps {
                sh ('''
                cat <<EOF > Manifest/Deployment.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${IMAGE_NAME}
  labels:
    app: ${IMAGE_NAME}
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ${IMAGE_NAME}
  template:
    metadata:
      labels:
        app: ${IMAGE_NAME}
    spec:
      containers:
      - name: ${IMAGE_NAME}
        image: ${HARBOR_URL}/${HARBOR_PROJECT}/${IMAGE_NAME}:${BUILD_NUMBER}
EOF
                ''')
                sh ('''
                cat <<EOF > Manifest/Service.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: ${IMAGE_NAME}-svc
spec:
  selector:
    app: ${IMAGE_NAME}
  ports:
  - port: ${SERVICE_PORT}
    targetPort: ${TARGET_PORT}
    protocol: TCP
  type: ClusterIP
EOF
                ''')
                sh ('''
                cat <<EOF > Manifest/HTTPproxy.yaml
---
apiVersion: projectcontour.io/v1
kind: HTTPProxy
metadata:
  annotations:
  labels:
    app: ${IMAGE_NAME}
  name: ${IMAGE_NAME}-${CLUSTER_01}-httpproxy
spec:
  routes:
  - conditions:
    - prefix: /
    services:
    - name: ${IMAGE_NAME}-svc
      port: ${SERVICE_PORT}
  virtualhost:
    fqdn: ${IMAGE_NAME}.${CLUSTER_01}.${DOMAIN}
---
apiVersion: projectcontour.io/v1
kind: HTTPProxy
metadata:
  annotations:
  labels:
    app: ${IMAGE_NAME}
  name: ${IMAGE_NAME}-${CLUSTER_02}-httpproxy
spec:
  routes:
  - conditions:
    - prefix: /
    services:
    - name: ${IMAGE_NAME}-svc
      port: ${SERVICE_PORT}
  virtualhost:
    fqdn: ${IMAGE_NAME}.${CLUSTER_02}.${DOMAIN}
---
apiVersion: projectcontour.io/v1
kind: HTTPProxy
metadata:
  annotations:
  labels:
    app: ${IMAGE_NAME}
  name: ${IMAGE_NAME}-${CLUSTER_03}-httpproxy
spec:
  routes:
  - conditions:
    - prefix: /
    services:
    - name: ${IMAGE_NAME}-svc
      port: ${SERVICE_PORT}
  virtualhost:
    fqdn: ${IMAGE_NAME}.${CLUSTER_03}.${DOMAIN}
---
apiVersion: projectcontour.io/v1
kind: HTTPProxy
metadata:
  annotations:
  labels:
    app: ${IMAGE_NAME}
  name: ${IMAGE_NAME}-httpproxy
spec:
  routes:
  - conditions:
    - prefix: /
    services:
    - name: ${IMAGE_NAME}-svc
      port: ${SERVICE_PORT}
  virtualhost:
    fqdn: ${IMAGE_NAME}.${DOMAIN}
EOF
                ''')
            }
        }
        
        stage('06. GitLab') {
            steps {
                    withCredentials([usernamePassword(credentialsId: "${GIT_CREDENTIALS_ID}", passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                        sh("git config --global user.email ${EMAIL}")
                        sh("git config --global user.name ${NAME}")
                        sh('git add .')
                        sh("git commit -m '${IMAGE_NAME}  # $BUILD_NUMBER'")
                        sh('git push http://${GIT_USERNAME}:${GIT_PASSWORD}@${GIT_URL}')
                    }
            }
        }
        
    }
}
