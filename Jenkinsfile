pipeline {
  agent any
  options { ansiColor('xterm'); timestamps() }

  environment {
    LC_ALL = 'en_US.UTF-8'
    LANG   = 'en_US.UTF-8'
  }

  stages {
    stage('Checkout sample app') {
      steps {
        git url: 'https://github.com/Venkateshsandupatla/sample-apps-ios-simple-objc.git'
      }
    }

    stage('Tooling sanity') {
      steps {
        sh '''
          set -euo pipefail
          echo "=== Xcode ==="; xcodebuild -version
          echo "=== Ruby ==="; ruby -v
          echo "=== Fastlane ==="; fastlane --version
          echo "=== Cocoapods ==="; pod --version || true
          echo "=== Simulators ==="; xcrun simctl list devices | head -n 50
        '''
      }
    }

    stage('Dependencies (Pods if present)') {
      steps {
        sh '''
          set -euo pipefail
          if [ -f "Podfile" ]; then
            pod install
          fi
        '''
      }
    }

    stage('Fastlane setup') {
      steps {
        sh '''
          set -euo pipefail
          mkdir -p fastlane

          # Auto-pick a shutdown iPhone simulator (prefer latest iOS)
          DEVICE_NAME=$(xcrun simctl list devices | \
            awk -F '[()]' '/iPhone .*Shutdown/{print $(NF-1)}' | head -n1)

          if [ -z "$DEVICE_NAME" ]; then
            echo "No iPhone simulator found in Shutdown state; falling back to iPhone 16 Pro"
            DEVICE_NAME="iPhone 16 Pro"
          fi

          cat > fastlane/Fastfile <<'RUBY'
          default_platform(:ios)
          platform :ios do
            desc "Run unit tests on simulator"
            lane :unit_test do
              # Device is passed via ENV['SIM_DEVICE'] from Jenkins stage
              device = ENV['SIM_DEVICE'] || "iPhone 16 Pro"
              scan(
                scheme: "sample-apps-ios-simple-objc",
                devices: [device],
                build_for_testing: true,
                clean: true,
                output_types: "junit",
                output_directory: "fastlane/test_output"
              )
            end
          end
          RUBY

          # record the chosen device for visibility
          echo "$DEVICE_NAME" > .sim_device
        '''
      }
    }

    stage('Unit Tests (Simulator)') {
      steps {
        sh '''
          set -euo pipefail
          SIM_DEVICE=$(cat .sim_device)
          echo "Using simulator: $SIM_DEVICE"
          # try to boot it to avoid "no runtime" races
          UDID=$(xcrun simctl list devices | \
            awk -v dev="$SIM_DEVICE" -F '[()]' '$0 ~ dev && $0 ~ /Shutdown/ {print $(NF-1); exit}')
          if [ -n "$UDID" ]; then
            xcrun simctl boot "$UDID" || true
            open -a Simulator || true
          fi

          # run tests
          fastlane unit_test SIM_DEVICE="$SIM_DEVICE"
        '''
      }
    }
  }

  post {
    always {
      junit allowEmptyResults: true, testResults: 'fastlane/test_output/report.junit'
      archiveArtifacts artifacts: 'fastlane/test_output/**/*', allowEmptyArchive: true
      echo 'Sanity pipeline finished.'
    }
  }
}
