/*
    This is an example pipeline that implement full CI/CD for a simple static web site packed in a Docker image.
    The pipeline is made up of 6 main steps
    1. Git clone and setup
    2. Build and local tests
    3. Publish Docker and Helm
    4. Deploy to dev and test
    5. Deploy to staging and test
    6. Optionally deploy to production and test
	usermod -aG docker jenkins
    usermod -aG root jenkins
	chmod 664 /var/run/docker.sock
But none of them work for me, I tried:

chmod 777 /var/run/docker.sock
 */

pipeline {
    // In this example, all is built and run from the master
	agent any

    environment {
        GITHUB_URL = "https://github.com/GMKBabu/eks-cicd.git"
        GITHUB_CREDENTIALS_ID = "0b61464e-dd11-4760-b30a-f988490eb429"
        CUSTOM_TAG = "${BUILD_NUMBER}"
        AWS_DEFAULT_REGION = 'us-east-1'
        AWS_ACCOUNT_ID = "504263020452"
        IMAGE_REPO_NAME = "eks"
        TEST_LOCAL_PORT = "80"
		CUSTOM_BUILD_NUMBER = "DEV-PRD-${BUILD_NUMBER}"
		ID = "${IMAGE_REPO_NAME}"
		IMAGE_NAME = "${IMAGE_REPO_NAME}:${CUSTOM_TAG}"
        TOPIC_ARN = "arn:aws:sns:us-east-1:504263020452:config-topic"
    }
    parameters {
        string (name: 'GITHUB_BRANCH_NAME', defaultValue: 'master', description: 'Git branch to build')
        booleanParam (name: 'DEPLOY_TO_PROD', defaultValue: false, description: 'If build and tests are good, proceed and deploy to production without manual approval')
    }
    triggers {
    //Run Polling of GitHub every minute everyday of the week
        pollSCM ('* * * * *')
        //cron ('0 0 * * 1-5')}
    // Pipeline stages
    }
    options {
	    buildDiscarder(logRotator(numToKeepStr: '5', artifactDaysToKeepStr: '3', artifactNumToKeepStr: '1'))
        timeout(time: 60, unit: 'MINUTES')
    }
    stages {
             stage("GITHUB_code_checkout") {
        steps {
		   // using for checkout the code from github
		 checkout([$class: 'GitSCM', branches: [[name: "*/${GITHUB_BRANCH_NAME}"]],doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], \
	       userRemoteConfigs: [[credentialsId: "${GITHUB_CREDENTIALS_ID}", url: "${GITHUB_URL}"]]])
	   script {
	       currentBuild.displayName = "${CUSTOM_BUILD_NUMBER}"
	     }
        }
        }
        ////////// Step 2 //////////
        stage('Build Docker Image and Test') {
            parallel {
                stage ('DOcker Image Build') {
                    steps {
                        echo "Building application and Docker image"
                        sh "docker build -t $IMAGE_NAME  ${WORKSPACE}/."
                        }
                    }

                stage('Docker Container Test') {
                    steps {
                        echo "Running Test"

                        // Kill container in case there is a leftover
                        sh "[ -z \"\$(docker ps -a | grep ${ID} 2>/dev/null)\" ] || docker rm -f ${ID}"

                        echo "Starting ${IMAGE_REPO_NAME} container"
                        sh "docker run --detach --name ${ID} --rm --publish ${TEST_LOCAL_PORT}:80 ${IMAGE_NAME}"
                    }
                }
            }
        }

        // Run the 3 tests on the currently running ACME Docker container
        stage('Local tests') {
            steps {
			    sh '''
				    host_ip=$(hostname -i)
				    curl -aG http://$host_ip:80
				'''
                        
                }
        }
        ////////// Step 3 //////////
		
        stage("Publish Docker Image") {
            parallel {
                stage('stop docker container') {
                    steps {
                        echo "Stop and remove container"
                        sh 'docker stop "${ID}"'
                    }
                }
                stage('Login ECR Repository') {
                    steps {
                        echo "login to ecr repository"
                        sh '(eval \$(aws ecr get-login  --no-include-email --region "${AWS_DEFAULT_REGION}"))'
                    }
                }
                stage('Pushing teh docker images to ECR Repository') {
                    steps {
                        echo 'Pushing "${IMAGE_NAME}" image to registry'
                        echo "change the docker image tag name"
                        sh 'docker tag "${IMAGE_NAME}" "${AWS_ACCOUNT_ID}".dkr.ecr."${AWS_DEFAULT_REGION}".amazonaws.com/"${IMAGE_NAME}"'
                        echo "Pushing the Docker image...  "
				        sh 'docker push "${AWS_ACCOUNT_ID}".dkr.ecr."${AWS_DEFAULT_REGION}".amazonaws.com/"${IMAGE_NAME}"'
                    }
                }
            }
        }

        // Waif for user manual approval, or proceed automatically if DEPLOY_TO_PROD is true
        stage('Go for Production?') {
            when {
                allOf {
                    environment name: 'GITHUB_BRANCH_NAME', value: 'master'
                    environment name: 'DEPLOY_TO_PROD', value: 'false'
                }
            }
            steps {
                // Prevent any older builds from deploying to production
               milestone(1)
                input("Proceed and deploy to Production?")
               milestone(2)
                script {
                    DEPLOY_PROD = true
                }
            }
        }

        stage('Deploy to Production') {
            when {
                anyOf {
                    expression { DEPLOY_PROD == true }
                    environment name: 'DEPLOY_TO_PROD', value: 'true'
                }
            }
            steps {
                script {
                    DEPLOY_PROD = true
					
					echo "generate imagePullSecrets"
					sh "chmod 777 ${WORKSPACE}/ecr-login.sh"
					sh "${WORKSPACE}/ecr-login.sh"

                    // Deploy with helm
                    echo "Deploying"
					sh """
					    
                        /usr/local/bin/helm upgrade --install cicd --set image.repository="${AWS_ACCOUNT_ID}".dkr.ecr."${AWS_DEFAULT_REGION}".amazonaws.com/"${IMAGE_NAME}" "${WORKSPACE}"/cicd
                    """
					sh "sleep 5"
                }
			}
			
        }

        stage("Production tests") {
            when {
                expression { DEPLOY_PROD == true }
            }
            parallel {
                stage('Test Helm list') {
                    steps {
                        echo "check helm list"
                        sh """
                        /usr/local/bin/helm list

                        """
                        sh "sleep 5"
                    }
                }
                stage("Check Ingress Url") {
                    steps {
			            sh '''
				            host_url=$(/root/bin/kubectl get ingress -n babu |grep ingress | awk '{print $3}')
				            curl -aG http://$host_url:80
         				'''
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Successful build occured"
            script {

                currentBuild.result = "SUCCESS"
            }
            NotifyEmail()
        }
        failure {
            echo "failure build occured"
            script {
                currentBuild.result = "FAILURE"
            }
            NotifyEmail()
        }

    }
}
def NotifyEmail() {
    sh '''
    DEPLOYMENT=$(/root/bin/kubectl get deployment -n babu -o wide | grep cicd | awk '{ print $2,$4,$5}')
    IMAGE=$(/root/bin/kubectl get deployment -n babu -o wide | grep cicd | awk '{ print $7}')
    INGRESS_URL=$(/root/bin/kubectl get ingress -n babu |grep ingress | awk '{print $3}')
    NODES=$(/root/bin/kubectl get nodes -n babu -o wide | grep -i Ready | wc -l)
    PODS=$(/root/bin/kubectl get pod -n babu -o wide |grep -E 'cicd|Running' | wc -l)
    PODS_IPS=$(/root/bin/kubectl get pod -n babu -o wide |grep -E 'cicd|Running' | awk '{ print $1,$6}')

    aws sns publish --topic-arn \"${TOPIC_ARN}\" \
    --region \"${AWS_DEFAULT_REGION}"\
    --subject \"Status: Job_Name: ${JOB_NAME}\" \
    --message "Job_Name: ${JOB_NAME}\n Deployment_Status: $DEPLOYMENT\n Deployment_Image: $IMAGE\n Ingress_URL: $INGRESS_URL\n  Nodes: $NODES\n No_PODS: $PODS\n Pods_ips: $PODS_IPS\n Build_Number: ${BUILD_NUMBER}\n Build_URL: ${BUILD_URL}"
    '''
}


/*
replace gmkbabu to <img src="'${WORKSPACE}'/GMKBabu.png" />
def NotifyEmail() {
        emailext (
            to: "babu.g0730@gmail.com",
            subject: "Status: ${currentBuild.result}",
            attachLog: true,
            body: """<style>
                       body, table, td, th, p {
                           font-family:verdana,helvetica,sans serif;
                           font-size:11px;
                           color:black;
                           }
                        td.bg1 { color:white; background-color:#595959; font-size:120% }
                        td.console { font-family:courier new, lucida console; }
                    </style>
                  <body>
                    <table border=2 cellspacing=2 cellpadding=2 width="40%">
                          <tr>
                              <td align="left" width="40%">
                                <b style="font-size: 170%;">GMKBabu</b>
                              </td>
                              <td valign="center" width="60%">
                                 <b style="font-size: 150%;">CICD Pipeline Information</b>
                              </td>
                            </tr>
                            <tr>
                                <td>URL:</td>
                                <td><a href='${BUILD_URL}'>${JOB_NAME}</a></td>
                            </tr>
                            <tr>
                                <td>BuildNumber:</td>
                                <td>${CUSTOM_BUILD_NUMBER}</td>
                            </tr>
                            <tr>
                                <td>JobName:</td>
                                <td>${JOB_NAME}</td>
                            </tr>
                    </table>
                    <br />
                  </body>"""
                )
}*/

