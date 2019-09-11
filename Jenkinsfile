pipeline {
    agent {
      node {label 'python'}
    }
    environment {
        APPLICATION_NAME = 'flask-test-openshift'
        GIT_REPO="https://github.com/acepabdurohman/flask-test-openshift.git"
        GIT_BRANCH="master"
        STAGE_TAG = "promoteToQA"
        DEV_PROJECT = "dev"
        STAGE_PROJECT = "stage"
        TEMPLATE_NAME = "flask-test-openshift"
        ARTIFACT_FOLDER = "target"
        PORT = 7070;
    }
    stages {
        stage('Get Latest Code') {
            steps {
                git branch: "${GIT_BRANCH}", url: "${GIT_REPO}"
            }
        }
        stage ("Install Dependencies") {
            steps {
                sh """
                pip install virtualenv
                virtualenv -p python3 venv
                source venv/bin/activate
                pip install -r app/requirements.pip
                deactivate
                """
            }
        }
        stage('Run Tests') {
            steps {
                sh '''
                source venv/bin/activate
                nosetests app --with-xunit
                deactivate
                '''
                junit "nosetests.xml"
            }
        }
        stage('Store Artifact'){
            steps{
                script{
                    def safeBuildName  = "${APPLICATION_NAME}_${BUILD_NUMBER}",
                        artifactFolder = "${ARTIFACT_FOLDER}",
                        fullFileName   = "${safeBuildName}.tar.gz",
                        applicationZip = "${artifactFolder}/${fullFileName}"
                        applicationDir = ["app",
                                            "config",
                                            "Dockerfile",
                                            ].join(" ");
                    def needTargetPath = !fileExists("${artifactFolder}")
                    if (needTargetPath) {
                        sh "mkdir ${artifactFolder}"
                    }
                    sh "tar -czvf ${applicationZip} ${applicationDir}"
                    archiveArtifacts artifacts: "${applicationZip}", excludes: null, onlyIfSuccessful: true
                }
            }
        }
        stage('Create Image Builder') {
            when {
                expression {
                    openshift.withCluster() {
                        openshift.withProject(DEV_PROJECT) {
                            return !openshift.selector("bc", "${TEMPLATE_NAME}").exists();
                        }
                    }
                }
            }
            steps {
                script {
                    openshift.withCluster() {
                    openshift.withProject(DEV_PROJECT) {
                        openshift.newBuild("--name=${TEMPLATE_NAME}", "--docker-image=docker.io/nginx:mainline-alpine", "--binary=true")
                        }
                    }
                }
            }
        }
        stage('Build Image') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject(env.DEV_PROJECT) {
                            openshift.selector("bc", "$TEMPLATE_NAME").startBuild("--from-archive=${ARTIFACT_FOLDER}/${APPLICATION_NAME}_${BUILD_NUMBER}.tar.gz", "--wait=true")
                        }
                    }
                }
            }
        }
        stage('Deploy to DEV') {
            when {
                expression {
                    openshift.withCluster() {
                        openshift.withProject(env.DEV_PROJECT) {
                            return !openshift.selector('dc', "${TEMPLATE_NAME}").exists()
                        }
                    }
                }
            }
        }
        steps {
            script {
                openshift.withCluster() {
                    openshift.withProject(env.DEV_PROJECT) {
                        def app = openshift.newApp("${TEMPLATE_NAME}:latest")
                        app.narrow("svc").expose("--port=${PORT}");
                        def dc = openshift.selector("dc", "${TEMPLATE_NAME}")
                        while (dc.object().spec.replicas != dc.object().status.availableReplicas) {
                            sleep 10
                        }
                    }
                }
            }
        }
    }
}