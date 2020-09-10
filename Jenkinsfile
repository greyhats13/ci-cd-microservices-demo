// ::DEFINE 
def image_name          = "gomicroservice"
def service_name        = "gomicroservice"
def repo_name           = "ci-cd-microservices-demo"
def unitTest_standard   = "80.0%"

// ::URL
def repo_url            = "https://github.com/greyhats13/${repo_name}.git"
def ecr_url             = "dkr.ecr.ap-southeast-1.amazonaws.com"
def ecr_credentialsId   = "ecr:ap-southeast-1:awscredentials"
def helm_git            = "https://github.com/greyhats13/helm-microservices-demo.git"
def artifact_url        = "http://nexus.blast.co.id/repository"
def spinnaker_webhook   = "http://spinnaker-login.blast.co.id/webhooks/${service_name}"
def automation_repo     = "https://github.com/greyhats13/${repo_name}.git"
def k6_repo             = "https://github.com/greyhats13/loadtest.git"

// ::NOTIFICATIONS
def telegram_url        = "https://api.telegram.org/bot1294113089:AAGIfRq_iAKckaeRgRqXxPwfR3yFpVtEKUw/sendMessage"
def telegram_chatid     = "-xxxxxxxxx"
def emails              = "imam.arief.rhmn@gmail.com"
def job_success         = "SUCCESS"
def job_error           = "ERROR"

// ::INITIALIZATION
def fullname            = "${service_name}-${env.BUILD_NUMBER}"
def version, helm_dir, runPipeline, unitTest_score

// ::POD-TEMPLATE
// Pod template setup for Jenkins slave should the CI/CD process is taking place.
podTemplate(
    label: fullname,
    containers: [
        //container template to perform docker build and docker push operation
        containerTemplate(name: 'docker', image: 'docker.io/docker', command: 'cat', ttyEnabled: true),
        //container template to perform helm command such as helm packaging
        containerTemplate(name: 'helm', image: 'docker.io/alpine/helm', command: 'cat', ttyEnabled: true),
        //container template to perform curl command
        containerTemplate(name: 'curl', image: 'centos', command: 'cat', ttyEnabled: true)
    ],
    volumes: [
        //the mounting for container
        hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock')
    ]) 
{

    node(fullname) {
      try {
            stage("Checkout") {
              runPipeline = 'UAT'
              //checkout process to Source Code Management
              def scm = checkout([$class: 'GitSCM', branches: [[name: runBranch]], userRemoteConfigs: [[credentialsId: 'github-auth-token', url: repo_url]]])
              echo "Running UAT Pipeline with ${scm.GIT_BRANCH} branch"
              //define version and helm directory
              version     = "debug"
              helm_dir    = "test"
            }
            stage("Unit Test") {                    
              def golang = tool name: 'golang', type: 'go'
              withEnv(["GOROOT=${golang}", "PATH+GO=${golang}/bin"]) {
                echo "Running Unit Test"
                // goEnv(lib_url: lib_url, lib_url_auth: lib_url_auth)
                try {
                  //call unit test function
                  unitTest()
                } catch (e) {
                  echo "${e}"
                }

                //get unit test value
                def unitTestGetValue = sh(returnStdout: true, script: 'go tool cover -func=coverage.out | grep total | sed "s/[[:blank:]]*$//;s/.*[[:blank:]]//"')
                unitTest_score = "Your score is ${unitTestGetValue}"
                echo "${unitTest_score}"
                //check if unit test value meet the given standard
                if (unitTestGetValue >= unitTest_standard){
                   echo "Unit Test fulfill standar value with score ${unitTestGetValue}/${unitTest_standard}"
                } else {
                    currentBuild.result = 'ABORTED'
                    error("Sorry your unit test score not fulfill standard score ${unitTestGetValue}/${unitTest_standard}")
                  }
              }
            }  
            stage("Code Review") {
              echo "Running Code Review with SonarQube"
              def scannerHome = tool "sonarscanner"
              withSonarQubeEnv ("sonarserver") {
                //call sonarScanGo function to perform codereview
                sonarScanGo(scannerHome: scannerHome, project_name: fullname, image_name: image_name, project_version: "${version}")
              }
              timeout(time: 10, unit: 'MINUTES') {
                waitForQualityGate abortPipeline: true
              }
            }
            stage("Artifactory") {
              parallel(
                //use container slave for docker to perform docker build and push
                'Container': {
                    stage('Build Container') {
                      container('docker') {
                        dockerBuild(ecr_url: ecr_url, image_name: image_name, image_version: version)
                      }
                    }
                    stage('Push Container') {
                      container('docker') {
                        docker.withRegistry("https://${ecr_url}", ecr_credentialsId) {
                          // ::LATEST
                          dockerPush(ecr_url: ecr_url, image_name: image_name, image_version: version)
                          // ::VERSION
                          dockerPushTag(ecr_url: ecr_url, image_name: image_name, srcVersion: version, dstVersion: "${version}-${BUILD_NUMBER}")
                        }
                      }
                    }
                },
                'Chart': {
                    stage('Packaging') {
                      //create helm directory
                      sh "mkdir helm"
                      //use container as jenkins slave for docker to perform helm linting, dry run, and helm pacakaging
                      container('helm') {
                        dir('helm') {
                          //perform checkout to branch master
                          checkout([$class: 'GitSCM', branches: [[name: '*/master']], userRemoteConfigs: [[credentialsId: 'github-auth-token', url: helm_git]]])
                          //get into helm directory
                          dir(helm_dir) {
                            //call helm lint function to check wheter there is syntax error in helm chart templates or values.yaml
                            helmLint(service_name)
                              //call helmDryRun function to simulate the deployment of helm should any error is triggered, but not peforming the actual deployment to cluster 
                              helmDryRun(name: service_name, service_name: service_name)
                                //call helm lint function to check wheter there is syntax error in helm chart templates or values.yaml
                                helmPackage(service_name)
                          }
                        }
                      }
                    }
                    stage('Push Chart') {
                      echo "Push Chart to Artifactory"
                      //use container as jenkins slave for curlto perform curl command to upload helm packaging to Nexus
                      container('curl') {
                        dir("helm") {
                          dir(helm_dir) {
                            //call the function push artifact to upload the artififact (helm.tgz) to Nexus using curl command
                            pushArtifact(name: "helm", artifact_url: artifact_url, artifact_repo: helm_dir)
                          }
                        }
                      }
                    }
                },
                'Binary': {
                    stage('Build Binary') {
                      def golang = tool name: 'golang', type: 'go'
                      withEnv(["GOROOT=${golang}", "PATH+GO=${golang}/bin"]) {
                        goBuild (name : service_name)
                        goBuild (name : "${service_name}-${BUILD_NUMBER}")
                      }
                    }
                    stage('Push Binary') {
                      pushArtifact(name: service_name, artifact_url: artifact_url, artifact_repo: "gobinary")
                      pushArtifact(name: "${service_name}-${BUILD_NUMBER}", artifact_url: artifact_url, artifact_repo: "gobinary")
                    }
                }
              )
            }
            stage("Notifications") {
              notifications(telegram_url: telegram_url, telegram_chatid: telegram_chatid, slack_channel: slack_channel, slack_colour: slack_colour, emails: emails,
                  job: env.JOB_NAME, job_numb: env.BUILD_NUMBER, job_url: env.BUILD_URL, job_status: job_success, unitTest_score: unitTest_score
              )
            }
      } catch (e) {
          stage("Error"){
                notifications(telegram_url: telegram_url, telegram_chatid: telegram_chatid,
                    job: env.JOB_NAME, job_numb: env.BUILD_NUMBER, job_url: env.BUILD_URL, job_status: job_error, unitTest_score: unitTest_score
                )
          }
       }
}

//function to perform code review using sonarcube
def sonarScanGo(Map args) {
    sh "${args.scannerHome}/bin/sonar-scanner -X \
    -Dsonar.projectName=${args.project_name}\
    -Dsonar.projectKey=mcs:go:${args.image_name}\
    -Dsonar.projectVersion=${args.project_version}\
    -Dsonar.sources=. \
    -Dsonar.go.file.suffixes=.go \
    -Dsonar.tests=. \
    -Dsonar.test.inclusions=**/**_test.go \
    -Dsonar.test.exclusions=**/vendor/** \
    -Dsonar.sources.inclusions=**/**.go \
    -Dsonar.exclusions=**/**.xml \
    -Dsonar.go.exclusions=**/*_test.go,**/vendor/**,**/testdata \
    -Dsonar.tests.reportPaths=report-tests.out \
    -Dsonar.go.govet.reportPaths=report-vet.out \
    -Dsonar.go.coverage.reportPaths=coverage.out"
}

// def goEnv(Map args) {
//     sh "go env -w GOSUMDB=off"
//     sh "go env -w CGO_ENABLED=0"
//     sh "git config --global url.'${args.lib_url_auth}'.insteadOf '${args.lib_url}'"
// }

//function to perform unit test on golang microservices
def unitTest() {
    sh "go vet > report-vet.out"
    sh "go test ./... -covermode=count -coverprofile coverage.out"
    sh "go tool cover -func=coverage.out"
}

//function to perform build golang binary
def goBuild(Map args) {
    sh "go build -o ${args.name}"
}


//function to perform docker build that is defined in dockerfile
def dockerBuild(Map args) {
    sh "docker build --network host -t ${args.ecr_url}/${args.image_name}:${args.image_version} ."
}

// def dockerPushTag(Map args) {
//     sh "docker tag ${args.ecr_url}/${args.image_name}:${args.srcVersion} ${args.ecr_url}/${args.image_name}:${args.dstVersion}"
//     sh "docker push ${args.ecr_url}/${args.image_name}:${args.dstVersion}"
// }

//function to perform docker push to Private Docker registry which is Elastic Container Registry
def dockerPush(Map args) {
    sh "docker push ${args.ecr_url}/${args.image_name}:${args.image_version}"
}

// def dockerPull(Map args) {
//     sh "docker pull ${args.ecr_url}/${args.image_name}:${args.image_version}"
// }

//Function to perform helm linting to check any syntax error in helm chart
def helmLint(String service_name) {
    echo "Running helm lint"
    sh "helm lint ${service_name}"
}

//Function to perform helm dry run to simulate deployment but not really deploy to k8s.
def helmDryRun(Map args) {
    echo "Running dry-run deployment"
    sh "helm install --dry-run --debug ${args.name} ${args.service_name} -n exercise"
}

//Function to perform helm package into compressed format .tgz
def helmPackage(String service_name) {
    echo "Running Helm Package"
    sh "helm package ${service_name}"
}

//Function to perform pushArtifact that will push helm chart tgz to private registry (nexus)
def pushArtifact(Map args) {
    echo "Upload to Artifactory Server"
    if (args.name == "helm") {
        sh "curl -v -n -u admin:admin123 -T *.tgz ${args.artifact_url}/${args.artifact_repo}/"        
    } else {
        sh "curl -v -n -u admin:admin123 -T ${args.name} ${args.artifact_url}/${args.artifact_repo}/"
    }
}

//function to deploy helm chart to spinnaker by triggering the spinnaker pipeline via webhook, and wait for the spinnaker to trigger jenkins back when deployment is done.
def deploySpinnaker(Map args) {
    def hook = registerWebhook()
    echo "Hi Spinnaker!"
    sh "curl ${args.spinnaker_webhook}-${args.version} -X POST -H 'content-type: application/json' -d '{ \"parameters\": { \"jenkins-url\": \"${hook.getURL()}\" }}'"
    def data = waitForWebhook hook
    echo "Webhook called with data: ${data}"
}

//function to perform notification should the pipeline is either failed or success.
def notifications(Map args) {
    def message = "CICD Pipeline ${args.job} ${args.job_status} with build ${args.job_numb} \n\n More info at: ${args.job_url} \n\n Unit Test: ${args.unitTest_score} \n\n Total Time : ${currentBuild.durationString}"
    parallel(
        "Telegram": {
            sh "curl -s -X POST ${args.telegram_url} -d chat_id=${args.telegram_chatid} -d text='${message}'"
        },
        "Slack": {
            //slackSend color: "${args.slack_colour}", message: "${message}", channel: "${args.slack_channel}"
        },
        "Email": {
            //emailext body: "${message}", subject: "Jenkins Build ${args.job_status}: Job ${args.job}", to: "${args.emails}"
        }
    )
}