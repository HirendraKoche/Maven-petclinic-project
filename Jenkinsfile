def githash = ""
def dockerRepo = "hirendrakoche/petclinic"
def dockerRepoURL = "https://hub.docker.com/v2/repositories/${dockerRepo}/tags"

podTemplate(
  showRawYaml: false,
  yaml: """
    apiVersion: v1
    kind: Pod
    metadata:
      namespace: jenkins
    spec:
      volumes:
        - name: m2-repo
          persistentVolumeClaim:
            claimName: jenkins-pvc
        - name: docker-sock
          hostPath:
            path: /var/run/docker.sock
      securityContext:
        runAsUser: 0
        runAsGroup: 0
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
  //pipeline code
  node(POD_LABEL){
    stage("Git Pull") {
      checkout scm
      githash = sh(returnStdout: true, script: "git rev-parse --short=6 HEAD").trim()
    }
    
    container('maven'){
      stage("Maven Build"){
        sh 'mvn clean package'
      }
    }

    container('docker'){
      stage("Create Image") {
        def imageTag = "${dockerRepo}:${githash}"
        withDockerRegistry(credentialsId: 'docker-hub-user') {
          sh """
            #!/bin/bash
            apk add curl
            if [ \$(curl --silent -f -lSL ${dockerRepoURL}/${githash} 2> /dev/null) ]; then
              echo "Image already exist at dockerhub: " ${imageTag}
            else
              docker image build -t ${imageTag} .
              docker push ${imageTag}
            fi
          """
        }
      }
    }
  }
}