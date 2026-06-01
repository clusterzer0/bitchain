// Centralized, pinned Rust toolchain image — bump here to move every stage at once.
// Build, test, and publish run inside this container so the Rust version stays
// decoupled from the Jenkins agent host. Pinned to latest stable as of 2026-06.
def RUST_IMAGE = 'rust:1.94'

pipeline {
  agent any

  environment {
    NEXUS_URL  = 'https://nexus.softsurve.com'
    CARGO_HOME = "${WORKSPACE}/.cargo"
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
          image RUST_IMAGE
          reuseNode true
        }
      }
      steps {
        sh '''
          rustup component add rustfmt clippy
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
          image RUST_IMAGE
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
            mkdir -p "$CARGO_HOME"
            cat >> "$CARGO_HOME/config.toml" <<EOF
[registries.lockamy]
index = "${NEXUS_URL}/repository/cargo-hosted/"

[registry]
default = "lockamy"
EOF
            printf '[registries.lockamy]\ntoken = "%s:%s"\n' \
              "${NEXUS_USER}" "${NEXUS_PASS}" >> "$CARGO_HOME/credentials.toml"
            chmod 0600 "$CARGO_HOME/credentials.toml"

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
