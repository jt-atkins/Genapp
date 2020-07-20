// Agent labels
def zOsAgentLabel = env.ZOS_AGENT_LABEL ? env.ZOS_AGENT_LABEL : 'e2e-pipeline'
def linuxAgent = 'master'

// GIT repositories
def srcGitRepo =  null
def srcGitBranch = null
def zAppBuildGitRepo = 'https://github.com/IBM/dbb-zappbuild.git'
def zAppBuildGitBranch = 'development' // Some important issues are not yet merged into master.
def dbbGitRepo = 'https://github.com/IBM/dbb'
def dbbGitBranch = 'master'

// DBB
def dbbHome=null
def dbbUrl=null
def dbbHlq=null
def dbbBuildType='-i'
def dbbGroovyzOpts= ''

// Artifactory
def artiUrl = null
def repositoryPath = null
def artiCredentialsId = 'artifactory_id'


// UCD
def ucdBuztool = null
def ucdApplication = 'GenApp-Deploy'
def ucdProcess = 'Deploy'
def ucdComponent = 'GenAppComponent'
def ucdEnv = 'Development'
def ucdSite = 'UrbanCodeE2EPipeline'

// Verbose
def verbose = env.VERBOSE && env.VERBOSE == 'true' ? true : false

// Private
def needDeploy = true
def buildVerbose = verbose ? '-v' : ''

pipeline {

    agent { label linuxAgent }

    options { skipDefaultCheckout(true) }

    stages {
        
        stage('Git Clone/Refresh') {
            agent { label zOsAgentLabel }
            steps {
                script {
                    if ( verbose ) {
                        echo sh(script: 'env|sort', returnStdout: true)
                    }
                    
                    dir('cics-genapp') {
                        sh(script: 'rm -f .git/info/sparse-checkout', returnStdout: true)
                        catchError(buildResult: 'SUCCESS', stageResult: 'SUCCESS'){
                            srcGitRepo = scm.getUserRemoteConfigs()[0].getUrl()
                            srcGitBranch = scm.branches[0].name
                        }
                        def scmVars = null
                        scmVars = checkout([$class: 'GitSCM', branches: [[name: srcGitBranch]],
                                                doGenerateSubmoduleConfigurations: false,
                                                submoduleCfg: [],
                                                userRemoteConfigs: [[
                                                                     // For now GenApp is not public
                                                                     credentialsId: 'github_id',
                                                                     url: srcGitRepo,
                                                                     ]]])
                    }
                    
                    dir("dbb-zappbuild") {
                        sh(script: 'rm -f .git/info/sparse-checkout', returnStdout: true)
                        def scmVars =
                            checkout([$class: 'GitSCM', branches: [[name: zAppBuildGitBranch]],
                              doGenerateSubmoduleConfigurations: false,
                              submoduleCfg: [],
                            userRemoteConfigs: [[
                                url: zAppBuildGitRepo,
                            ]]])
                    }
                    
                    dir("dbb") {
                        sh(script: 'rm -f .git/info/sparse-checkout', returnStdout: true)
                        def scmVars =
                            checkout([$class: 'GitSCM', branches: [[name: dbbGitBranch]],
                              doGenerateSubmoduleConfigurations: false,
                              extensions: [
                                       [$class: 'SparseCheckoutPaths',  sparseCheckoutPaths:[[$class:'SparseCheckoutPath', path:'Pipeline/CreateUCDComponentVersion/']]]
                                    ],
                              submoduleCfg: [],
                            userRemoteConfigs: [[
                                url: dbbGitRepo,
                            ]]])
                    }
                }
            }
        }

        stage('DBB Build') {
            steps {
                script{
                    node( zOsAgentLabel ) {
                        if ( env.DBB_BUILD_EXTRA_OPTS != null ) {
                           dbbBuildExtraOpts = env.DBB_BUILD_EXTRA_OPTS
                        }
                        if ( env.DBB_BUILD_TYPE != null ) {
                            dbbBuildType = env.DBB_BUILD_TYPE
                        }
                        dbbHome = env.DBB_HOME
                        dbbUrl = env.DBB_URL
                        dbbHlq = env.DBB_HLQ
                        sh "$dbbHome/bin/groovyz $dbbGroovyzOpts ${WORKSPACE}/dbb-zappbuild/build.groovy --logEncoding UTF-8 -w ${WORKSPACE} --application cics-genapp --sourceDir ${WORKSPACE}  --workDir ${WORKSPACE}/BUILD-${BUILD_NUMBER} --hlq ${dbbHlq}.GENAPP --url $dbbUrl -pw ADMIN $dbbBuildType $buildVerbose"
                        def files = findFiles(glob: "**BUILD-${BUILD_NUMBER}/**/buildList.txt")
                        // Do not deploy if nothing in the build list
                        needDeploy = files.length > 0 && files[0].length > 0
                    }
                }
            }
            post {
                always {
                    node( zOsAgentLabel ) {
                        dir("${WORKSPACE}/BUILD-${BUILD_NUMBER}") {
                            archiveArtifacts allowEmptyArchive: true,
                                            artifacts: '**/*.log,**/*.json,**/*.html',
                                            excludes: '**/*clist',
                                            onlyIfSuccessful: false
                        }
                    }
                }
            }
        }
        
        stage('UCD Package') {
            steps {
                script {
                    node( zOsAgentLabel ) { 
                        if ( needDeploy ) {
                            artiUrl = env.ARTIFACTORY_URL
                            repositoryPath = env.ARTIFACTORY_REPO_PATH
                            ucdBuztool = env.UCD_BUZTOOL_PATH
                            BUILD_OUTPUT_FOLDER = sh (script: "ls ${WORKSPACE}/BUILD-${BUILD_NUMBER}", returnStdout: true).trim()
                            dir("${WORKSPACE}/BUILD-${BUILD_NUMBER}/${BUILD_OUTPUT_FOLDER}") {
                                withCredentials([usernamePassword(credentialsId: artiCredentialsId, usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                                    writeFile file: "${WORKSPACE}/BUILD-${BUILD_NUMBER}/artifactoy.properties", encoding: "ibm-1047",
                                       text:"""password=$PASSWORD
username=$USERNAME
Repository_type=artifactory
repository=${repositoryPath}
url=${artiUrl}
                                      """
                                }
                                sh "$dbbHome/bin/groovyz $dbbGroovyzOpts ${WORKSPACE}/dbb/Pipeline/CreateUCDComponentVersion/dbb-ucd-packaging.groovy --buztool ${ucdBuztool} --component ${ucdComponent} --workDir ${WORKSPACE}/BUILD-${BUILD_NUMBER}/${BUILD_OUTPUT_FOLDER} --artifactRepository ${WORKSPACE}/BUILD-${BUILD_NUMBER}/artifactoy.properties"
                            }
                        }
                    }
                }
            }
        }
        
        stage('UCD Deploy') {
            steps {
                script{
                    if ( needDeploy ) {
                        script{
                            step(
                                  [$class: 'UCDeployPublisher',
                                    deploy: [
                                        deployApp: ucdApplication,
                                        deployDesc: 'Requested from Jenkins',
                                        deployEnv: ucdEnv,
                                        deployOnlyChanged: false,
                                        deployProc: ucdProcess,
                                        deployVersions: ucdComponent + ':latest'],
                                    siteName: ucdSite])
                        }
                    }
                }
            }
        }
    }
}
