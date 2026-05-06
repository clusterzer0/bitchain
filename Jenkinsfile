pipeline {
  agent any

  environment {
    NEXUS_URL = 'https://nexus.softsurve.com'
  }

  stages {

    // ── Pre-flight ─────────────────────────────────────────────────────────────
    stage('Pre-flight') {
      steps {
        script {
          def msg = sh(script: 'git log -1 --pretty=%B', returnStdout: true).trim()
          if (msg.contains('[skip ci]')) {
            currentBuild.result = 'NOT_BUILT'
            error('Commit contains [skip ci] — aborting.')
          }
        }
      }
    }

    // ── Build & Test ───────────────────────────────────────────────────────────
    stage('Build & Test') {
      agent {
        docker {
          image 'rust:slim'
          reuseNode true
        }
      }
      steps {
        sh '''
          cargo fmt --check
          cargo clippy -- -D warnings
          cargo test
          cargo build --release
        '''
      }
      post {
        success {
          archiveArtifacts artifacts: 'target/release/bitchain', allowEmptyArchive: false
        }
      }
    }

    // ── Publish → Nexus (Cargo) ────────────────────────────────────────────────
    stage('Publish') {
      agent {
        docker {
          image 'rust:slim'
          reuseNode true
        }
      }
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'nexus-credentials',
          usernameVariable: 'NEXUS_USER',
          passwordVariable: 'NEXUS_PASS'
        )]) {
          sh '''
            mkdir -p ~/.cargo
            cat >> ~/.cargo/config.toml <<EOF
[registries.lockamy]
index = "${NEXUS_URL}/repository/cargo-hosted/"

[registry]
default = "lockamy"
EOF
            printf '[registries.lockamy]\ntoken = "%s:%s"\n' \
              "${NEXUS_USER}" "${NEXUS_PASS}" >> ~/.cargo/credentials.toml
            chmod 0600 ~/.cargo/credentials.toml

            cargo publish --registry lockamy
          '''
        }
      }
    }

  }

  post {
    success {
      echo 'bitchain published to nexus.softsurve.com/repository/cargo-hosted/'
    }
    failure {
      echo 'Pipeline failed.'
    }
    always {
      cleanWs()
    }
  }
}
