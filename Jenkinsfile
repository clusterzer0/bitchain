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
      when { branch 'main' }
      agent {
        docker {
          image RUST_IMAGE
          reuseNode true
        }
      }
      steps {
        withCredentials([
          usernamePassword(
            credentialsId: 'nexus-credentials',
            usernameVariable: 'NEXUS_USER',
            passwordVariable: 'NEXUS_PASS'
          ),
          usernamePassword(
            credentialsId: 'github-token',
            usernameVariable: 'GIT_USER',
            passwordVariable: 'GIT_TOKEN'
          )
        ]) {
          sh '''
            git config user.email "ci@softsurve.com"
            git config user.name  "Sol CI"

            set -eu

            origin=$(git config --get remote.origin.url)
            path=$(printf '%s' "$origin" | sed -E 's#(git@github.com:|https://github.com/)##; s#[.]git$##')
            remote="https://${GIT_USER}:${GIT_TOKEN}@github.com/${path}.git"

            # Tags are the version source of truth — bump the patch above the latest vX.Y.Z.
            git fetch --tags --quiet "$remote" || true
            latest=$(git tag -l 'v*.*.*' | sort -V | tail -1)
            if [ -z "$latest" ]; then
              next="0.1.0"
            else
              v=${latest#v}; maj=${v%%.*}; rest=${v#*.}; min=${rest%%.*}; pat=${rest##*.}
              next="${maj}.${min}.$((pat + 1))"
            fi
            echo "Publishing ${path} v${next}"

            # Set the crate version for this publish (ephemeral; tags stay the source of truth).
            if [ -f Cargo.toml ]; then
              sed -i 's/^version = ".*"/version = "'"$next"'"/' Cargo.toml
            fi

            # Cargo registry auth.
            mkdir -p "$CARGO_HOME"
            cat >> "$CARGO_HOME/config.toml" <<EOF
[registries.lockamy]
index = "sparse+${NEXUS_URL}/repository/cargo-group/"

[registry]
default = "lockamy"
EOF
            printf '[registries.lockamy]\ntoken = "%s:%s"\n' \
              "${NEXUS_USER}" "${NEXUS_PASS}" >> "$CARGO_HOME/credentials.toml"
            chmod 0600 "$CARGO_HOME/credentials.toml"

            # Publish (allow-dirty: the version was just set in-tree).
            cargo publish --registry lockamy --allow-dirty

            # Record the release as a tag and push it back to origin.
            git tag "v${next}"
            git push "$remote" "v${next}"
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
