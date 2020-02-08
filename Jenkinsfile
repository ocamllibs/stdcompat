properties([
  [
    $class: 'ThrottleJobProperty',
    categories: ['category'],
    limitOneJobWithMatchingParams: false,
    maxConcurrentPerNode: 4,
    maxConcurrentTotal: 0,
    paramsToUseForLimit: '',
    throttleEnabled: true,
    throttleOption: 'category'
  ],
])

pipeline {
    agent none

    stages {
        stage('Prepare') {
            agent {
                label 'linux'
            }
            steps {
                sh 'docker build -t stdcompat .'
            }
        }
        stage('Bootstrap') {
            agent {
                label 'linux'
            }
            steps {
                sh 'docker run --rm --volume $PWD:/workspace stdcompat bash -c \'cd /workspace && make -f Makefile.bootstrap\''
                stash name: 'bootstrap'
            }
        }
        stage('Configure') {
            agent {
                label 'linux'
            }
            steps {
                unstash 'bootstrap'
                sh 'docker run --rm --volume $PWD:/workspace stdcompat bash -c \'cd /workspace && mkdir build && cd build && ../configure\''
                stash name: 'configure'
            }
        }
        stage('Build') {
            agent {
                label 'linux'
            }
            steps {
                unstash 'configure'
                sh 'docker run --rm --volume $PWD:/workspace stdcompat bash -c \'cd /workspace/build && make\''
            }
        }
        stage('Test with magic') {
            agent {
                label 'linux'
            }
            steps {
                script {
                    def switches = sh (
                        script: 'docker run --rm stdcompat opam switch -s',
                        returnStdout: true
                    ).split('\n')
                    def branches = [:]
                    for (i in switches) {
                        def switch_name = i
                        branches[switch_name] = {
                            node('linux') {
                                sh "rm -rf build"
                                unstash 'bootstrap'
                                sh "docker run --rm --volume \$PWD:/workspace stdcompat bash -c 'cd /workspace && opam config exec --switch $switch_name -- sh -c '\\''mkdir build && cd build && ../../configure && make && make tests && ../configure && make && make tests'\\'"
                            }
                        }
                    }
                    throttle(['category']) {
                        parallel branches
                    }
                }
            }
        }
        stage('Test without magic') {
            agent {
                label 'linux'
            }
            steps {
                script {
                    def switches = sh (
                        script: 'docker run --rm stdcompat opam switch -s',
                        returnStdout: true
                    ).split('\n')
                    def branches = [:]
                    for (i in switches) {
                        def switch_name = i
                        branches[switch_name] = {
                            node('linux') {
                                sh "rm -rf build"
                                unstash 'bootstrap'
                                sh "docker run --rm --volume \$PWD:/workspace stdcompat bash -c 'cd /workspace && opam config exec --switch $switch_name -- sh -c '\\''mkdir build && cd build && ../../configure && make && make tests && ../configure --disable-magic && make && make tests'\\'"
                            }
                        }
                    }
                    throttle(['category']) {
                        parallel branches
                    }
                }
            }
        }
        stage('Test windows') {
            agent {
                label 'windows'
            }
            steps {
                script {
                    def versions = ["4.08.1", "4.09.0", "4.10.0+rc1"]
                    def branches = [:]
                    for (i in versions) {
                        def version = i
                        branches[version] = {
                            node('windows') {
                                checkout scm
                                bat """
call "C:\\Program Files (x86)\\Microsoft Visual Studio\\2017\\Community\\VC\\Auxiliary\\Build\\vcvars64.bat"
set PATH=C:\\tools\\cygwin\\bin;%PATH%
bash -c "eval \$(~/ocaml-4.10.0+rc1/tools/msvs-promote-path) && ci/cygwin-compile-ocaml.sh "$version" && export PATH="/cygdrive/c/ocaml/$version:$PATH" && make -f Makefile.bootstrap && ./configure && make && make tests"
                                """
                            }
                        }
                    }
                    throttle(['category']) {
                        parallel branches
                    }
                }
            }
        }
        stage('Deploy') {
            agent {
                label 'linux'
            }
            steps {
                unstash 'configure'
                sh 'cd build && make dist'
                archiveArtifacts artifacts: 'build/*.tar.gz', fingerprint: true
            }
        }
    }
}
