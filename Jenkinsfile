// GitOps flow: build the image, render the Helm chart, and commit the rendered manifests to
//   https://github.com/elevy99927-demos/gitops  ->  systems/<DEPARTMENT>/<ENV>/<user>.yaml
// Argo CD (ApplicationSet/systems-manifests.yaml) deploys that file.

properties([
  parameters([
    choice(name: 'ENV', choices: ['dev', 'qa', 'prd'], description: 'Environment folder: systems/<DEPARTMENT>/<ENV>/'),
    string(name: 'DEPARTMENT', defaultValue: 'devops', description: 'Department folder: hr, finance, legal, digital, devops'),
    booleanParam(name: 'SERVICE_MONITOR', defaultValue: false, description: 'Render the ServiceMonitor (needs the Prometheus Operator CRDs in the cluster)')
  ])
])

def user        = "elevy99927"              // your name: Docker Hub user AND gitops file name
def appname     = "helloworld"
def appimage    = "${user}/${appname}"      // docker.io/<user>/helloworld
def apptag      = "${env.BUILD_NUMBER}"
def envName     = params.ENV ?: 'dev'
def department  = params.DEPARTMENT ?: 'devops'
def serviceMon  = params.SERVICE_MONITOR ? 'true' : 'false'
def gitopsRepo  = "github.com/elevy99927-demos/gitops.git"
def gitopsFile  = "systems/${department}/${envName}/${user}.yaml"

podTemplate(containers: [
      containerTemplate(name: 'jnlp', image: 'jenkins/inbound-agent', ttyEnabled: true),
      containerTemplate(name: 'deployer', image: 'elevy99927/k8s-deployer:latest', command: 'cat', ttyEnabled: true),
      containerTemplate(name: 'docker', image: 'gcr.io/kaniko-project/executor:v1.23.0-debug', command: '/busybox/cat', ttyEnabled: true)
  ],
  volumes: [
     secretVolume(mountPath: '/kaniko/.docker/', secretName: 'docker-cred'),               // Docker Hub login for <user>
     secretVolume(mountPath: '/var/run/secrets/github-token', secretName: 'github-token')  // GitHub token with write access to elevy99927-demos/gitops
  ])  {
    node(POD_LABEL) {
        stage('checkout') {
            container('jnlp') {
                sh '/usr/bin/git config --global http.sslVerify false'
                checkout scm
            }
        }

        stage('build') {
            container('docker') {
                echo "Building ${appimage}:${apptag} with Kaniko..."
                sh "/kaniko/executor --force --context=dir://${env.WORKSPACE} --destination=${appimage}:${apptag}"
            }
        }

        stage('deploy') {
            container('deployer') {
                echo "Rendering chart into ${gitopsRepo} -> ${gitopsFile}"
                sh """#!/bin/sh
                    set -e
                    command -v git >/dev/null || apk add --no-cache git
                    GIT_TOKEN=\$(cat /var/run/secrets/github-token/token)

                    rm -rf gitops
                    git clone https://x-access-token:\${GIT_TOKEN}@${gitopsRepo} gitops
                    cd gitops
                    git config user.name  "Jenkins (${user})"
                    git config user.email "elevy99927-demos@users.noreply.github.com"

                    mkdir -p systems/${department}/${envName}
                    helm template ${user} ${env.WORKSPACE}/chart \\
                        --set image.repository=${appimage} \\
                        --set image.tag=${apptag} \\
                        --set serviceMonitor.enabled=${serviceMon} \\
                        > ${gitopsFile}

                    git add ${gitopsFile}
                    if git diff --cached --quiet; then
                        echo "No changes in ${gitopsFile} - nothing to deploy"
                        exit 0
                    fi
                    git commit -m "Deploy ${appimage}:${apptag} to ${department}/${envName} (build ${env.BUILD_NUMBER})"

                    # the whole class pushes to the same repo - rebase and retry on conflict
                    for i in 1 2 3 4 5; do
                        if git push origin HEAD:main; then exit 0; fi
                        echo "push rejected, rebasing (attempt \$i)"
                        git pull --rebase origin main
                        sleep \$i
                    done
                    echo "push failed after 5 attempts"; exit 1
                """
            }
        }

        stage('add-dashboard') {
            container('deployer') {
                sh """
                    echo " TBD: Add dashboard to grafana"
                """
            }
        }
    }
  }
