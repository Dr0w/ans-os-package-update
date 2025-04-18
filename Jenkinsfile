pipeline {
  agent any
  environment {
    PATH = "/usr/local/bin:/usr/bin:/bin:$PATH"
  }
  parameters {
    string(name: 'TARGET_IP', defaultValue: '', description: 'Target IP/hostname')
    choice(name: 'CREDENTIALS',   choices: ['ubuntu-dev-creds','mac-dev-creds'], description: 'SSH key credentials')
    choice(name: 'SUDO_CREDENTIALS',   choices: ['ubuntu-sudo-creds','mac-sudo-creds'], description: 'Sudo‑password credentials')
    string(name: 'VAULT_CREDENTIALS',   defaultValue: 'ansible-vault-pass', description: 'Vault‑password (Secret Text) credentials ID')
    booleanParam(name: 'DO_CLEAN', defaultValue: false, description: 'Cleanup after update')
  }

  stages {
    stage('Preparation') {
      steps {
        script {
          if (params.TARGET_IP == '' || params.TARGET_IP == 'target-host') {
            error "Please set a valid TARGET_IP"
          }
        }
        checkout scm
      }
    }

    stage('Run Ansible Playbook') {
      steps {
        withCredentials([
          sshUserPrivateKey(credentialsId: params.CREDENTIALS,
                            keyFileVariable: 'SSH_KEY',
                            usernameVariable: 'SSH_USER'),
          usernamePassword(credentialsId: params.SUDO_CREDENTIALS,
                           usernameVariable: 'SUDO_USER',
                           passwordVariable: 'SUDO_PASS'),
          string(credentialsId: params.VAULT_CREDENTIALS,
                 variable: 'VAULT_PASS')
        ]) {
          script {
            // 1) Write vault password for Ansible
            sh """
              mkdir -p ansible
              echo "\$VAULT_PASS" > ansible/vault_pass.txt
              chmod 600 ansible/vault_pass.txt
            """

            // 2) Build the -e argument for cleanup
            def extraVars = "ansible_become_pass=\${SUDO_PASS}"
            if (params.DO_CLEAN) {
              extraVars += " clean=true"
            }

            // 3) Run the playbook against the single host
            sh """
              ansible-playbook \
                -i "${params.TARGET_IP}," \
                playbooks/update.yml \
                --user "\${SSH_USER}" \
                --private-key "\${SSH_KEY}" \
                --vault-password-file ansible/vault_pass.txt \
                --extra-vars "${extraVars}"
            """
          }
        }
      }
    }
  }

  post {
    always {
      script {
        // reuse the SSH credentials and fetch the remote update.log
        withCredentials([sshUserPrivateKey(credentialsId: params.CREDENTIALS,
                                           keyFileVariable: 'SSH_KEY',
                                           usernameVariable: 'SSH_USER')]) {
          def target = "${SSH_USER}@${params.TARGET_IP}"
          echo "Retrieving update.log from ${target}"
          sh """
            scp -o StrictHostKeyChecking=no -i \$SSH_KEY \
              ${target}:~/update.log ./update.log || echo 'update.log not found'
          """
          if (fileExists('update.log')) {
            def log = readFile('update.log').toLowerCase()
            if (log.contains('warning') || log.contains('error')) {
              error "Warnings or errors detected in update.log"
            } else {
              echo "No warnings/errors detected; cleaning up remote files..."
              sh """
                ssh -o StrictHostKeyChecking=no -i \$SSH_KEY ${target} \
                  'rm -f ~/update.log'
              """
            }
          } else {
            echo "Skipping log analysis: update.log missing"
          }
        }
      }
    }
  }
}
