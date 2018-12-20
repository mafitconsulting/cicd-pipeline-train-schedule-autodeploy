pipeline {
    agent any
    environment {
        DOCKER_IMAGE_NAME = "fieldhousem/train-schedule"
        CANARY_REPLICAS = 0
    }
    stages {
        stage('Build') {
            steps {
                echo 'Running build automation'
                sh './gradlew build --no-daemon'
                archiveArtifacts artifacts: 'dist/trainSchedule.zip'
            }
        }
        stage('Build Docker Image') {
            when {
                branch 'feature/la_lab_branch'
            }
            steps {
                script {
                    app = docker.build(DOCKER_IMAGE_NAME)
                    app.inside {
                        sh 'echo Hello, World!'
                    }
                }
            }
        }
        stage('Push Docker Image') {
            when {
                branch 'feature/la_lab_branch'
            }
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker_hub') {
                        app.push("${env.BUILD_NUMBER}")
                        app.push("latest")
                    }
                }
            }
        }
        stage('CanaryDeploy') {
            when {
                branch 'feature/la_lab_branch'
            }
            environment { 
                CANARY_REPLICAS = 1
            }
            steps {
                kubernetesDeploy(
                    kubeconfigId: 'kubeconfig',
                    configs: 'train-schedule-kube-canary.yml',
                    enableConfigSubstitution: true
                )
            }
        }
        stage('SmokeTest') {
            when {
                branch 'la_lab_branch'
            }
            steps {
                script {
                    sleep (time: 5)
                    def response = httpRequestRes("$WORKER_1")
                    if (response.status != 200) {
                        response = httpRequestRes("$WORKER_2")
                        if (response.status != 200) {
                            error("Smoke test against canary deployment failed.")
                        }
                    }
                }
            }
        }
        stage('DeployToProduction') {
            when {
                branch 'feature/la_lab_branch'
            }
            steps {
                milestone(1)
                kubernetesDeploy(
                    kubeconfigId: 'kubeconfig',
                    configs: 'train-schedule-kube.yml',
                    enableConfigSubstitution: true
                )
            }
        }
    }
    post {       
        cleanup {
            kubernetesDeploy(
                kubeconfigId: 'kubeconfig',
                configs: 'train-schedule-kube-canary.yml',
                enableConfigSubstitution: true
            )
        }
    }

}
def httpRequestRes(worker) {
      node = worker.trim()
      response = httpRequest (
          url: "http://$node:8081/",
          timeout: 30
      )
      return response
}
