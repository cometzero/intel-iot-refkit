#!groovy
//
// Jenkins Pipeline script to produce builds and run tests
//
// Copyright (c) 2016, Intel Corporation.
//
// This program is free software; you can redistribute it and/or modify it
// under the terms and conditions of the GNU General Public License,
// version 2, as published by the Free Software Foundation.
//
// This program is distributed in the hope it will be useful, but WITHOUT
// ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or
// FITNESS FOR A PARTICULAR PURPOSE.  See the GNU General Public License for
// more details.
//

def is_pr = env.JOB_NAME.endsWith("_pull-requests")
def target_machine = "intel-corei7-64"

// JOB_NAME expected to be in form <layer>_<branch>
def current_project = "${env.JOB_NAME}".tokenize("_")[0]
def image_name = "${current_project}_build:${env.BUILD_TAG}"
def ci_build_id = "${env.BUILD_TIMESTAMP}-build-${env.BUILD_NUMBER}"
def test_runs = [:]
def testinfo_data = ""
def ci_git_commit = ""
def global_sum_log = ""
def added_commits = ""
def builder_node = ""
def builder_wspace = ""
// reasonable value: keep few recent, dont take risk to fill disk
int num_build_dirs_to_keep = 4

// Define global environment common for all docker sessions
def script_env = """
    export WORKSPACE=\$PWD
    export HOME=\$JENKINS_HOME
    export CURRENT_PROJECT=${current_project}
    export BUILD_CACHE_DIR=${env.PUBLISH_DIR}/bb-cache
    export GIT_PROXY_COMMAND=oe-git-proxy
    export CI_BUILD_ID=${ci_build_id}
    export GIT_COMMITTER_NAME="IOT Refkit CI"
    export GIT_COMMITTER_EMAIL='refkit-ci@yoctoproject.org'
    export TARGET_MACHINE=${target_machine}
"""

try {
    timestamps {
        node('rk-docker') {
            ws("workspace/builder-slot-${env.EXECUTOR_NUMBER}") {
                // remember node and workspace needed by workspace cleaner
                builder_node = "${env.NODE_NAME}"
                builder_wspace = "${env.WORKSPACE}"
                set_gh_status_pending(is_pr, 'Prepare for build')
                stage('Cleanup workspace') {
                    deleteDir()
                }
                stage('Checkout content') {
                    checkout_content(is_pr)
                }
                stage('Build docker image') {
                    build_docker_image(image_name)
                }
                def docker_image = docker.image(image_name)
                run_args = ["--device=/dev/kvm -v ${env.PUBLISH_DIR}:${env.PUBLISH_DIR}:rw",
                            run_proxy_args()].join(" ")
                docker_image.inside(run_args) {
                    set_gh_status_pending(is_pr, 'Pre-build tests')
                    params = ["${script_env}", "docker/pre-build.sh"].join("\n")
                    stage('Pre-build tests') {
                        sh "${params}"
                    }
                    try {
                        set_gh_status_pending(is_pr, 'Building')
                        params = ["${script_env}", "docker/build-project.sh"].join("\n")
                        stage('Build') {
                            sh "${params}"
                        }
                    } catch (Exception e) {
                        throw e
                    } finally {
                        set_gh_status_pending(is_pr, 'Store images')
                        params = ["${script_env}", "docker/publish-project.sh"].join("\n")
                        stage('Store images') {
                            sh "${params}"
                        }
                    }
                    set_gh_status_pending(is_pr, 'Post-build tests')
                    params = ["${script_env}", "docker/post-build.sh"].join("\n")
                    stage('Post-build tests') {
                        sh "${params}"
                    }
                } // docker_image
                tester_script = readFile "docker/tester-exec.sh"
                qemu_script = readFile "docker/run-qemu.exp"
                testinfo_data = readFile "${target_machine}.testinfo.csv"
                if ( !is_pr ) {
                    ci_git_commit = readFile("ci_git_commit").trim()
                    // This command expects that each new master build is based on a github merge
                    sh "git rev-list HEAD^...HEAD --oneline --no-merges | sed 's/[^ ]* /    /' > added_commits"
                    added_commits = readFile("added_commits")
                }
            } // ws
            // rotate workspace out of way, rename using CI_BUILD_ID.
            // Older trees are deleted later in test_runs[], without keeping build waiting.
            ws("workspace") {
                sh "mv ${builder_wspace} ${builder_wspace}.ci-prev.${ci_build_id}"
            }
        } // node
    } // timestamps

    // insert older trees deletion job into tester jobs array
    test_runs["clean_older_workspaces"] = {
        node(builder_node) {
            ws("workspace") {
                trim_build_dirs(num_build_dirs_to_keep)
            }
        }
    }

    test_targets = testinfo_data.split("\n")
    for(int i = 0; i < test_targets.size() && test_targets[i] != ""; i++) {
        def one_target_testinfo = test_targets[i]
        def test_device = one_target_testinfo.split(',')[4]
        def test_machine = one_target_testinfo.split(',')[3]
        def img = one_target_testinfo.split(",")[0]
        test_runs["test_${i}_${test_device}"] = {
            node('refkit-tester') {
                deleteDir() // clean workspace
                echo "Testing test_${test_device} with image_info: ${one_target_testinfo}"
                writeFile file: 'tester-exec.sh', text: tester_script
                writeFile file: 'run-qemu.exp', text: qemu_script
                // append newline so that tester-exec.sh can parse it using "read"
                one_target_testinfo += "\n"
                // create testinfo.csv on this tester describing one image
                writeFile file: "testinfo.csv", text: one_target_testinfo
                try {
                    withEnv(["CI_BUILD_ID=${ci_build_id}",
                        "MACHINE=${test_machine}",
                        "TEST_DEVICE=${test_device}" ]) {
                            sh 'chmod a+x tester-exec.sh run-qemu.exp && ./tester-exec.sh'
                    }
                } catch (Exception e) {
                    throw e
                } finally {
                    // read tests summary prepared by tester-exec.sh
                    // Here one tester adds it's summary piece to the global buffer.
                    global_sum_log += readFile "results-summary-${test_device}.${img}.log"
                    archiveArtifacts allowEmptyArchive: true,
                                     artifacts: '*.log, *.xml'
                }
                // without locking we may lose tester result set(s)
                // if testers run xunit step in nearly same time
                lock(resource: "step-xunit") {
                    step_xunit()
                }
            } // node
        } // test_runs =
    } // for i
    stage('Parallel test run') {
        set_gh_status_pending(is_pr, 'Testing')
        timestamps {
            try {
                parallel test_runs
            } catch (Exception e) {
                currentBuild.result = 'UNSTABLE'
            }
        }
    }
} catch (Exception e) {
    echo "Error: ${e}"
    if (currentBuild.result == null) {
        // Set currentBuild.result as FAILURE if there is an error in building
        currentBuild.result = 'FAILURE'
    }
    throw e
} finally {
    //  If tests stage was skipped because of no tests, then currentBuild.result
    //  remains null until end so manually set it as SUCCESS
    if (currentBuild.result == null) {
        currentBuild.result = 'SUCCESS'
    }
    echo "Finally: build result is ${currentBuild.result}"
    if (is_pr) {
        if (currentBuild.result == 'UNSTABLE') {
            setGitHubPullRequestStatus state: 'FAILURE', context: "${env.JOB_NAME}", message: "Build result: ${currentBuild.result}"
        } else {
            setGitHubPullRequestStatus state: "${currentBuild.result}", context: "${env.JOB_NAME}", message: "Build result: ${currentBuild.result}"
        }
    } else {
        // send summary email after non-PR build
        email = "Git commit hash: ${ci_git_commit} \n\n"
        email += "Added commits:\n\n${added_commits}\n"
        email += "Test results:\n\n${global_sum_log}"
        def subject = "${currentBuild.result}: Job ${env.JOB_NAME} [${env.BUILD_NUMBER}]"
        echo "${email}"
        node('rk-mailer') {
            writeFile file: 'msg.txt', text: email
            sh "cat msg.txt |mailx -s '${subject}' ${env.RK_NOTIFICATION_MAIL_RECIPIENTS}"
        }
    }
}

echo "End of pipeline, build result is ${currentBuild.result}"

// Support functions:
def build_proxy_args() {
    return ["--build-arg http_proxy=${env.http_proxy}",
            "--build-arg https_proxy=${env.https_proxy}",
            "--build-arg ALL_PROXY=${env.ALL_PROXY}"].join(" ")
}

def run_proxy_args() {
    return [ "-e http_proxy=${env.http_proxy}",
             "-e https_proxy=${env.https_proxy}",
             "-e ALL_PROXY=${env.ALL_PROXY}",
             "-e no_proxy=${env.NO_PROXY}"].join(" ")
}

def build_user_args() {
    dir(pwd([tmp:true])+"/.build_user_args") {
         // get jenkins user uid/gid
        sh "id -u > jenkins_uid"
        jenkins_uid = readFile("jenkins_uid").trim()
        sh "id -g > jenkins_gid"
        jenkins_gid = readFile("jenkins_gid").trim()
        deleteDir()
    }
    return "--build-arg uid=${jenkins_uid} --build-arg gid=${jenkins_gid}"
}

def checkout_content(is_pr) {
    if (is_pr) {
        // we are building pull request
        echo "Checkout: PR case"
        checkout([$class: 'GitSCM',
            branches: [
                [name: "origin-pull/$GITHUB_PR_NUMBER/$GITHUB_PR_COND_REF"]
            ],
            doGenerateSubmoduleConfigurations: false,
            extensions: [
                [$class: 'SubmoduleOption',
                    disableSubmodules: false,
                    recursiveSubmodules: true,
                    reference: "${env.PUBLISH_DIR}/bb-cache/.git-mirror",
                    trackingSubmodules: false],
                [$class: 'CleanBeforeCheckout']
            ],
            submoduleCfg: [],
            userRemoteConfigs: [
                [credentialsId: "${GITHUB_AUTH}",
                    name: 'origin-pull',
                    refspec: "+refs/pull/$GITHUB_PR_NUMBER/*:refs/remotes/origin-pull/$GITHUB_PR_NUMBER/*",
                    url: "${GITHUB_PROJECT}"]
            ]
        ])
    } else {
        echo "Checkout: MASTER case"
        checkout poll: false, scm: scm
    }
}

def build_docker_image(image_name) {
    // Base container OS to use, see docker configs in docker/
    def build_os = "opensuse-42.2"
    def build_args = [ build_proxy_args(), build_user_args()].join(" ")
    sh "docker build -t ${image_name} ${build_args} docker/${build_os}"
    dockerFingerprintFrom dockerfile: "docker/${build_os}/Dockerfile", image: "${image_name}"
}

def set_gh_status_pending(is_pr, _msg) {
    if (is_pr) {
        setGitHubPullRequestStatus state: 'PENDING', context: "${env.JOB_NAME}", message: "${_msg}"
    }
}

def step_xunit() {
    step([$class: 'XUnitPublisher',
    testTimeMargin: '3000',
    thresholdMode: 1,
    thresholds: [
        [$class: 'FailedThreshold',
            failureNewThreshold: '0',
            failureThreshold: '0',
            unstableNewThreshold: '99999',
            unstableThreshold: '99999'],
        [$class: 'SkippedThreshold',
            failureNewThreshold: '99999',
            failureThreshold: '99999',
            unstableNewThreshold: '99999',
            unstableThreshold: '99999']],
    tools: [
        [$class: 'JUnitType',
            deleteOutputFiles: true,
            failIfNotNew: true,
            pattern: 'TEST-*.xml',
            skipNoTestFiles: false,
            stopProcessingIfError: true]]])
}

// Delete older builder trees.
// While majority/regular workspaces are named builder-slot-0, (for EXECUTOR=0),
// Jenkins may create additional trees as builder-slot-0_X.
// Wildcard covers all such workspaces, dont want some pattern filling disk independently.
def trim_build_dirs(num_to_keep) {
    sh """
dirs=`find ${env.WORKSPACE} -mindepth 1 -maxdepth 1 -type d -name "builder-slot-*.ci-prev.*" |sort -n |head -n -${num_to_keep} |tr '\n' ' '`
if [ -n "\${dirs}" ]; then
    ionice -c 3 rm -fr \$dirs
fi
"""
}
