pipeline{
    agent any
    environment{
	    ENVIRONMENT = 'TEST'
	    TIME_STAMP = new java.text.SimpleDateFormat('yyyy_MM_dd').format(new Date())
        DOCKER_TAG = "${ENVIRONMENT}_${TIME_STAMP}_${env.BUILD_NUMBER}"
    }
    stages{
        stage("sonar quality check"){
            agent{
                docker {
                    image 'openjdk:11'
                }
            }
            steps{
                script{
                    withSonarQubeEnv(credentialsId: 'sonarqube') {
                            sh 'chmod +x gradlew'
                            sh './gradlew sonarqube'
                    }

                    timeout(5) {
                      def qg = waitForQualityGate()
                      if (qg.status != 'OK') {
                           error "Pipeline aborted due to quality gate failure: ${qg.status}"
                      }
                    }

                }
            }
        }
        stage("docker build & docker push to nexus"){
            steps{
                script{
                    withCredentials([string(credentialsId: 'nexus_password', variable: 'nexus_password')]) {
                             sh '''
                                docker build -t 54.83.102.185:8083/springapp:${DOCKER_TAG} .
                                docker login -u admin -p $nexus_password 54.83.102.185:8083
                                docker push 54.83.102.185:8083/springapp:${DOCKER_TAG}
				docker logout
			    '''
		    }
		}
	    }
	}
	stage("docker image push to dockerhub and remove the images locally"){
            steps{
                script{    
	                     sh '''
			        cat my_password.txt | docker login --username krishneswari --password-stdin
				docker tag 54.83.102.185:8083/springapp:${DOCKER_TAG} krishneswari/springapp:${DOCKER_TAG}
                                docker push krishneswari/springapp:${DOCKER_TAG}
				docker rmi 54.83.102.185:8083/springapp:${DOCKER_TAG}
				docker rmi krishneswari/springapp:${DOCKER_TAG}
				docker logout
                            '''
		}  
            }
        }
        stage("Identifying misconfigurations using datree in helm charts"){
            steps{
                script{

                    dir('kubernetes/') {
                        withEnv(['DATREE_TOKEN=8JuAwRqHpUgrUDEomZP9Tp']) {
                              sh 'helm datree test myapp/'
                        }    
                    }
                }
            }
        }
        stage("Pushing the helm charts to nexus"){
            steps{
                script{
                    withCredentials([string(credentialsId: 'nexus_password', variable: 'nexus_password')]) {
                          dir('kubernetes/') {
                             sh '''
                                  helmversion=$( helm show chart myapp | grep version | cut -d: -f 2 | tr -d ' ')
                                  tar -czvf myapp-${helmversion}.tgz myapp/
                                  curl -u admin:$nexus_password http://54.83.102.185:8081/repository/helm-repo/ --upload-file myapp-${helmversion}.tgz -v
                            '''
                          }
                    }
                }
            }
        }
        stage("Copying the playbook and kubernetes manifests to Ansible server"){
		    steps{
			    script{
				    echo "${WORKSPACE}"
					echo "${DOCKER_TAG}"
					        sh '''
								 sudo grep -irl {DOCKER_TAG} ${WORKSPACE}/kubernetes/manifests-yamls/deployment.yaml | xargs sed -i "s/{DOCKER_TAG}/${DOCKER_TAG}/g"
								 sudo scp -o StrictHostKeyChecking=no -i /root/.ssh/id_rsa ${WORKSPACE}/kubernetes/manifests-yamls/*.yaml root@172.31.88.87:/etc/ansible/k8s/
							'''     
				}
            }
        }			
							
        stage('manual approval'){
            steps{
                script{
                    timeout(10) {
                        mail bcc: '', body: "<br>Project: ${env.JOB_NAME} <br>Build Number: ${env.BUILD_NUMBER} <br> Go to build url and approve the deployment request <br> URL of the build: ${env.BUILD_URL}", cc: '', charset: 'UTF-8', from: '', mimeType: 'text/html', replyTo: '', subject: "${currentBuild.result} CI: Project name -> ${env.JOB_NAME}", to: "vakr1570@gmail.com";  
                        input(id: "Deploy Gate", message: "Deploy ${params.project_name}?", ok: 'Deploy')
                    }
                }
            }
        }

        stage('Deploying application on k8s cluster from ansible-server using playbook') {
            steps {
               script{
			        sh 'ssh -o StrictHostKeyChecking=no -i /root/.ssh/id_rsa root@172.31.88.87 "/root/bin/kubectl config get-contexts; /root/bin/kubectl config use-context kubernetes-admin@kubernetes; whoami && hostname; ansible-playbook /etc/ansible/k8s/playbook-deployment-service.yaml; sudo rm -rf /etc/ansible/k8s/*.yaml;"'   
               }
            }
        }
        
        stage('verifying app deployment'){
            steps{
                script{
                     withCredentials([kubeconfigFile(credentialsId: 'k8s', variable: 'KUBECONFIG')]) {
                         sh 'kubectl run curl --image=curlimages/curl -i --rm --restart=Never -- curl sample-tomcat-service:8080'

                     }
                }
            }
        }
    }
	   
    post {
		always {
			mail bcc: '', body: "<br>Project: ${env.JOB_NAME} <br>Build Number: ${env.BUILD_NUMBER} <br> URL of the build: ${env.BUILD_URL}", cc: '', charset: 'UTF-8', from: '', mimeType: 'text/html', replyTo: '', subject: "${currentBuild.result} CI: Project name -> ${env.JOB_NAME}", to: "vakr1570@gmail.com";  
		 }
	   }
}
