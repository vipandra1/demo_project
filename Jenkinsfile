pipeline {
    agent { label 'master' }
    parameters {
        choice(name: 'BUILD_FOR_DEPLOYMENT', choices: ["yes", "no"], description: 'When no, Skip deployment stage')
	choice(name: 'BUILD_FOR_HPA', choices: ["no", "yes"], description: 'When no, Skip hpa stages')
	choice(name: 'BUILD_FOR_ROLLBACK', choices: ["no", "yes"], description: 'When no, Skip rollback stages')
    }
    stages {
        stage('Take Choice From user') {
            steps {
                // getting parameter if build trigger mannual 
                echo "Build for deployment ${BUILD_FOR_DEPLOYMENT}"
		echo "Build for hpa ${BUILD_FOR_HPA}"
		echo "Build for rollback ${BUILD_FOR_ROLLBACK}"
            }
            post {
                failure {
                slackSend channel: 'loco_testing', message: "*****Pipeline failed on user inpi*****"
                }
            }
        }
        stage('Building Image') {
            when { expression { params.BUILD_FOR_DEPLOYMENT == "yes" } }
            steps {
                sh('''#!/bin/bash
                echo "echo building image with the container with ${BRANCH}" 
		        ''')
            }
            post {
                failure {
                slackSend channel: 'loco_testing', message: "*****image is not build*****"
                }
            }
        }
	stage('pushing image to ECR') {
            when { expression { params.BUILD_FOR_DEPLOYMENT == "yes" } }
            steps {
                sh('''#!/bin/bash
                echo "pushing image to ECR repo ${BRANCH}" 
		        ''')
            }
            post {
                failure {
                slackSend channel: 'loco_testing', message: "*****image not pushed.*****"
                }
            }
        }
	stage('deploying pods on cluster') {
            when { expression { params.BUILD_FOR_DEPLOYMENT == "yes" } }
            steps {
                sh('''#!/bin/bash
                echo "deploying pods on cluster" 
		        ''')
            }
            post {
                failure {
                slackSend channel: 'loco_testing', message: "*****pdos not deployed.*****"
                }
            }
        }
	stage('changing value of hpa') {
            when { expression { params.BUILD_FOR_HPA == "yes" } }
            steps {
		script {
                    def valueHpa = input(
                        message: 'Please provide a number:',
                        ok: 'Continue',
                        parameters: [string(name: 'HPA_VALUE', defaultValue: '', description: 'Enter your number')]
                    )
		    HPA_VALUE = valueHpa
		    echo "changing HPA value ${HPA_VALUE}"
                } 
            }
            post {
		success { 
			script {
                slackSend channel: 'loco_testing', message: "*****HPA value Change to ${HPA_VALUE}.*****"
			}
                } 
                failure {
                slackSend channel: 'loco_testing', message: "*****changing hpa value failure.*****"
                }
            }
        }
	stage('Roll Back stage with image version') {
            when { expression { params.BUILD_FOR_ROLLBACK == "yes" } }
            steps {
		 script {
		    def ecrImages = sh(script: "aws ecr describe-images --repository-name locodemoapp --region ap-south-1 --output json --query 'sort_by(imageDetails,& imagePushedAt)[-5:] | reverse(@)' --no-paginate", returnStdout: true).trim()
                  //  def ecrImages = sh(script: 'aws ecr list-images --repository-name locodemoapp --region ap-south-1 --query "imageIds[0:5]" --output json', returnStdout: true).trim()
                    def imageIds = readJSON text: ecrImages
                    echo "${imageIds}"
                    def options = imageIds.collect { it.imageTags }

                    echo "Image Tags: ${options}"
                    def userInput = input(
                        id: 'imageChoice',
                        ok: 'Continue',
                        message: 'Select an image to deploy:',
                        parameters: [choice(name: 'IMAGE_TAG', choices: options, description: 'choose any image for rollback')]
                    )
			 IMAGE_TAG = userInput
                    echo "${IMAGE_TAG}"
                 }
            }
            post {
		success {
			script {
                slackSend channel: 'loco_testing', message: "*****ROLLBACK sucessful with image version ${IMAGE_TAG}.*****"
			}
                }   
                failure {
                slackSend channel: 'loco_testing', message: "*****Roll back to particualr version failure.*****"
                }
            }
        }
    }
    post {
        always {
            notifySlack(currentBuild.currentResult)
        }
    }
}

def notifySlack(String buildStatus = 'STARTED') {
    def color
    if (buildStatus == 'STARTED') {
        color = '#D4DADF'
    } else if (buildStatus == 'SUCCESS' ) {
        color = 'good'
    } else if (buildStatus == 'UNSTABLE') {
        color = 'warning'
    } else {
        color = 'danger'
    }
    def msg = "`${env.JOB_NAME}` build number #${env.BUILD_NUMBER}:`${buildStatus}`\n${env.BUILD_URL}   "
    slackSend(channel: '#loco_testing', color: color, message: msg)
}
