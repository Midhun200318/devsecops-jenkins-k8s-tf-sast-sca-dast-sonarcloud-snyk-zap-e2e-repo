pipeline {
    agent any
    tools { 
        maven 'Maven_3_8_4'  
    }
    environment {
        MAVEN_OPTS = "-Xmx2048m"
    }
    stages {
        stage('CompileandRunSonarAnalysis') {
            steps {	
                sh 'mvn clean verify sonar:sonar -Dsonar.projectKey=mmbuggywebapp -Dsonar.organization=mmbuggywebapp -Dsonar.host.url=https://sonarcloud.io -Dsonar.token=363744f02b4367747d67f3029802dd91b5f4c343'
            }
        }

        stage('RunSCAAnalysisUsingSnyk') {
            steps {		
                withCredentials([string(credentialsId: 'SNYK_TOKEN', variable: 'SNYK_TOKEN')]) {
                    sh 'mvn snyk:test -fn'
                }
            }
        }

        stage('Build') { 
            steps { 
                withDockerRegistry([credentialsId: "dockerlogin", url: ""]) {
                    script {
                        def app = docker.build("mid")   // ✅ fixed
                    }
                }
            }
        }

        stage('Push') {
            steps {
                script {
                    docker.withRegistry('https://787939039510.dkr.ecr.us-west-2.amazonaws.com', 'ecr:us-west-2:aws-credentials') {
                        docker.image("mid").push("latest")   // ✅ safe reference
                    }
                }
            }
        }
        
        stage('Kubernetes Deployment of MID Bugg Web Application') {
            steps {
                withKubeConfig([credentialsId: 'kubelogin']) {
                    sh 'kubectl delete all --all -n devsecops || true'
                    sh 'kubectl apply -f deployment.yaml --namespace=devsecops'
                }
            }
        }
        
        stage('wait_for_testing') {
            steps {
                sh 'pwd; sleep 180; echo "Application Has been deployed on K8S"'
            }
        }
        
        stage('RunDASTUsingZAP') {
            steps {
                withKubeConfig([credentialsId: 'kubelogin']) {
                    sh '''
                    TARGET=$(kubectl get svc midbuggy -n devsecops -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
                    
                    sudo docker run --rm -v $(pwd):/zap/wrk/:rw \
                      zaproxy/zap-stable \
                      zap-baseline.py \
                      -t http://$TARGET \
                      -r zap_report.html
                    '''
                    archiveArtifacts artifacts: 'zap_report.html'
                }
            }
        }
    }
}
