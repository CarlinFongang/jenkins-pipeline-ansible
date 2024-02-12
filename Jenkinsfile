pipeline {
    agent none
    stages {
        stage('Check yaml syntax') {
            agent { docker { image 'sdesbure/yamllint' } }
            steps {
                sh 'yamllint --version'
                sh 'yamllint \${WORKSPACE}'
            }
        }
        stage('Check markdown syntax') {
            agent { docker { image 'ruby:alpine' } }
            steps {
                sh 'apk --no-cache add git'
                sh 'gem install mdl'
                sh 'mdl --version'
                sh 'mdl --style all --warnings --git-recurse \${WORKSPACE}'
            }
        }
        stage('Prepare ansible environment') {
            agent any
            environment {
                VAULTKEY = credentials('vaultkey')
            }
            steps {
                sh 'echo \$VAULTKEY > vault.key'
            }
        }
        stage('Test and deploy the application') {
            environment {
                //SUDOPASS = credentials('sudopass')
                EC2_SSH_KEY = credentials('ec2_ssh_key') // Nouvelle clé SSH pour accéder à l'instance EC2
                EC2_HOST = '44.201.61.29' // Remplacez par l'adresse IP publique ou le DNS de votre instance EC2
                EC2_USER = 'ubuntu' // Remplacez par l'utilisateur SSH de votre instance EC2
            }
            agent { docker { image 'registry.gitlab.com/op_so/docker/ansible/master:de818487' } }
            stages {
               stage("Verify ansible playbook syntax") {
                   steps {
                       sh 'ansible-lint deploy.yml'
                   }
               }
               stage("Deploy app in production") {
                    when {
                       expression { GIT_BRANCH == 'origin/master' }
                    }
                   steps {
                       sh '''
                       sudo apt-get update                                       

                       ssh -o StrictHostKeyChecking=no -i "$EC2_SSH_KEY" "$EC2_USER@$EC2_HOST" 'sudo apt-get update && sudo apt-get install -y docker.io'
                       ssh -o StrictHostKeyChecking=no -i "$EC2_SSH_KEY" "$EC2_USER@$EC2_HOST" 'sudo systemctl start docker && sudo systemctl enable docker'
                       ssh -o StrictHostKeyChecking=no -i "$EC2_SSH_KEY" "$EC2_USER@$EC2_HOST" 'logout'
                       ssh -o StrictHostKeyChecking=no -i "$EC2_SSH_KEY" "$EC2_USER@$EC2_HOST" 'sudo apt-get install -y sshpass'
                       ssh -o StrictHostKeyChecking=no -i "$EC2_SSH_KEY" "$EC2_USER@$EC2_HOST" 'ansible-playbook  -i hosts.yml --vault-password-file vault.key  --extra-vars "ansible_sudo_pass=$SUDOPASS" deploy.yml'
                       '''
                   }
               } 
            }
          }
      }
    }
