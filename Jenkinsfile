
pipeline {
    environment { 
	
		USERNAME='manmeetdevgun'

		/////////GIT-CONFIGS///////////
		GITURL = 'https://github.com/superuserz/app_manmeetdevgun.git'
		
		
		////////SONAR-CONFIGS/////////
		SONAR_INSTALLATION = 'Test_Sonar'
        SONAR_PROJECTKEY = 'sonar-manmeetdevgun'
        SONAR_HOST = 'http://localhost:9000'
        SONAR_TOKEN = 'c1084e0ae20a2a86ee4ba7001da3d9d8575411e7'
		SONAR_COVERAGEPATH = 'target/site/jacoco/jacoco.xml'
		
		////////DOCKER-CONFIGS////////
        DOCKER_CONTAINER_MASTER_PORT = '7200'
        DOCKER_CONTAINER_DEVELOP_PORT = '7300'
		DOCKER_REPO = 'superuserz'
		

        
        //////////KUBERNETES-CONFIGS/////////
		KUBERNETES_DEPLOYMENTFILE = 'deployment.yml'
		KUBERNETES_MASTERPORT = '30157'
		KUBERNETES_DEVELOPPORT = '30158'
		KUBERNETES_NAMESPACE = 'kubernetes-cluster-manmeetdevgun'
		GCE_PROJECTID = 'calcium-rigging-322119'
		GCE_CLUSTER = 'nagp-k8s-jenkins-cluster'
		GCE_CLUSTERLOCATION = 'asia-south1-a'
		GCE_JENKINS_SA_KEY = 'gke'
		GCE_ACCOUNT = 'manmeet.devgun3152707@gmail.com'

    }
    agent any
    
    tools { 
        maven 'maven3' 
    }
    
    options {

        timeout(time: 1, unit: 'HOURS')
        skipStagesAfterUnstable() 
    }
    
    //Decalare Stages below this
    stages {
        
        stage('Checkout') {
            steps {
                
                script {
                    
                    git poll: true, url: GITURL, branch: env.BRANCH_NAME
                }
            }
        } //Checkout Stage End.  
        
        stage('Build') {
            steps {
                echo "M2_HOME = ${M2_HOME}"
                echo "${env.BRANCH_NAME}"
                bat 'mvn -B -DskipTests clean package'  //Build the Project
                bat 'mvn verify'        //Generate jcocco reports
                bat 'dir target'
            }
        } //Build Stage End.
        
        stage('Unit Testing') {
            when {
                expression { env.BRANCH_NAME == 'master' }
            }
            steps {
                bat 'mvn test'
            }
        } //Unit Testing Stage End.
        
        stage('Sonar Analysis') {
            when {
                expression { env.BRANCH_NAME == 'develop' }
            }
            
            steps  {
                            withSonarQubeEnv(SONAR_INSTALLATION) {       //Run Sonar Qube Analysis. For Quality gate, you need to setup up a webhook in sonar. Not in scope of this assignment.
                            bat "mvn package sonar:sonar \
  				                -Dsonar.host.url=${SONAR_HOST} \
				                -Dsonar.projectKey=${SONAR_PROJECTKEY} \
				                -Dsonar.login=${SONAR_TOKEN} \
                                -Dsonar.coverage.jacoco.xmlReportPaths=${SONAR_COVERAGEPATH} \
                                -Dsonar.java.binaries=src/main/java"
                            }
            }
        } //Sonar Analysis Stage End.
        
        stage('Build Docker Image') {
                steps {
                    script {
                        bat "docker build -t ${DOCKER_REPO}/i-${USERNAME}-${env.BRANCH_NAME}:v1 --build-arg JAR_FILE=target/nagp-devops-0.0.1-SNAPSHOT.jar ."
                    }
                }
        } //Build Docker Stage End
        
        stage('Containers') {
            
            parallel {
                stage('Docker Push') {
                    agent any
                    steps  {
                        script  { 
                                    withCredentials([usernamePassword(credentialsId: 'DockerHub', passwordVariable: 'dockerpassword', usernameVariable: 'dockerusername')]) {
                                        bat "docker login -u ${dockerusername} -p ${dockerpassword}"
                                        
                                        bat "docker push ${DOCKER_REPO}/i-${USERNAME}-${env.BRANCH_NAME}:v1"
                                    }
                                }
                           } 
                }
                
                stage('Pre-container Check') {
                    agent any
                    steps {
                        script {
                            try {
                                    bat "docker stop c-${USERNAME}-${env.BRANCH_NAME}"   // stop already running container
                                    
                                    bat "docker container rm c-${USERNAME}-${env.BRANCH_NAME}"  // remove the old container
                                    
			                        sleep 5 //seconds //give some time for container to stop.
                                } catch (Exception err) {
                                    //If no container with same name is running, this step will throw an exceptoin. Handle it and do Nothing.
                                }
                       }
                   }
               }
           }
        }
        stage('Docker Deploy') {
            steps {
                script {
                        
                        if (env.BRANCH_NAME == 'master') {
                            bat "docker run --pull=allways -itd -p ${DOCKER_CONTAINER_MASTER_PORT}:8080 --name c-${USERNAME}-${env.BRANCH_NAME} ${DOCKER_REPO}/i-${USERNAME}-${env.BRANCH_NAME}:v1"
                            //Run the New image on Docker instance
                        }   
                        if (env.BRANCH_NAME == 'develop') {
                            bat "docker run --pull=allways -itd -p ${DOCKER_CONTAINER_DEVELOP_PORT}:8080 --name c-${USERNAME}-${env.BRANCH_NAME} ${DOCKER_REPO}/i-${USERNAME}-${env.BRANCH_NAME}:v1"
                            //Run the New image on Docker instance
                            
                        }
                } 
            }
        }//Docker Deploy Stage End
        
        stage('Kubernetes Deployment'){
            steps{
                script{
                    
                    withCredentials([file(credentialsId: 'gcesakey', variable: 'gcesakey')]) {      //gcesakey refers to the service account key for GCE service account.
                                                                                                    //Ensure NOT to commit the SA key to any public repo.
                        
                    bat "gcloud config set account ${GCE_ACCOUNT}"               //Set gcloud to correct google-account.
                    
                    bat "gcloud auth activate-service-account --key-file $gcesakey"         //Set gcloud to correct google-account's service account.

                    bat "gcloud container clusters get-credentials ${GCE_CLUSTER} --zone ${GCE_CLUSTERLOCATION} --project ${GCE_PROJECTID}"  //connect to the gcloud kubernetes cluster.
                    
                    bat "kubectl apply -f ${KUBERNETES_DEPLOYMENTFILE}"  // Apply the deployment.
					
					bat "kubectl config set-context --current --namespace=${KUBERNETES_NAMESPACE}" //set name-space
                    
                    bat "kubectl set image deployment i-${USERNAME}-${env.BRANCH_NAME} i-${USERNAME}-${env.BRANCH_NAME}=${DOCKER_REPO}/i-${USERNAME}-${env.BRANCH_NAME}:v1"  //set the deployment with build image.
					
						try {
							if(env.BRANCH_NAME == 'master'){
								bat "gcloud compute firewall-rules create master-node-port --allow tcp:${KUBERNETES_MASTERPORT} --project ${GCE_PROJECTID}"   //Set appropraie firewall Rile to connect to VM
							}
                        
							if(env.BRANCH_NAME == 'develop'){
                            bat "gcloud compute firewall-rules create develop-node-port --allow tcp:${KUBERNETES_DEVELOPPORT} --project ${GCE_PROJECTID}"  //Set appropraie firewall Rile to connect to VM
							}  
							} catch (Exception e) {}
						}  
                }
            }
        } //Kubernetes Stage end
    } //stages
}//pipline
