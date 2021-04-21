properties([
    parameters([
        string(defaultValue: 'itops-cast', description: 'account name for ADO service', name: 'ACCOUNT', trim: false),

        string(defaultValue: 'arn:aws:iam::478919403635:role/cbc-itops-cast', description: 'pod IAM role', name: 'CBJ_ROLE', trim: false),

        string(defaultValue: 'arn:aws:iam::460220756400:role/cast-mgmt-cb-jenkins', description: 'IAM role to be assumed in CRE DEV AWS account', name: 'CRE_DEV_ADO_ROLE', trim: false),

        string(defaultValue: 'arn:aws:iam::143454207788:role/cre-prod-mgmt-cbj-jenkins ', description: 'IAM role to be assumed in CRE PROD AWS account', name: 'CRE_PROD_ADO_ROLE', trim: false),


        choice(choices: ['460220756400'], description: 'CRE DEV Account Number', name: 'CRE_DEV_ACCTNUM', trim: false),
        choice(choices: ['us-west-2'], description: 'CRE DEV Account Region', name: 'CRE_DEV_REGION', trim: false),

        choice(choices: ['143454207788'], description: 'CRE PROD Account Number', name: 'CRE_PROD_ACCTNUM', trim: false),
        choice(choices: ['us-west-2'],  description: 'CRE PROD Account Region', name: 'CRE_PROD_REGION', trim: false),

        choice(choices: ['dev', 'prod'], description: 'application environment?', name: 'ENV', trim: false),


        choice(choices: ['campapp', 'campdb', 'campdb-migrator'], description: 'application REPO NAME?', name: 'REPO_NAME', trim: false),
        choice(choices: ['reactnodeapp', 'postgrest', 'migrator'], description: 'ECS Service Name?', name: 'ECS_SERVICE_NAME', trim: false),
        booleanParam (name: 'XRAY_SCAN', defaultValue: false, description: 'Scan artifacts using Xray')
    ])
])

pipeline {

  agent {
    kubernetes {
      label 'campappbuildnode'
      yaml """
      apiVersion: v1
      kind: Pod
      spec:
        serviceAccountName: ${params.ACCOUNT}
        restartPolicy: Never
        containers:
        - name: docker-daemon
          image: docker:19.03.1-dind
          securityContext:
            privileged: true
          env:
            - name: DOCKER_TLS_CERTDIR
              value: ""
            - name: DOCKER_DRIVER
              value: overlay2
            - name: AWS_WEB_IDENTITY_TOKEN_FILE
              value: "/var/run/secrets/eks.amazonaws.com/serviceaccount/token"
            - name: AWS_ROLE_ARN
              value: ${params.CBJ_ROLE}
        - name: jfrogcli
          image: docker.bintray.io/jfrog/jfrog-cli-go
          command: ['cat']
          tty: true
          env:
            - name: DOCKER_HOST
              value: tcp://localhost:2375
            - name: AWS_WEB_IDENTITY_TOKEN_FILE
              value: "/var/run/secrets/eks.amazonaws.com/serviceaccount/token"
            - name: AWS_ROLE_ARN
              value: ${params.CBJ_ROLE}
            - name: JFROG_CLI_BUILD_NUMBER
              value: ${env.BUILD_NUMBER}
            """
          }
  }



  stages {

    stage('Provide Approval for Building new Image') {
      agent none
      steps {
        script {
          env.DEPLOY_APPROVAL = input message: 'User input required', parameters: [choice(name: 'Build  ${params.REPO_NAME}  to ${params.ENV}', choices: 'no\nyes', description: 'Choose "yes" if you want to build this  ${params.REPO_NAME} for ${params.ENV}')]
        }
      }
    }

    stage('BuildDockerImage') {
       steps {
        container("docker-daemon") {
        sh '''
        ls -altr
        docker build . -t campapp-sandbox-test -f Dockerfile
        docker tag campapp-sandbox-test artifactory.cloud.cms.gov/cre-sandbox-cloudbees-dev-docker-prod-local/cre-testing-camp
        '''
        }
       }
    }

    stage('jfrogpush') {
      steps {
        container('jfrogcli') {
          withCredentials([usernamePassword(credentialsId: 'test-jfrog', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
            sh '''
              apk add docker
              jfrog config add jfrog-arti --artifactory-url=https://artifactory.cloud.cms.gov/artifactory --user $USER --password $PASS
              jfrog config show jfrog-arti
              docker tag cre-testing-camp:latest artifactory.cloud.cms.gov/cre-sandbox-cloudbees-dev-docker-prod-local/cre-testing-camp:latest
              jfrog rt docker-push artifactory.cloud.cms.gov/cre-sandbox-cloudbees-dev-docker-prod-local/cre-testing-camp:latest docker --build-name=cre-testing-camp --build-number=${JFROG_CLI_BUILD_NUMBER}
              jfrog rt bp cre-testing-camp ${JFROG_CLI_BUILD_NUMBER}
              jfrog rt bs cre-testing-camp ${JFROG_CLI_BUILD_NUMBER}
            '''
          }
        }
      }
      }


  }
}

