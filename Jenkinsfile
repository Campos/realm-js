#!groovy
import groovy.json.JsonOutput

repoName = 'realm-js' // This is a global variable

def getSourceArchive() {
  getSourceArchiveForCommit(repoName, env.BRANCH_NAME)
}

def getSourceArchiveForCommit(repoName, branchName) {
  checkout([
      $class: 'GitSCM',
      browser: [$class: 'GithubWeb', repoUrl: "git@github.com:realm/${repoName}.git"],
      extensions: [
          [$class: 'CleanCheckout'],
          [$class: 'WipeWorkspace'],
          [$class: 'SubmoduleOption', recursiveSubmodules: true]
      ],
      gitTool: 'native git',
      userRemoteConfigs: [[
          credentialsId: 'realm-ci-ssh',
          name: 'origin',
          refspec: '+refs/tags/*:refs/remotes/origin/tags/* +refs/heads/*:refs/remotes/origin/*',
          url: "git@github.com:realm/${repoName}.git"
      ]]
  ])
}

def readGitTag() {
  sh "git describe --exact-match --tags HEAD | tail -n 1 > tag.txt 2>&1 || true"
  def tag = readFile('tag.txt').trim()
  return tag
}

def readGitSha() {
  sh "git rev-parse HEAD | cut -b1-8 > sha.txt"
  def sha = readFile('sha.txt').readLines().last().trim()
  return sha
}

def getVersion(){
  def dependencies = readProperties file: 'dependencies.list'
  def gitTag = readGitTag()
  def gitSha = readGitSha()
  if (gitTag == "") {
    return "${dependencies.VERSION}-g${gitSha}"
  }
  else {
    return dependencies.VERSION
  }
}

def setBuildName(newBuildName) {
  currentBuild.displayName = "${currentBuild.displayName} - ${newBuildName}"
}

def gitTag
def gitSha
def dependencies
def version

stage('check') {
  node('docker') {
    sh "env"
    getSourceArchive()

    dependencies = readProperties file: 'dependencies.list'

    gitTag = readGitTag()
    gitSha = readGitSha()
    version = getVersion()
    echo "tag: ${gitTag}"
    if (gitTag == "") {
      echo "No tag given for this build"
      setBuildName("${gitSha}")
    } else {
      if (gitTag != "v${dependencies.VERSION}") {
        echo "Git tag '${gitTag}' does not match v${dependencies.VERSION}"
      } else {
        echo "Building release: '${gitTag}'"
        setBuildName("Tag ${gitTag}")
      }
    }
    echo "version: ${version}"
  }
}


def doBuild(target, configuration) {
  def nodeSpec = "docker"
  if (target == "react-tests-android") {
    nodeSpec = "FastLinux"
  } else if (target == "macos") {
    nodeSpec = 'osx_vegas'
  }

  node(nodeSpec) {
    getSourceArchive()
    sh """
      if [ ${target} = node-linux ]; then
        bash scripts/docker-test.sh node ${configuration}
      else
        bash scripts/test.sh ${target} ${configuration}
      fi
    """
    //step([$class: 'GitHubCommitStatusSetter', contextSource: [$class: 'ManuallyEnteredCommitContextSource', context: "${target}_${configuration}"]])
  }
}

stage('build') {
  def configurations = ['Debug', 'Release']
  def targets = ['node', 'node-linux']
  def jobs = [:]
  for (int i = 0; i < targets.size(); i++) {
    def targetName = targets[i];
    for (int j = 0; j < configurations.size(); j++) {
      def configurationName = configurations[j];
  
      jobs["${targetName}_${configurationName}"] = {
        doBuild(targetName, configurationName)
      }
    }
  }
  parallel(jobs)
}