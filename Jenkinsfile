@Library('gnt-cd-library@master') _

def cloudValue = "kubernetes"
def podLabel = "${BUILD_NUMBER}".replace("/", "-").replace(" ", "-")
def podDefinition = """
apiVersion: v1
kind: Pod
spec:
  nodeSelector:
    env: "dev"
  containers:
  - name: gntcd-images
    image: '265143165931.dkr.ecr.eu-west-1.amazonaws.com/sre/cd-images:gntcd'
    imagePullPolicy: Always
    tty: true
    resources:
      limits:
        cpu: 1500m
        memory: 2.5Gi
      requests:
        cpu: 750m
        memory: 1Gi
"""


    def scmConfig = [
        branch : 'master',
        gitCredentialsId : 'git-credential',
        gitURL : 'https://git.{url}.com'
    ]

    def sshConfig = [
        credentialsId : 'ssh-credential'
    ]

    def nexusConfig = [
        credentialsId : 'credential'
    ]

    def rollback = false

def initialParametersInput = input(
    id: 'CRISJAVACDInitialInputs', 
    message: 'Jenkins Pipeline to Deploy to DEV/SIT/STAGE environment(s)', 
    ok: 'Confirm',
    parameters: [
        stringParam(name: 'ApplicationVersion', description: 'Application version', defaultValue: '1.0'),
        booleanParam(name: 'DEV', description: 'Deploy to the DEV Environment?', defaultValue: true),
        booleanParam(name: 'SIT', description: 'Deploy to the SIT Environment?', defaultValue: false),
        booleanParam(name: 'STG', description: 'Deploy to the STG Environment?', defaultValue: false),
        booleanParam(name: 'Verbose', description: 'Additional details in logs', defaultValue: false)
    ]
)

def genericDeployScript = """
    echo "Deployment script"
"""

DEV_ENV = [
    deployScripts: [genericDeployScript],
    envType: "dev"
]

SIT_ENV = [
    deployScripts: [genericDeployScript],
    envType: "sit"
]

STG_ENV = [
    deployScripts: [genericDeployScript],
    envType: "stg"
]

/*
* Run a command on the target host
* - commands: script to run
* - envType: environment type (dev/sit/stage/prod)
* - sshCredentialsId: id of Jenkins credentials used for ssh
* - nexusCredentialsId: id of Jenkins credentials used for Nexus
* - tag: The operation to be performed - deploy/rollback/upload
* - verbose: print detailed logs
*/
def runOnTargetHost(commands,
                    applicationVersion,
                    envType,
                    sshCredentialsId,
                    nexusCredentialsId,
                    tag,
                    verbose) {
    withCredentials(
            [
                [
                $class: 'UsernamePasswordMultiBinding', 
                credentialsId: nexusCredentialsId,
                usernameVariable: 'nexusUsername', passwordVariable: 'nexusPassword'
                ]
            ]
    ) 
    {
        withCredentials(
            bindings:
                [
                    sshUserPrivateKey(
                        credentialsId: sshCredentialsId,
                        usernameVariable: 'svcUsername', 
                        keyFileVariable: 'svcPrivateKeyFile',
                        passphraseVariable: 'svcPassphrase'
                    )
                ] 
        ) 
        {

            timestamps {
                    sh """
                        cd CD/ansible

                        echo "Running ansible-playbook"
                        ansible-playbook \
                            -i inventories/${envType}/hosts \
                            pipeline.yaml \
                            -u ${svcUsername} \
                            --private-key ${svcPrivateKeyFile} \
                            -e nexus_username=${nexusUsername} \
                            -e nexus_password=${nexusPassword} \
                            -e application_version=${applicationVersion} \
                            -e env_type=${envType} \
                            --tags ${tag}
                    """
            }
        } //withCredentials
    } //withCredentials
}

pipeline {
// For Jenkinsfile validation purposes. Should be removed in the future.
// agent any
agent {
    kubernetes {
                cloud cloudValue
                label podLabel
                defaultContainer 'gntcd-images'
                yaml podDefinition
            }
} // agent

environment {
    TEAMS_HOOK_NAME = "cicd"
    TEAMS_HOOK_URL = ""
} // environment

stages {

    stage('Git Checkout') {
    steps('Checkout') {
        script {
        checkout(
            [
            $class: 'GitSCM', 
            branches: [[ 
                name: scmConfig.branch 
            ]], 
            doGenerateSubmoduleConfigurations: false, 
            extensions: [], 
            submoduleCfg: [], 
            userRemoteConfigs: [[
                credentialsId: scmConfig.gitCredentialsId, 
                url: scmConfig.gitURL
            ]]
            ]
        )
        } //script
    } //steps
    } // stage

    stage("Dependencies") {
        steps {
            script {
            // For Linux packages install. Should be moved to docker image and removed in the future .
            // Debian packages install
                timestamps {
                    sh """
                        mkdir -p ~/.ssh
                        printf "Host *\n\tStrictHostKeyChecking no" > ~/.ssh/config
                        printf "[defaults]\nallow_world_readable_tmpfiles=True" >> ~/.ansible.cfg
                        printf "\n[privilege_escalation]\nbecome_exe = '~/asuser sudo'" >> ~/.ansible.cfg
                    """
                } // timestamp 
            } //script
        } // steps
    } // stage

    stage("DEV Deploy") {

        when { 
            expression { 
            (initialParametersInput.DEV == true) && (rollback == false) }
        }

        steps {
            script {
                for (script in DEV_ENV.deployScripts) {
                    runOnTargetHost( 
                        script,    // commands
                        initialParametersInput.ApplicationVersion, //applicationVersion
                        DEV_ENV.envType, //envType
                        sshConfig.credentialsId, // sshCredentialsId
                        nexusConfig.credentialsId, // nexusCredentialsId
                        'deploy', //tag
                        initialParametersInput.Verbose // verbose
                    )
                }
            } //script
        } // steps
    } // stage


    stage("DEV Sanity Check") {

        when { 
            expression { 
            (initialParametersInput.DEV == true) && (rollback == false) }
        }

        steps {
            script {
                try {
                    input( message: """
    The deployment to DEV is complete.

    Press :
    - Proceed to continue (SIT Deploy)
    - Abort to Rollback (to last successful DEV deployment)""" )
                } 
                catch(e) {
                    rollback = true
                }
                
                if ( rollback == false ) {
                    echo "DEV Sanity Check passed"
                    for (script in DEV_ENV.deployScripts) {
                    // Upload to sit folder in the Nexus repository
                        runOnTargetHost( 
                            script,    // commands
                            initialParametersInput.ApplicationVersion, //applicationVersion
                            DEV_ENV.envType, //envType
                            sshConfig.credentialsId, // sshCredentialsId
                            nexusConfig.credentialsId, // nexusCredentialsId
                            'upload', //tag
                            initialParametersInput.Verbose // verbose
                        )
                    }
                }
                else {
                    echo "DEV Sanity Check failed"
                    echo "Rolling back DEV deployment"
                    for (script in DEV_ENV.deployScripts) {
                        runOnTargetHost( 
                            script,    // commands
                            initialParametersInput.ApplicationVersion, //applicationVersion
                            DEV_ENV.envType, //envType
                            sshConfig.credentialsId, // sshCredentialsId
                            nexusConfig.credentialsId, // nexusCredentialsId
                            'rollback', //tag
                            initialParametersInput.Verbose // verbose
                        )
                    }
                }
            } //script
        } // steps
    } // stage

    stage("SIT Deploy") {

        when { 
            expression { 
            (initialParametersInput.SIT == true) && (rollback == false) }
        }

        steps {
            script {
                for (script in SIT_ENV.deployScripts) {
                    runOnTargetHost( 
                        script,    // commands
                        initialParametersInput.ApplicationVersion, //applicationVersion
                        SIT_ENV.envType, //envType
                        sshConfig.credentialsId, // sshCredentialsId
                        nexusConfig.credentialsId, // nexusCredentialsId
                        'deploy', //tag
                        initialParametersInput.Verbose // verbose
                    )
                }
            } //script
        } // steps
    } // stage

    stage("SIT Automated Tests") {

        when { 
            expression { 
            (initialParametersInput.SIT == true) && (rollback == false) }
        }

        steps {
            script {
            echo "Placeholder for automated tests"
            }
        }
    }

    stage("SIT Sanity Check") {

        when { 
            expression { 
            (initialParametersInput.SIT == true) && (rollback == false) }
        }

        steps {
            script {
                try {
                    input( message: """
    The deployment to SIT is complete.

    Press :
    - Proceed to continue (STAGE Deploy)
    - Abort to Rollback (to last successful SIT deployment)""" )
                } 
                catch(e) {
                    rollback = true
                }
                
                if ( rollback == false ) {
                    echo "SIT Sanity Check passed"
                    for (script in SIT_ENV.deployScripts) {
                    // Upload to sit folder in the Nexus repository
                    runOnTargetHost( 
                        script,    // commands
                        initialParametersInput.ApplicationVersion, //applicationVersion
                        SIT_ENV.envType, //envType
                        sshConfig.credentialsId, // sshCredentialsId
                        nexusConfig.credentialsId, // nexusCredentialsId
                        'upload', //tag
                        initialParametersInput.Verbose // verbose
                    )
                    }
                }
                else {
                    echo "SIT Sanity Check failed"
                    echo "Rolling back SIT deployment"
                    for (script in SIT_ENV.deployScripts) {
                        runOnTargetHost( 
                            script,    // commands
                            initialParametersInput.ApplicationVersion, //applicationVersion
                            SIT_ENV.envType, //envType
                            sshConfig.credentialsId, // sshCredentialsId
                            nexusConfig.credentialsId, // nexusCredentialsId
                            'rollback', //tag
                            initialParametersInput.Verbose // verbose
                        )
                    }
                }
            } //script
        } // steps
    } // stage

    stage("STG Deploy") {

        when { 
            expression { 
            (initialParametersInput.STG == true) && (rollback == false) }
        }

        steps {
            script {
                for (script in STG_ENV.deployScripts) {
                    runOnTargetHost( 
                        script,    // commands
                        initialParametersInput.ApplicationVersion, //applicationVersion
                        STG_ENV.envType, //envType
                        sshConfig.credentialsId, // sshCredentialsId
                        nexusConfig.credentialsId, // nexusCredentialsId
                        'deploy', //tag
                        initialParametersInput.Verbose // verbose
                    )
                }
            } //script
        } // steps
    } // stage

    stage("STG Automated Tests") {

        when { 
            expression { 
            (initialParametersInput.STG == true) && (rollback == false) }
        }

        steps {
            script {
            echo "Placeholder for automated tests"
            }
        }
    }

    stage("STG Sanity Check") {

        when { 
            expression { 
            (initialParametersInput.STG == true) && (rollback == false) }
        }

        steps {
            script {
                try {
                    input( message: """
    The deployment to STG is complete.

    Press :
    - Proceed to continue (STG Deploy)
    - Abort to Rollback (to last successful STG deployment)""" )
                } 
                catch(e) {
                    rollback = true
                }
                
                if ( rollback == false ) {
                    echo "STG Sanity Check passed"
                    for (script in STG_ENV.deployScripts) {
                    // Upload to stg folder in the Nexus repository
                    runOnTargetHost( 
                        script,    // commands
                        initialParametersInput.ApplicationVersion, //applicationVersion
                        STG_ENV.envType, //envType
                        sshConfig.credentialsId, // sshCredentialsId
                        nexusConfig.credentialsId, // nexusCredentialsId
                        'upload', //tag
                        initialParametersInput.Verbose // verbose
                    )
                    }
                }
                else {
                    echo "STG Sanity Check failed"
                    echo "Rolling back STG deployment"
                    for (script in STG_ENV.deployScripts) {
                        runOnTargetHost( 
                            script,    // commands
                            initialParametersInput.ApplicationVersion, //applicationVersion
                            STG_ENV.envType, //envType
                            sshConfig.credentialsId, // sshCredentialsId
                            nexusConfig.credentialsId, // nexusCredentialsId
                            'rollback', //tag
                            initialParametersInput.Verbose // verbose
                        )
                    }
                }
            } //script
        } // steps
    } // stage
 } // stages
} // pipeline