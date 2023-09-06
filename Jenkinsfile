pipeline {
    agent any
    tools{
        maven 'MAVEN3'
    }
    environment {
        registry = '729590520513.dkr.ecr.us-west-2.amazonaws.com/hellodatarepo'
        registryCredential = 'awscreds'
        dockerimage = ''
    }
    stages{


        stage("Checkout the project") {
           steps{
              git branch: 'docker', url: 'https://github.com/devopshydclub/vprofile-project.git'
           }
        }


        stage("Build the package"){
            steps {
                sh 'mvn clean package'
            }
        }


        stage('Test'){
            steps {
            sh 'mvn test'
            }
        }


        stage ('CODE ANALYSIS WITH CHECKSTYLE'){
                steps {
                    sh 'mvn checkstyle:checkstyle'
                }
                post {
                    success {
                        echo 'Generated Analysis Result'
                    }
                }
        }


		stage('JUnit Tests') {
          steps {
            script {
              def mvnHome = tool name: 'MAVEN3', type: 'hudson.tasks.Maven$MavenInstallation'
              def junitReportPath = 'target/surefire-reports/**/*.xml'
            
            // Run JUnit tests
            sh "${mvnHome}/bin/mvn test"
            
            // Archive JUnit test reports
            junit allowEmptyResults: true, testResults: junitReportPath
            }
          }
        }




         stage('build && SonarQube analysis') {
           environment {
            scannerHome = tool 'sonar4.7'
         }
           steps {
               withSonarQubeEnv('sonar') {
                sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                  -Dsonar.projectName=vprofile-repo \
                  -Dsonar.projectVersion=1.0 \
                  -Dsonar.sources=src/ \
                  -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                  -Dsonar.junit.reportsPath=target/surefire-reports/ \
                  -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                  -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
               }
           }
       }




        stage("Sonar Quality Check"){
        steps{
            script{
             withSonarQubeEnv(installationName: 'sonar', credentialsId: 'jenkins-token') {
             sh 'mvn sonar:sonar'
            }
             timeout(time: 1, unit: 'HOURS') {
              def qg = waitForQualityGate()
              if (qg.status != 'OK') {
                  error "Pipeline aborted due to quality gate failure: ${qg.status}"
         }
             }
            }
      }
    }  


        stage('Generate Jacoco Reports') {
            steps {
                // This step generates Jacoco reports
                sh 'mvn jacoco:report'
            }
            post {
                always {
                    // Archive the generated Jacoco reports for reference
                    archiveArtifacts allowEmptyArchive: true, artifacts: 'target/site/jacoco/**'
                }
            }
    }




    stage('Building the Image') {
        steps {
            script {
            dockerImage = docker.build( registry + ":$BUILD_NUMBER", "./Docker-files/app/multistage/")
        }
    }
    }


        // Image scanning using Anchore Grype
        stage('Anchore Grype Scan Image') {
            steps {
                script {
					def imageDigest = sh(script: "aws ecr describe-images --repository-name hellodatarepo --output json --region us-west-2 | jq -r '.imageDetails | sort_by(.imagePushedAt) | reverse | .[0].imageDigest'", returnStdout: true).trim()
					def imageTags = sh(script: "aws ecr describe-images --repository-name hellodatarepo --output json --region us-west-2 | jq -r '.imageDetails | sort_by(.imagePushedAt) | reverse | .[0].imageTags[]'", returnStdout: true).trim()
					
					def imageName = "${registry}:${imageTags}"
					sh "grype ${imageName}" // Run Grype scan
                }
            }
        }


// Upload image to amazon ecr
    stage ('Deploy the Image to Amazon ECR') {
       steps {
           script {
                docker.withRegistry("http://" + registry, "ecr:us-west-2:" + registryCredential ) {
                dockerImage.push("$BUILD_NUMBER")
                dockerImage.push('latest')
            }
         }
      }
    }


stage('Download and Configure Kubeconfig') {
            steps {
                script {
                   // Retrieve AWS credentials from Jenkins credentials with ID 'awscreds'
                    def awsCredentials = credentials('awscreds')
                    def awsAccessKeyId = awsCredentials

                    // Define the AWS region and cluster name
                    def awsRegion = 'us-west-2'
                    def clusterName = 'fleetman'

                    // Define the KUBECONFIG file path
                    def kubeconfigPath = "${env.WORKSPACE}/kubeconfig.yaml"

                    // Set the AWS credentials for this session
                    sh """
                        aws configure set aws_access_key_id ${awsAccessKeyId}
                        aws configure set region ${awsRegion}
                    """

                    // Download the KUBECONFIG file from EKS
                    sh "aws eks --region ${awsRegion} update-kubeconfig --name ${clusterName} --kubeconfig ${kubeconfigPath}"

                    // Set the KUBECONFIG environment variable
                    env.KUBECONFIG = kubeconfigPath

                    // Verify that the KUBECONFIG variable is set correctly
                    echo "KUBECONFIG set to: ${env.KUBECONFIG}"

                    // List available contexts in the KUBECONFIG file
                    sh "kubectl config get-contexts"

                    // Automatically set the current context to the first available context
                    sh "kubectl config use-context \$(kubectl config get-contexts -o name | head -n 1)"
                }
            }
        }

        stage('Deploy to Amazon EKS') {
            steps {
                script {
                    // Use the 'workloads.yaml' file for deployment
                    sh "kubectl apply -f workloads.yaml"
                }
            }
        }


  } //stages
   post {
        success {
            // Clean up the downloaded kubeconfig file
            sh "rm -f ${env.KUBECONFIG}"
        }
    }
}
