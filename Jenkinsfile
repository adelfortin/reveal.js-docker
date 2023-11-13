#!groovy
@Library('github.com/cloudogu/ces-build-lib@1.35.2')
import com.cloudogu.ces.cesbuildlib.*

node('docker') {

    properties([
            // Keep only the last 10 build to preserve space
            buildDiscarder(logRotator(numToKeepStr: '10')),
            // Don't run concurrent builds for a branch, because they use the same workspace directory
            disableConcurrentBuilds(),
    ])

    Docker docker = new Docker(this)
    Git git = new Git(this)

    catchError {

        stage('Checkout') {
            checkout scm
            git.clean('')
            // Otherwise git.isTag() will not be reliable. Jenkins seems to do a sparse checkout only
            sh "git fetch --tags"
        }

        def devImage
        def prodImage
        String imageName = "cloudogu/reveal.js:${createVersion(git)}"

        stage('Build Images') {
            withEnv(['DOCKER_BUILDKIT=false']) {
                devImage = docker.build "${imageName}-dev", '--build-arg ENV=dev .'
                prodImage = docker.build imageName, '.'
            }
        }

        stage('Test Images') {
            smokeTest(devImage, '8000')
            smokeTest(prodImage, '8080')
        }

        stage('Deploy Images') {
            docker.withRegistry('', 'cesmarvin-dockerhub-access-token') {
                if (git.isTag()) {
                    devImage.push()
                    prodImage.push()
                }

                if (env.BRANCH_NAME == "master") {
                    devImage.push('dev')
                    prodImage.push('latest')
                }
            }
        }
    }

    mailIfStatusChanged(git.commitAuthorEmail)
}

String createVersion(git) {
    String versionName
    if (git.isTag()) {
        versionName = git.tag
        currentBuild.description = versionName
    } else {
        // E.g. "201708140933-1674930"
        versionName = "${new Date().format('yyyyMMddHHmm')}-${git.commitHashShort}"
    }
    echo "Building version ${versionName} on branch ${env.BRANCH_NAME}"

    return versionName
}

void smokeTest(def image, String port) {
    String expectedTitle='myTitle'
    Docker docker = new Docker(this)

    image.withRun("-e TITLE=${expectedTitle}") { container ->

        def ip = docker.findIp(container)
        if (!ip || !getAndTestTitle("http://${ip}:${port}", expectedTitle)) {
            echo "Container Smoke Test failed."
            echo "Container log:"
            echo new Sh(this).returnStdOut("docker logs ${container.id}")
        }
        
    }
}

void getAndTestTitle(String url, String title) {
    sh ("wget -O- --retry-connrefused --tries=30 -q --wait=1 ${url} | grep ${title} ")
}