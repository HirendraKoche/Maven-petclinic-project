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
    container('maven'){
      stage 'Download repo'  
      checkout scm
      stage 'Build code'
      sh 'mvn clean package'
    }

    container('docker'){
      stage 'Create Image'
      def githash = sh(returnStdout: true, script: 'git rev-parse --short=6 HEAD').trim()
      def imageTag = "hirendrakoche/petclinic:${gihash}"
      echo ${imageTag}
      def isImageExist = sh(returnStdout: true, script: 'docker image inspect ${imageTag} --format="true" 2> /dev/null').trim()
      if (isImageExist == "true"){
        println ("Docker Image exist: %s", ${imageTag})
      }else {
        println "Docker Image Not exist. "
      }
    }
  }
}