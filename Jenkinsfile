pipeline {

  agent {
    node {
      label env.CI_SLAVE
    }
  }

  options {
    timestamps()
  }

  parameters {
    string(defaultValue: '', description:'Please enter app: [org/space/service]', name: 'cf_src')
    string(defaultValue: '', description:'Please enter app: [org/space/service]', name: 'cf_dest')
    string(defaultValue: 'eu-west-1', description:'Please enter region: ', name: 'cf_region')
  }

  stages {

    stage('Init') {
      steps {
        script {
          ansiColor('xterm') {
            validateDeclarativePipeline("${env.WORKSPACE}/Jenkinsfile")
            deployer = docker.image("quay.io/uktrade/deployer:${env.GIT_BRANCH.split("/")[1]}")
            deployer.pull()
          }
        }
      }
    }

    stage('Task') {
      steps {
        script {
          ansiColor('xterm') {
            deployer.inside {
              withCredentials([string(credentialsId: env.GDS_PAAS_CONFIG, variable: 'paas_config_raw')]) {
                paas_config = readJSON text: paas_config_raw
              }
              if (!params.cf_region) {
                cf_region = paas_config.default
              }
              paas_region = paas_config.regions."${cf_region}"
              echo "\u001B[32mINFO: Setting PaaS region to ${paas_region.name}.\u001B[m"

              withCredentials([usernamePassword(credentialsId: paas_region.credential, passwordVariable: 'gds_pass', usernameVariable: 'gds_user')]) {
                sh """
                  cf api ${paas_region.api}
                  cf auth ${gds_user} ${gds_pass}
                """
              }

              src = params.cf_src.split("/")
              dest = params.cf_dest.split("/")
              echo "\u001B[32mINFO: Purge destination database ${dest[2]}.\u001B[m"
              sh "cf target -o ${dest[0]} -s ${dest[1]}"
              sh """
                cf conduit -p $((1024 + RANDOM % 54511)) ${dest[2]} -- sql < purge.sql
              """
              echo "\u001B[32mINFO: Restoring database from src[2] to ${dest[2]}.\u001B[m"
              sh "cf target -o ${src[0]} -s ${src[1]}"
              sh """
                cf conduit -p $((1024 + RANDOM % 54511)) --no-interactive src[2] -- pg_dump -O -x --disable-triggers | sed -e '/PostgreSQL database dump complete/s/$/\n\\\q/' | cf conduit -p $((1024 + RANDOM % 54511)) --org ${dest[0]} --space ${dest[1]} --no-interactive dest[2] -- psql
              """
            }
          }
        }
      }
    }

  }
}