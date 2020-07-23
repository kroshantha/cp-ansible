#!/usr/bin/env groovy

import static groovy.json.JsonOutput.*

/* These are variables that can be used to test an un-released version of the Confluent Platform that resides at
 * a different HTTPS Endpoint other than `https://packages.confluent.io`. You do not need to specify *any* of them
 * for normal testing purposes, and are purely here for Confluent Inc's usage only. 
 */

// The version to install, set to the "next" version to test the "next" version.
def confluent_package_version = string(name: 'CONFLUENT_PACKAGE_VERSION',
    defaultValue: '',
    description: 'Confluent Version to install and test (ie: 5.4.1)'
)

// The HTTP(S) endpoint from which to obtain the platform packages
def confluent_common_repository_baseurl = string(name: 'CONFLUENT_PACKAGE_BASEURL',
    defaultValue: '',
    description: 'Packaging Base URL from where to download packages (ie: https://packages.confluent.io)'
)

// Confluent's nightly packages use a different version scheme, so this parameter controls the "suffix" value
// of the packages that are installed.
def confluent_release_quality = choice(name: 'CONFLUENT_RELEASE_QUALITY',
    choices: ['prod', 'snapshot'],
    defaultValue: 'prod',
    description: 'Determines the release extention (suffix) (ie: "prod" for public releases, "snapshot" for nightly builds)',
)

// Parameter for the molecule test scenario to run
def molecule_scenario_name = choice(name: 'SCENARIO_NAME',
    choices: ['plaintext-rhel'],
    defaultValue: 'plaintext-rhel',
    description: 'The Ansible Molecule scenario name to run',
)

def config = jobConfig {
    nodeLabel = 'docker-oraclejdk8'
    slackChannel = '#ansible-eng'
    timeoutHours = 4
    runMergeCheck = false
    properties = [parameters([confluent_package_version, confluent_common_repository_baseurl, confluent_release_quality, molecule_scenario_name])]
}

def job = {
    /*stage('Install Molecule and Latest Ansible') {
        sh '''
            sudo pip install --upgrade 'ansible==2.9.*'
            sudo pip install molecule docker
        '''
    }*/

    def override_config = [:]

    if(params.CONFLUENT_PACKAGE_BASEURL) {
        override_config['confluent_common_repository_baseurl'] = params.CONFLUENT_PACKAGE_BASEURL
    } else if (targetBranch().toString().endsWith('.x')) {
        /* This condition imples we're in a dev (.x) branch and therefore the release in confluent_package_version 
           does not yet exist on https://packages.confluent.io so we have to query the packaging job for the last
           successful build location (what utilities.getLastNightlyPackagingBaseURL returns). We also override the
           confluent_package_*_suffix to an empty string so it will install the (expected) latest version */
        override_config['confluent_common_repository_baseurl'] = utilities.getLastNightlyPackagingBaseURL(targetBranch().toString())
        override_config['confluent_package_redhat_suffix'] = ""
        override_config['confluent_package_debian_suffix'] = ""
    }

    if(params.CONFLUENT_PACKAGE_VERSION) {
        override_config['confluent_package_version'] = params.CONFLUENT_PACKAGE_VERSION
        override_config['confluent_repo_version'] = params.CONFLUENT_PACKAGE_VERSION.tokenize('.')[0..1].join('.')

        if(confluent_release_quality != 'prod') {
            // 'prod' case doesn't need anything overriden
            switch(params.CONFLUENT_RELEASE_QUALITY) {
                case "snapshot":
                    override_config['confluent_package_redhat_suffix'] = "-${params.CONFLUENT_PACKAGE_VERSION}-0.1.SNAPSHOT"
                    override_config['confluent_package_debian_suffix'] = "=${params.CONFLUENT_PACKAGE_VERSION}~SNAPSHOT-1"

                    // Disable reporting for nightly builds
                    config.testbreakReporting = false
                    config.slackChannel = null
                break
                default:
                    error("Unknown release quality ${params.CONFLUENT_RELEASE_QUALITY}")
                break
            }
        }
    }

    def molecule_args = ""
    if(override_config) {
        override_config['bootstrap'] = false
        def base_config = [
            'provisioner': [
                'inventory': [
                    'group_vars': [
                        'all': override_config
                    ]
                ]
            ]
        ]
        echo "Overriding Ansible vars for testing with base-config:\n" + prettyPrint(toJson(override_config))

        writeYaml file: "roles/confluent.test/base-config.yml", data: base_config

        molecule_args = "--base-config base-config.yml"
    }

    /*withDockerServer([uri: dockerHost()]) {
        stage("Test Scenario: ${params.SCENARIO_NAME}") {
            sh """
cd roles/confluent.test
molecule ${molecule_args} test -s ${params.SCENARIO_NAME}
            """
        }
    }*/
}

def post = {
    withDockerServer([uri: dockerHost()]) {
        stage("Destroy Scenario: ${params.SCENARIO_NAME}") {
            sh """
cd roles/confluent.test
molecule destroy -s ${params.SCENARIO_NAME} || true
"""
        }
    }
}

runJob config, job, post
