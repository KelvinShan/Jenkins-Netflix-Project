def COLOR_MAP = [
    'SUCCESS': 'good',
    'FAILURE': 'danger'
    ]

pipeline{
    agent any
    environment {
        SCANNER_HOME=tool 'sonarqube-scanner'
        TMDB_V3_API_KEY = credentials('tmdb-api-key')
        IMAGE_NAME = "netflix"
        CONTAINER_NAME = "netflix" 
        PORTS = '8082,8083,8084'

    }
    stages{
        stage('Clean Workspace')
        {
            steps{
                cleanWs()
            }
        }
        stage('GIT SCM clone')
        {
            steps{
                git branch: 'dev', url: 'https://github.com/KelvinShan/Jenkins-Netflix-Project.git'
            }
        }
        stage('Code quality')
        {
            steps{
                withSonarQubeEnv('SonarQube-Server'){
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=DevSecOps-Project \
                    -Dsonar.projectKey=DevSecOps-Project'''
                }
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
                dependencyCheck additionalArguments: "--scan ./ --disableYarnAudit --disableNodeAudit --nvdApiKey ${NVD_API_KEY}", odcInstallation: 'owasp'
            }
            dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
       }
        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"     
            }
        }
        stage('Clean Up Docker Resources') {
            steps {
                script {
                    sh '''
                    if docker ps -a --format '{{.Names}}' | grep -q $CONTAINER_NAME; then
                        echo "Stopping and removing container: $CONTAINER_NAME"
                        docker stop $CONTAINER_NAME
                        docker rm $CONTAINER_NAME
                    else
                        echo "Container $CONTAINER_NAME does not exist."
                    fi
                    '''

                    // Remove the specific image
                    sh '''
                    if docker images -q $IMAGE_NAME; then
                        echo "Removing image: $IMAGE_NAME"
                        docker rmi -f $IMAGE_NAME
                    else
                        echo "Image $IMAGE_NAME does not exist."
                    fi
                    '''
                }
            }
        }
        
        stage("Docker Build"){
            steps{
                script{
                       sh 'docker build --build-arg TMDB_V3_API_KEY=$TMDB_V3_API_KEY -t $IMAGE_NAME .'
                }
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image $IMAGE_NAME > trivyimage.txt"
            }
        }
        stage('Deploy to container'){
            steps{
                script{
                    sh 'docker run -itd --name $CONTAINER_NAME -p 8081:80 $IMAGE_NAME'
                }
            }
        }
        
    }
    post {

        always {
            echo 'slack Notification.'
            slackSend channel: '#social',
            color: COLOR_MAP [currentBuild.currentResult],
            message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URl}"
            
        }
        // failure {
        //     emailext(
        //         to: 'muhammadabdullah3602@gmail.com',
        //         subject: 'Build Status : ${BUILD_STATUS} of Build Number : ${BUILD_NUMBER}',
        //         body: 'this is the build status for this build',
        //         attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
        //     )
        // }
    }
}