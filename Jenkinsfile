pipeline {
    agent none
    stages {
	stage('checkout SCM') {
		steps {	
                    checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[url: 'https://github.com/lusianneDjepang/student-list-jenkins-Sonar-CI-CD.git']]])
                }
	}	
        stage('Code Analysis') {
	    def scannerhome = tool 'sonar-scanner';
            withSonarQubeEnv ('SonarQube Server'){
		    steps {    
		        sh ' ${scannerhome}/bin/sonar-runner -D sonar.projectKey=ansible:demo -D sonar.sources=. '
		    }	    
            }
        }
        stage('Check bash syntax') {
            agent { docker { image 'koalaman/shellcheck-alpine:stable' } }
            steps {
                sh 'shellcheck --version'
                sh 'apk --no-cache add grep'
                sh '''
                for file in $(grep -IRl "#!(/usr/bin/env |/bin/)" --exclude-dir ".git" --exclude Jenkinsfile \${WORKSPACE}); do
                  if ! shellcheck -x $file; then
                    export FAILED=1
                  else
                    echo "$file OK"
                  fi
                done
                if [ "${FAILED}" = "1" ]; then
                  exit 1
                fi
                '''
            }
        }
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
                DEVOPSKEY = credentials('devopskey')
            }
            steps {
                sh 'echo \$VAULTKEY > vault.key'
                sh 'cp \$DEVOPSKEY id_rsa'
                sh 'chmod 600 id_rsa'
            }
        }
        stage('Test and deploy the application') {
            agent { docker { image 'registry.gitlab.com/robconnolly/docker-ansible:latest' } }
            stages {
               stage("Install ansible role dependencies") {
                   steps {
                       sh 'ansible-galaxy install -r roles/requirements.yml'
                   }
               }
               stage("Ping targeted hosts") {
                   steps {
                       sh 'ansible all -m ping -i hosts --private-key id_rsa'
                   }
               }
               stage("Vérify ansible playbook syntax") {
                   steps {
                       sh 'ansible-lint -x 306 install_student_list.yml student-list'
                       sh 'echo "${GIT_BRANCH}"'
                   }
               }
               stage("Build docker images on build host") {
                   when {
                      expression { GIT_BRANCH == 'origin/master' }
                  }
                   steps {
                       sh 'ansible-playbook  -i hosts --vault-password-file vault.key --private-key id_rsa --tags "build" --limit build install_student_list.yml'
                   }
               }
               stage("Deploy app in production") {
                    when {
                       expression { GIT_BRANCH == 'origin/master' }
                    }
                   steps {
                       sh 'ansible-playbook  -i hosts --vault-password-file vault.key --private-key id_rsa --tags "deploy" --limit prod install_student_list.yml'
                   }
               }
            }
        }
    }
}

