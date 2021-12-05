pipeline{
    agent any
    options {
        buildDiscarder(logRotator(numToKeepStr: "3"))
    }
    environment{
        VERSION="${env.BUILD_ID}"
        IMAGE="eshop/webhooks.client"
        ACR="abouhamed.azurecr.io"
        SOLUTION = "./src/eShopOnContainers-ServicesAndWebApps.sln"
        PROJECT = "./src/Web/WebhookClient/WebhookClient.csproj"
        CHART_DIR = "./deploy/webhooks-web"
        SONAR_PRO_KEY = "webhooks-client"
    }
    stages{
        stage("Sonar Test"){
            agent{
                docker {
                    image 'nosinovacao/dotnet-sonar'
                    args '-u root' 
                }
            }  
            steps{
                withSonarQubeEnv('sonar') {
                    sh '''
                        dotnet /sonar-scanner/SonarScanner.MSBuild.dll begin /k:${SONAR_PRO_KEY}
                        dotnet restore ${SOLUTION}
                        dotnet publish ${PROJECT} --no-restore -c Release -o /app
                        dotnet /sonar-scanner/SonarScanner.MSBuild.dll end
                        '''
                }
            }
        }
        stage("Sonar Quality Gate") {
            steps {
              timeout(time: 3, unit: 'MINUTES') {
                waitForQualityGate abortPipeline: true
              }
            }
        }
        stage("Docker Build"){
            steps{
                dir('./src') {
                    sh 'docker build -t $ACR/$IMAGE:$VERSION .'
                }
            }
        }
        stage("Docker Push"){
            steps{
                withCredentials([usernamePassword(credentialsId: 'acr', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                    sh 'echo $PASS | docker login $ACR -u $USER --password-stdin'
                    sh 'docker push $ACR/$IMAGE:$VERSION'
                }
            }
        }
        stage("datree: Validate Helm Config"){
            steps{
                withCredentials([string(credentialsId: 'datree_token', variable: 'DATREE_TOKEN')]) {
                    sh 'datree config set token ${DATREE_TOKEN}'
                    sh 'helm datree test ${CHART_DIR}'
                }
            }
        }
        stage("Helm Lint & Package"){
            steps{
                    sh '''
                        helm lint $CHART_DIR
                        helm package \
                            --app-version $VERSION \
                            $CHART_DIR
                        '''    
            }
        }
        stage("Helm Push"){
            steps{
                withCredentials([usernamePassword(credentialsId: 'acr', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                    sh '''
                        export HELM_EXPERIMENTAL_OCI=1
                        echo $PASS | helm registry login $ACR --username $USER --password-stdin
                        chartVersion=$( helm show chart $CHART_DIR | grep version | cut -d: -f 2 | tr -d ' ')
                        helm push ./*.tgz oci://$ACR/helm
                        '''
                }    
            }
        }
    }
    post {
		always {
			mail bcc: '', body: "<br>Project: ${env.JOB_NAME} <br>Build Number: ${env.BUILD_NUMBER} <br> URL de build: ${env.BUILD_URL}", cc: '', charset: 'UTF-8', from: '', mimeType: 'text/html', replyTo: '', subject: "${currentBuild.result} CI: Project name -> ${env.JOB_NAME}", to: "devopsahmed89@gmail.com";  
		}
	}
}