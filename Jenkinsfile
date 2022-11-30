def pipelineType
def lasttag

pipeline {
    agent any
    tools {
        maven 'maven'
    }
    environment{
        GIT_AUTH = credentials('token-danilo')
        commitId = sh(returnStdout: true, script: 'git rev-parse HEAD')
        commitcheck = sh(returnStdout: true, script: '[ "$CHANGE_BRANCH" = "" ] || git rev-parse origin/$CHANGE_BRANCH')
        result = sh(returnStdout: true, script: 'git log --format=%B HEAD -n1')
        author = sh(returnStdout: true, script: 'git log --pretty=%an HEAD -n1')
        GIT_COMMIT_USERNAME = sh (script: 'git show -s --pretty=%an', returnStdout: true ).trim()
        commitmsg = sh(returnStdout: true, script: 'git log --oneline -n 1 HEAD')
        projectname = sh(script: 'git remote get-url origin | cut -d "/" -f5', returnStdout: true)
        commiteremail = sh(returnStdout: true, script: 'git log --pretty=%ae HEAD -n1')
        jenkinsurl = sh(script: 'echo "${BUILD_URL}"' , returnStdout: true).trim()

    }
    stages {
        stage('Checkout') {
            steps {
                cleanWs()
                checkout scm
            }
        }
        stage("CD o CI") {
            steps {
                script {
                    if(env.BRANCH_NAME == 'main'){
                        pipelineType = "Pipeline CD"
                    }else{
                        pipelineType = "Pipeline CI"
                    }
                }
            }
        }
        stage('Validate Commit'){
            when { branch 'feature-*' }
            steps{
                script{
                    if(env.commitmsg.contains("major") || env.commitmsg.contains("minor") || env.commitmsg.contains("patch")){

                        echo "Commit Mensaje Valido"
                    }else{
                        slackSend color: "danger", message: "Grupo 3 - " + pipelineType + " - Rama : " + env.BRANCH_NAME + " - Stage : " + env.STAGE_NAME + " - Fail"
                        echo "no contiene ninguna palabra"
                        currentBuild.result = 'ABORTED'
                        error("No se ha encontrado ninguna palabra clave para el incremento de version")
                    }
                }
            }
        }
        stage("Build"){
            when { anyOf { branch 'feature-*'; branch 'main' } }
            steps {
                sh './mvnw clean compile -e'
            }
            post{
                failure {
                    slackSend color: "danger", message: "Grupo 3 - " + pipelineType + " - Rama : " + env.BRANCH_NAME + " - Stage : " + env.STAGE_NAME + " - Fail"
                }
            }
        }
        stage('Test') {
            when { anyOf { branch 'feature-*'; branch 'main' } }
            steps {
                sh './mvnw test -e'
            }
            post{
                failure {
                    slackSend color: "danger", message: "Grupo 3 - " + pipelineType + " - Rama : " + env.BRANCH_NAME + " - Stage : " + env.STAGE_NAME + " - Fail"
                }
            }
        }
        stage('SonarQube') {
            when { anyOf { branch 'feature-*'; branch 'main' } }
            steps {
                withSonarQubeEnv('sonar-public') {
                    sh './mvnw clean package sonar:sonar'
                }
            }
            post{
                failure {
                    slackSend color: "danger", message: "Grupo 3 - " + pipelineType + " - Rama : " + env.BRANCH_NAME + " - Stage : " + env.STAGE_NAME + " - Fail"
                }
            }
        }
        stage('pull request rama feature-*'){
            when { branch 'feature-*' }
            steps{
                script {
                    jsonObj='{"title":"{title}","body":"{body}","head":"{branchname}","base":"{main}"}'
                    jsonObj=jsonObj.replace("{title}", "PR Generado por Jenkins")
                    jsonObj=jsonObj.replace("{body}", "Body prueba")
                    jsonObj=jsonObj.replace("{branchname}", env.BRANCH_NAME)
                    jsonObj=jsonObj.replace("{main}", "main")                                      
                    echo "JSON: $jsonObj"
                    statusCode=sh(script: "curl -o /dev/null -s -w \"%{http_code}\" -X POST -H \"Accept: application/vnd.github+json\" -H \"Authorization: Bearer $GIT_AUTH_PSW\" https://api.github.com/repos/grupo3-seccion1/ms-iclab/pulls --data-raw '$jsonObj'", returnStdout: true)                         
                    echo "Resultado Pull request :  $statusCode" 
                    if(statusCode == "201"){
                        try {
                            sh 'git fetch --tags'
                            // lasttag = sh(returnStdout: true, script: 'git describe --abbrev=0 --tags')
                            lasttag = sh(returnStdout: true, script: 'git describe --tags `git rev-list --tags --max-count=1`')
                        }catch(Exception e){
                            lasttag = "v0.0.0"
                        }
                        echo "Ultimo tag: $lasttag"
                        echo "mensaje commit : "+env.commitmsg
                        lasttag = lasttag.trim()
                        echo "lasttag: "+lasttag
                        lasttag = lasttag.substring(1)
                        echo "lasttag: "+lasttag
                        lasttag = lasttag.split("\\.")
                        echo "lasttag: "+lasttag
                        //ver si mensaje contiene una palabra
                        if(env.commitmsg.contains("major")){
                            lasttag = (lasttag[0].toInteger()+1)+".0.0"
                        }else if(env.commitmsg.contains("minor")){
                            lasttag = lasttag[0]+"."+(lasttag[1].toInteger()+1)+"."+lasttag[2]
                        }else if(env.commitmsg.contains("patch")){
                            lasttag = lasttag[0]+"."+lasttag[1]+"."+(lasttag[2].toInteger()+1)
                        }else{
                            echo "no contiene ninguna palabra"
                            currentBuild.result = 'ABORTED'
                            slackSend color: "danger", message: "Grupo 3 - " + pipelineType + " - Rama : " + env.BRANCH_NAME + " - Stage : " + env.STAGE_NAME + " - Fail"
                            error("No se ha encontrado ninguna palabra clave para el incremento de version")
                        }
                        echo "lasttag: "+lasttag
                        sh "git tag -a v"+lasttag+" -m 'v"+lasttag+"'"
                        sh "git config --global user.email 'danilovidalm@gmail.com'"
                        sh "git config --global user.name 'Jenkins'"
                        try{
                            sh ('''
                                git config --local credential.helper "!f() { echo username=\\$GIT_AUTH_USR; echo password=\\$GIT_AUTH_PSW; }; f"
                                git push --tags
                            ''')
                            slackSend color: "good", message: "Grupo 3 - " + pipelineType + " - Rama : " + env.BRANCH_NAME + " - Stage : " + env.STAGE_NAME + " - Success"
                        }catch(Exception e){
                            echo "Error al hacer push de tag"
                            slackSend color: "danger", message: "Grupo 3 - " + pipelineType + " - Rama : " + env.BRANCH_NAME + " - Stage : " + env.STAGE_NAME + " - Fail"
                        }
                    }else{
                        slackSend color: "danger", message: "Grupo 3 - " + pipelineType + " - Rama : " + env.BRANCH_NAME + " - Stage : " + env.STAGE_NAME + " - Fail"
                    }
                }
            }
        }
        stage('Package'){
            when { branch 'main' }
            steps {
                sh './mvnw package -e'
            }
            post{
                failure {
                    slackSend color: "danger", message: "Grupo 3 - " + pipelineType + " - Rama : " + env.BRANCH_NAME + " - Stage : " + env.STAGE_NAME + " - Fail"
                }
            }
        }
        stage('Upload Nexus'){
            when { branch 'main' }
            steps {
                script{
                    sh 'git fetch --tags'
                    // lasttag = sh(returnStdout: true, script: 'git describe --abbrev=0 --tags')
                    lasttag = sh(returnStdout: true, script: 'git describe --tags `git rev-list --tags --max-count=1`')
                    lasttag = lasttag.trim()
                    lasttag = lasttag.substring(1)
                    echo "lasttag: "+lasttag
                }
                nexusPublisher nexusInstanceId: 'nexus01', nexusRepositoryId: 'devops-usach-nexus', packages: [[$class: 'MavenPackage', mavenAssetList: [[classifier: '', extension: '', filePath: './build/DevOpsUsach2020-0.0.1.jar']], mavenCoordinate: [artifactId: 'DevOpsUsach2020', groupId: 'com.devopsusach2020', packaging: 'jar', version: lasttag]]]
            }
            post{
                failure {
                    slackSend color: "danger", message: "Grupo 3 - " + pipelineType + " - Rama : " + env.BRANCH_NAME + " - Stage : " + env.STAGE_NAME + " - Fail"
                }
            }
        }
        stage('Download Artifact From Nexus'){
            when { branch 'main' }
            steps{
                sh "curl -X GET -u admin:1qazxsw2 https://nexus.danilovidalm.com/repository/devops-usach-nexus/com/devopsusach2020/DevOpsUsach2020/"+lasttag+"/DevOpsUsach2020-"+lasttag+".jar -O"
            }
            post{
                failure {
                    slackSend color: "danger", message: "Grupo 3 - " + pipelineType + " - Rama : " + env.BRANCH_NAME + " - Stage : " + env.STAGE_NAME + " - Fail"
                }
            }
        }
        stage('Run Artifact Jar'){
            when { branch 'main' }
            steps{
                sh "nohup java -jar ./DevOpsUsach2020-"+lasttag+".jar&"
            }
            post {
                failure {
                    slackSend color: "danger", message: "Grupo 3 - " + pipelineType + " - Rama : " + env.BRANCH_NAME + " - Stage : " + env.STAGE_NAME + " - Fail"
                }
            }
        }
        stage('sleep'){
            when { branch 'main' }
            steps{
                echo 'Sleeping...'
                sleep time: 20, unit: 'SECONDS'
                sh 'ps -ef | grep java'
            }
        }
        stage('CURL Localhost:8081'){
            when { branch 'main' }
            steps{
                script{
                    try{
                        responseStatus = sh (script: "curl -o /dev/null -s -w \"%{http_code}\" http://localhost:8081/rest/mscovid/estadoMundial", returnStdout: true)
                    }catch(Exception e){
                        echo "Error al hacer curl"
                        echo e
                        slackSend color: "danger", message: "Grupo 3 - " + pipelineType + " - Rama : " + env.BRANCH_NAME + " - Stage : " + env.STAGE_NAME + " - Fail"
                    }
                    if(responseStatus == "200"){
                        echo "Success"
                        slackSend color: "good", message: "Grupo 3 - " + pipelineType + " - Rama : " + env.BRANCH_NAME + " - Stage : " + env.STAGE_NAME + " - Success"
                    }else{
                        echo "Fail"
                        slackSend color: "danger", message: "Grupo 3 - " + pipelineType + " - Rama : " + env.BRANCH_NAME + " - Stage : " + env.STAGE_NAME + " - Fail"
                    }
                }

            }
        }
    }
}