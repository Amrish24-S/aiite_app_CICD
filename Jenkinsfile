def COLOR_MAP = [
    'SUCCESS': 'good', 
    'FAILURE': 'danger',
]
pipeline {
    
	agent {label 'mvn_slave'}
    tools {
        maven 'Maven-3.9.6'
        jdk 'JDK-17'
    }
    environment {
        registryCredentials = 'docker_cred'
        registry = 'amrish24/aiite-app-img'
        img_name = 'aiite-app-img'
        img_name_tags = "${registry}:${BUILD_ID}"
    }
	
    stages {
        stage('CLEAN WS'){
            steps{
                cleanWs()
            }
        }
        stage("CLONE GIT") {
            steps { 
                    // Let's clone the source
                    git branch: 'main', url: 'https://github.com/Amrish24-S/aiite_project.git'
            }
        }

        stage('UNIT TEST'){
            steps {
                sh 'mvn test'
            }
        }

	    stage('INTEGRATION TEST'){
            steps {
                sh 'mvn verify -DskipUnitTests'
            }
        }
		
        stage ('CHECKSTYLE ANALYSIS') {
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
            post{
                success{
                    echo "Generated Code Analysis Result"
                }
            }
        }
	stage('CODE ANALYSIS SONARQUBE') {
          
		  environment {
             scannerHome = tool 'Sonar'
          }

          steps {
            withSonarQubeEnv('Sonar_server') {
               sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                   -Dsonar.projectName=vprofile-repo \
                   -Dsonar.projectVersion=1.0 \
                   -Dsonar.sources=src/ \
                   -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                   -Dsonar.junit.reportsPath=target/surefire-reports/ \
                   -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                   -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
            }
            timeout (time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
            }

          }

        }
        stage('BUILD'){
            steps {
                sh 'mvn clean install -DskipTests'
            }
            post {
                success {
                    echo 'Now Archiving...'
                    archiveArtifacts artifacts: '**/target/*.war'
                }
            }
        }
        stage('TRIVY FS SCAN'){
            steps{
                sh 'trivy fs . > trivy.txt'
            }
        }
       stage('OWASP CHECK') {
            steps {
                script {
                    def dependencyCheckHome = tool 'Owasp-check'
                    sh "${dependencyCheckHome}/bin/dependency-check.sh -s ./ -f XML -o ./owasp-check.xml"
                    dependencyCheckPublisher pattern: '**/owasp-check.xml'
                }
            }
        }
        stage('IMAGE BUILD'){
            steps {
                script{
                    dockerImage = docker.build registry + ":$BUILD_ID"
                }
            }
        }
        stage('TRIVY IMG SCAN'){
            steps{
                sh "trivy image ${img_name_tags} > trivyimg.txt"
            }
        }
        stage ('PUSH IMAGE TO DOCKERHUB') {
            steps {
                script{
                    docker.withRegistry('', registryCredentials){
                        dockerImage.push("$BUILD_ID")
                        dockerImage.push("latest")
                    }
                }
            }
        }
        stage('Update Deployment File') {
            environment {
                GIT_REPO_NAME = "aiite_app_CD"
                GIT_USER_NAME = "Amrish24-S"
            }
            steps {
                withCredentials([string(credentialsId: 'github', variable: 'GITHUB_TOKEN')]) {
                    sh "git clone https://github.com/Amrish24-S/aiite_app_CD.git"
                    dir("${GIT_REPO_NAME}") {
                        sh '''
                            git config user.email "itzamrish@gmail.com"
                            git config user.name "Amrish24-S"
                            BUILD_NUMBER=${BUILD_NUMBER}
                            sed -i "s/${img_name}.*/${img_name}:${BUILD_NUMBER}/g" helm/aiite_app/templates/app_deployment.yaml
                            git add helm/aiite_app/templates/app_deployment.yaml
                            git commit -m "Update deployment image to version ${BUILD_NUMBER}"
                            git push https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME} HEAD:main
                        '''
                    }
                }
            }
        }

    }
    post {
        always {
            echo 'Slack Notifications.'
            slackSend channel: '#aiite_cicd',
                color: COLOR_MAP[currentBuild.currentResult],
                message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URL}"
        }
    }
}
