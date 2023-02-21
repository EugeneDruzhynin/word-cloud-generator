pipeline {
    agent any
    environment{
        NEXUS_REPO='word-cloud-build'
        BRANCH='pipeline'
    }
    stages {
        stage('Build app and upload to Nexus') {
            agent {
                docker { image 'golang:1.16' }
            }
            steps {
                git 'https://github.com/EugeneDruzhynin/word-cloud-generator.git'

                sh '''
                  pwd
                  make lint
                  make test'''

                sh '''export GOPATH=$WORKSPACE/go
                    export PATH="$PATH:$(go env GOPATH)/bin"
                    go get github.com/tools/godep
                    go get github.com/smartystreets/goconvey
                    go get github.com/GeertJohan/go.rice/rice
                    go get github.com/wickett/word-cloud-generator/wordyapi
                    go get github.com/gorilla/mux
                    sed -i "s/1.DEVELOPMENT/1.${BUILD_NUMBER}/g" static/version
                    GOOS=linux GOARCH=amd64 go build -o ./artifacts/word-cloud-generator -v
                    gzip -f artifacts/word-cloud-generator
                    ls -l artifacts'''

                nexusArtifactUploader(
                    nexusVersion: 'nexus3',
                    protocol: 'http',
                    nexusUrl: 'nexus:8081',
                    groupId: "${BRANCH}",
                    version: "1.${BUILD_NUMBER}",
                    repository: 'word-cloud-build',
                    credentialsId: 'nexus_password',
                    artifacts: [
                        [artifactId: 'world-cloud-generator',
                        classifier: '',
                        file: 'artifacts/word-cloud-generator.gz',
                        type: 'gz']
                    ]
                )
            }
        }
        stage ('Upload for testing') {
            agent {
                dockerfile {
                    filename '/vagrant/final/dockerfile-alpine'
                }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'nexus_password', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASSWORD')]) {
                    sh '''
                        curl -u ${NEXUS_USER}:${NEXUS_PASSWORD} -X GET "http://nexus:8081/repository/${NEXUS_REPO}/${BRANCH}/word-cloud-generator/1.${BUILD_NUMBER}/word-cloud-generator-1.${BUILD_NUMBER}.gz"  -o /opt/wordcloud/word-cloud-generator.gz"
                        gunzip -f /opt/wordcloud/word-cloud-generator.gz;chmod +x /opt/wordcloud/word-cloud-generator; sudo systemctl start wordcloud
                        sleep 10
                        res=`curl -s -H "Content-Type: application/json" -d '{"text":"ths is a really really really important thing this is"}' http://localhost:8888/version 
| jq '. | length'`
                        if [ "1" != "$res" ]; then
                        exit 99
                        fi

                        res=`curl -s -H "Content-Type: application/json" -d '{"text":"ths is a really really really important thing this is"}' http://localhost:8888/api | jq '. | length'`
                        if [ "7" != "$res" ]; then
                        exit 99
                        fi'''
                }
            }
        }
    }
}
