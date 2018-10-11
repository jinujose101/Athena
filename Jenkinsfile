pipeline {
	agent { label 'master' }
	environment {        	
		myVersion = '0.9'
	        dotnet = 'path\to\dotnet.exe'
    	}
        tools {
       		 msbuild '.NET Core 2.0.0'
        }
	stages {
		stage('Checkout') {
			steps {
				checkout scm
			}
		}
		stage ('UnitTests')
		{
			bat returnStatus: true, script: "\"C:/Program Files/dotnet/dotnet.exe\" test \"${workspace}/XUnitTestProject1.sln\" --logger \"trx;LogFileName=unit_tests.xml\" --no-build"
			step([$class: 'MSTestPublisher', testResultsFile:"**/unit_tests.xml", failOnError: true, keepLongStdio: true])
			
		}
		stage('Build image') {
			steps {
				sh 'docker build -t {IMAGE_NAME} .'
			}
		} 
		stage('Push image') {
			steps {
				script {
					docker.withRegistry('https://{ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com', 'ecr:us-east-1:aws-credentials') {
						docker.image('{IMAGE_NAME}').push("v${env.BUILD_NUMBER}")
						docker.image('{IMAGE_NAME}').push('latest')
					}
				}
			}
		}
		stage('Run instance')
		{
			steps {
				sh 'sed -e "s;%BUILD_NUMBER%;${BUILD_NUMBER};g" {TASK_NAME}.json >  {TASK_NAME}-v${BUILD_NUMBER}.json'
				sh 'aws ecs register-task-definition --family {TASK_NAME} --cli-input-json file://{TASK_NAME}-v${BUILD_NUMBER}.json --region us-east-1'
				sh '''	
					TASK_REVISION=`aws ecs describe-task-definition --task-definition {TASK_NAME} --region us-east-1 | jq .taskDefinition.revision`
					SERVICES=`aws ecs describe-services --services {SERVICE_NAME} --cluster {CLUSTER_NAME} --region us-east-1 | jq '.services[] | length'`
					if [ $SERVICES == "" ]; then 
						aws ecs create-service --cluster {CLUSTER_NAME} --region us-east-1 --service {SERVICE_NAME} --task-definition {TASK_NAME}:${TASK_REVISION} --desired-count 1 
					else 
						aws ecs update-service --cluster {CLUSTER_NAME} --service {SERVICE_NAME} --task-definition {TASK_NAME}:${TASK_REVISION} --desired-count 1 --region us-east-1
					fi
				'''
			}
		}
		stage('Clean up')
		{		
			steps {
				sh 'docker image prune -a -f'
				deleteDir()
				script {
					currentBuild.result = 'SUCCESS'
				}
			}
		}
	}
	post {
        failure {
            script {
                currentBuild.result = 'FAILURE'
            }
        }

        always {
            step([$class: 'Mailer',
                notifyEveryUnstableBuild: true,
                recipients: "{RECIPIENTS}",
                sendToIndividuals: true])
        }
    }
}
