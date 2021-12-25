podTemplate(
  showRawYaml: false,
  yaml: """
    apiVersion: v1
    kind: Pod
    metadata:
      namespace: jenkins
    spec:
      serviceAccountName: jenkins
      volumes:
        - name: m2-repo
          persistentVolumeClaim:
            claimName: jenkins-pvc
      
      containers:
        - name: maven
          image: maven:3-alpine
          imagePullPolicy: IfNotPresent
          command:
            - /bin/bash
            - -c
          args:
            - cat
          tty: true
          workingDir: /home/jenkins/agent
          volumeMounts:
            - name: m2-repo
              mountPath: /root/.m2
              subPath: m2
        
        - name: jnlp
          image: jenkins/inbound-agent
          imagePullPolicy: IfNotPresent
          tty: true
          workingDir: /home/jenkins/agent
          env:
            - name: JENKINS_URL
              value: 'http://jenkins-svc:8080/jenkins'
            - name: JENKINS_TUNNEL
              value: 'jenkins-svc:50000'
  """
){
  //pipeline code
  node(POD_LABEL){
    container('maven'){
      notify('Started')
      stage 'CheckOut Code'
      checkout scm

      stage 'Build code'
      sh 'mvn clean package'

      stage 'Publish Artifacts'
        archiveArtifacts allowEmptyArchive: true, artifacts: 'target/*.war', followSymlinks: false, onlyIfSuccessful: true

        stash includes: 'target/*.war, Dockerfile', name: 'petclinic-build-stash'
        stash includes: 'kube-petclinic.yaml', name: 'kube-petclinic'

      stage 'Publish Test Results'
      junit skipPublishingChecks: true, allowEmptyResults: true, testResults: 'target/surefire-reports/*.xml'

      notify('Completed')
    }
  }
}

podTemplate(
  showRawYaml: false,
  yaml: """
    apiVersion: v1
    kind: Pod
    metadata:
      namespace: jenkins
    spec:
      serviceAccountName: jenkins
      volumes:
        - name: docker-sock
          hostPath:
            path: /var/run/docker.sock
      
      containers:
        - name: jnlp
          image: jenkins/inbound-agent
          imagePullPolicy: IfNotPresent
          tty: true
          workingDir: /home/jenkins/agent
          env:
            - name: JENKINS_URL
              value: 'http://jenkins-svc:8080/jenkins'
            - name: JENKINS_TUNNEL
              value: 'jenkins-svc:50000'
        
        - name: docker
          image: docker:20
          imagePullPolicy: IfNotPresent
          tty: true
          workingDir: /home/jenkins/agent
          volumeMounts:
            - name: docker-sock
              mountPath: /var/run/docker.sock
  """
){
  node(POD_LABEL){
    container('docker'){
      stage 'Create Image'
      unstash 'petclinic-build-stash'
      sh 'docker build -t hirendrakoche/petclinic:$BUILD_NUMBER .'

      withDockerRegistry(credentialsId: 'docker-hub-user') {
        sh 'docker push hirendrakoche/petclinic:$BUILD_NUMBER'
      }
    }
  }
}


podTemplate(
  showRawYaml: false,
  yaml: """
    apiVersion: v1
    kind: Pod
    metadata:
      namespace: jenkins
    spec:
      serviceAccountName: jenkins
      volumes:
        - name: kube-vol
          hostPath: 
            path: /root/.kube
      
      containers: 
        - name: jnlp
          image: jenkins/inbound-agent
          imagePullPolicy: IfNotPresent
          tty: true
          workingDir: /home/jenkins/agent
          env:
            - name: JENKINS_URL
              value: 'http://jenkins-svc:8080/jenkins'
            - name: JENKINS_TUNNEL
              value: 'jenkins-svc:50000'
        
        - name: kubectl
          image: kubectl:lts
          imagePullPolicy: IfNotPresent
          tty: true
          workingDir: /home/jenkins/agent
          volumeMounts:
            - name: kube-vol
              mountPath: /root/.kube
  """
){  
  node(POD_LABEL){
    container('kubectl'){
      stage 'Deploy Image'
      notify('ReadToDeploy')
      input "Deploy?"
      unstash 'kube-petclinic'
      sh 'sed -i "s|BUILD|$BUILD_NUMBER|g" kube-petclinic.yaml'
      sh 'kubectl apply -f kube-petclinic.yaml'
      notify('Deployed')
    }
  }
}

def notify(STATUS) {
    emailext body: """<p>Check console output at &QUOT;<a href='${env.BUILD_URL}/console'>${env.JOB_NAME} [${env.BUILD_NUMBER}]</a>&QUOT;</p>""", subject: """$JOB_NAME - #$BUILD_NUMBER : $STATUS""", to: 'hirendrakoche1@outlook.com'
}