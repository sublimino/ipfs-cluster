pipeline {
  agent none

  environment {
    CONTAINER_TAG = 'latest'
    ENVIRONMENT = 'ops'
    GIT_CREDENTIALS = "ssh-key-jenkins-bot"
  }

  stages {
    stage('Build') {
      agent {
        docker {
          image 'docker.io/controlplane/gcloud-sdk:latest'
          args '-v /var/run/docker.sock:/var/run/docker.sock ' +
              '--user=root ' +
              '--cap-drop=ALL ' +
              '--cap-add=DAC_OVERRIDE'
        }
      }

      steps {
        ansiColor('xterm') {
          sh 'make docker-build-test CONTAINER_TAG="${CONTAINER_TAG}"'
        }
      }
    }

    stage('Test') {
      agent {
        docker {
          image 'docker.io/controlplane/gcloud-sdk:latest'
          args '-v /var/run/docker.sock:/var/run/docker.sock ' +
              '--user=root ' +
              '--cap-drop=ALL ' +
              '--cap-add=DAC_OVERRIDE'
        }
      }
      environment {
        HOME = "/tmp/home/"
        TEST_FILE = "test/test-localhost-remote.yaml"
      }

      steps {
        ansiColor('xterm') {
          sh "make docker"
        }
      }
    }

    stage('Push') {
      agent {
        docker {
          image 'docker.io/controlplane/gcloud-sdk:latest'
          args '-v /var/run/docker.sock:/var/run/docker.sock ' +
              '--user=root ' +
              '--cap-drop=ALL ' +
              '--cap-add=DAC_OVERRIDE'
        }
      }

      environment {
        DOCKER_REGISTRY_CREDENTIALS = credentials("${ENVIRONMENT}_docker_credentials")
      }

      steps {
        ansiColor('xterm') {
          sh """
            echo '${DOCKER_REGISTRY_CREDENTIALS_PSW}' \
            | docker login \
              --username '${DOCKER_REGISTRY_CREDENTIALS_USR}' \
              --password-stdin


            make docker-push-test CONTAINER_TAG='${CONTAINER_TAG}'
          """
        }
      }
    }

    stage('Kubernetes Test') {
      agent {
        docker {
          image 'docker.io/controlplane/gcloud-sdk:latest'
          args '-v /var/run/docker.sock:/var/run/docker.sock ' +
              '--user=root ' +
              '--cap-drop=ALL ' +
              '--cap-add=DAC_OVERRIDE'
        }
      }

      options {
        timeout(time: 25, unit: 'MINUTES')
        retry(1)
        timestamps()
      }

      environment {
        DOCKER_REGISTRY_CREDENTIALS = credentials("${ENVIRONMENT}_docker_credentials")
        SSH_CREDENTIALS = credentials("dev-digitalocean_ssh_credentials")
        K8S_MASTER_HOST = "167.99.195.141"
      }

      steps {
        ansiColor('xterm') {
          sh """#!/bin/bash

            set -euxo pipefail  
            
            trap 'rm -rf ~/.ssh/id_rsa /tmp/admin.conf' EXIT
            
            mkdir -p ~/.ssh
            echo '$SSH_CREDENTIALS' | base64 -d > ~/.ssh/id_rsa
            chmod 600 ~/.ssh/id_rsa
      
            ssh-keyscan \${K8S_MASTER_HOST} >> ~/.ssh/known_hosts
            scp root@\${K8S_MASTER_HOST}:/etc/kubernetes/admin.conf /tmp/
            
            export KUBECONFIG=/tmp/admin.conf
            sed -E "s,(server: https://)[^:]*(:6443.*),\\1\${K8S_MASTER_HOST}\\2,g" -i \${KUBECONFIG}

            kubectl get nodes

            export GOPATH="\$(pwd)/gopath"
            mkdir "\${GOPATH}/src" -p
            cd "\${GOPATH}/src"
            
            rm -rf kubernetes-ipfs || true
            git clone https://github.com/sublimino/kubernetes-ipfs.git
            cd kubernetes-ipfs

            ls -lasp

            CID_FILE=\$(mktemp --dry-run) 
            echo "\${CID_FILE}"
            
            docker pull controlplane/kubernetes-ipfs:latest
            docker run \
              -d \
              --cidfile=\${CID_FILE} \
              controlplane/kubernetes-ipfs:latest \
                sleep infinity
            CID=\$(cat "\${CID_FILE}")
            docker cp \${CID}:/app/src/kubernetes-ipfs/kubernetes-ipfs ./kubernetes-ipfs
            docker kill \${CID}            
            
            ./kubernetes-ipfs --help || true
            
            hr () 
            { 
                printf '=%.0s' \$(seq \$(tput cols));
                echo
            }

            ./init.sh
            
            for YML in \$(find tests -type f -name '*.yml' | grep -Ev 'template.yml\$'); do 
              hr
              echo "Starting: \${YML}"; 
              hr
              ./kubernetes-ipfs "\${YML}" || exit \$?; 
              echo; 
              hr
              echo "Ending: \${YML}"; 
              hr; 
            done   

          """
        }
      }
    }
  }
}
