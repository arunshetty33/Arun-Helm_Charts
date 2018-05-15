properties([
    parameters([
        string(name: 'CLI_TOOLS_IMAGE_NAME', description: 'Name of the CLI Tools container to utilize for the pipline', defaultValue: 'cli-tools'),
        string(name: 'CLI_TOOLS_IMAGE_TAG', description: 'Tag of the CLI Tools container to utilize for the pipline', defaultValue: 'latest'),
        string(name: 'AWS_ECR_REGISTRY', description: 'Base path for the ECR repo', defaultValue: '085170591206.dkr.ecr.us-east-1.amazonaws.com'),
        string(name: 'JENKINS_ECR_CREDENTIALS', description: 'Jenkins credentials to be utilized for ECR and S3 access', defaultValue: 'ecr:us-east-1:exp-dev'),
        //string(name: 'GIT_COMMIT_OVERRIDE', description: 'Explicit Git commit to compare against', defaultValue: '')
        choice(choices: ["sandbox", "dev", "test", "infrastructure", "uat", "prod"].join("\n"), description: 'Environment to deploy to', name: 'ENVIRONMENT', defaultValue: 'dev'),
        choice(choices: ["edp", "emp", "system"].join("\n"), description: 'Platform to deploy to', name: 'PLATFORM', defaultValue: 'edp'),
        string(name: 'RELEASE', description: 'Release to deploy', defaultValue: '')
    ])
])

def newRelease = false
def deployedCharts
def chartInfo

node('edp') {
  stage('Chart checkout') {
    def pwd = pwd()
    echo "Executing on ${GroovySystem.version} in ${pwd}"
    def scmInfo = checkout(scm)
    echo sh(returnStdout: true, script: 'env')

    // Check if release exists in project
    def chartDeployments = loadAllChartDeployments(params.ENVIRONMENT)
    if (!doesReleaseExistInProject(params.RELEASE, chartDeployments)) {
      currentBuild.rawBuild.result = Result.ABORTED
      throw new hudson.AbortException("${params.RELEASE} doesn't exist in project!")
    }

    // Get chart info to use when displaying deploy prompt.  Need to get before stash.
    chartInfo = getChartInfoByReleaseAndEnvironment(params.RELEASE, params.ENVIRONMENT, chartDeployments)
  }
}

node('edp') {
  stage ("${params.ENVIRONMENT} - Lint") {
    processChart(params.RELEASE, params.ENVIRONMENT, chartInfo, this.&helm_lint)
  }
}

node('edp') {
  stage("${params.ENVIRONMENT} - Diff") {
    // Check if it's a new release and has never been deployed
    deployedCharts = getDeployedCharts(params.ENVIRONMENT)
    newRelease = deployedCharts.any({ it.name == params.RELEASE })
    processChart(params.RELEASE, params.ENVIRONMENT, chartInfo, this.&helm_diff)

    stash('helm-chart-workspace')
  }
}

// This must reside outside any node/stage otherwise it doesn't actually short circuit
if (!isMaster() && params.ENVIRONMENT != "dev") {
  echo "On a non-master branch, promotion to alternate environments is disabled"
  currentBuild.result = 'SUCCESS'
  return
}

stage("${params.ENVIRONMENT} - Promotion prompt") {
  // Print out some info on what's going to updated
  echo "Chart to deploy to ${params.ENVIRONMENT}: Release=${chartInfo.release} Chart:${chartInfo.chart} Version=${chartInfo.version} Namespace=${chartInfo.namespace}"
  if (newRelease) {
    echo "New Release"
  } else {
    deployedChart = deployedCharts.find { it.release == params.RELEASE }
    echo "Chart deployed in ${params.ENVIRONMENT}: Release=${deployedChart.release} Chart:${deployedChart.chart} Version=${deployedChart.version} Namespace=${deployedChart.namespace} "
  }

  timeout(time:5, unit:'DAYS') {
    input "Promote changes to ${params.ENVIRONMENT}?"
  }
}

node('edp') {
  stage("${params.ENVIRONMENT} - Promotion") {
    unstash('helm-chart-workspace')
    processChart(params.RELEASE, params.ENVIRONMENT, chartInfo, this.&helm_install)
  }
}

////// Helpers

def processChart(release, environment, chartInfo, callback) {
  def chart = chartInfo.chart
  docker.withRegistry("https://${params.AWS_ECR_REGISTRY}", "${params.JENKINS_ECR_CREDENTIALS}") {
    docker.image("${params.AWS_ECR_REGISTRY}/${params.CLI_TOOLS_IMAGE_NAME}:${params.CLI_TOOLS_IMAGE_TAG}").inside {
      echo "Rendering a kubeconfig for helm to use"
      render_helmconfig(environment, '/tmp/kubeconfig')

      echo "Performing dry-run installation on cluster"
      def localChartInfo = [
        release: release,
        env: environment,
        platform: chartInfo.platform,
        chart: chart,
        kubeconfig: '/tmp/kubeconfig'
      ]
      withDecryptedValues(localChartInfo) {
        callback.call(localChartInfo)
      }
    }
  }
}

def loadAllChartDeployments(environment)
{
  def yamlDeploymentFiles = [charts: []]
  def deployments = sh(returnStdout: true, script: "find ./deployments/${environment} -name charts.yaml").trim().split('\n').toList()

  deployments.each() { deploymentFile ->
    // Extract the platform name from the path hiearchy
    // [., deployments, prod, system, charts.yaml]
    def platformName = deploymentFile.tokenize('/')[3]

    // Extract all the releases and augment with a platform attribute
    def newCharts = readYaml(file: deploymentFile).charts.collect {
      it.platform = platformName
      return it
    }

    yamlDeploymentFiles.charts.addAll(newCharts)
  }
  return yamlDeploymentFiles
}

def getChartInfos(chartName, environment, chartDeployments)
{
  def chartInfos = []
  // Lookup chart info from config file
  def charts = chartDeployments.charts.findAll {chart -> chart.chart.tokenize('/').last() == chartName}
  def chartVersion = getChartVersion(chartName)

  charts.each () { chart ->
    def chartInfo = [
        chart: chart.chart.tokenize('/').last(),
        env: environment,
        kubeconfig: '/tmp/kubeconfig',
        namespace: chart.namespace,
        release: chart.name,
        version: chartVersion,
        values: chart.values
      ]
    chartInfos.add(chartInfo)
  }
  return chartInfos
}

def getChartInfoByReleaseAndEnvironment(release, environment, chartDeployments)
{
  def chart = chartDeployments.charts.find {chart -> chart.name == release}
  //def chartVersion = getChartVersion(chartName)

  def chartInfo = [
      chart: chart.chart.tokenize('/').last(),
      platform: chart.platform,
      env: environment,
      kubeconfig: '/tmp/kubeconfig',
      namespace: chart.namespace,
      release: chart.name,
      version: getChartVersion(chart.chart.tokenize('/').last()),
      values: chart.values
    ]

  return chartInfo
}

def render_helmconfig(environment, kubeconfig) {
  withCredentials([string(credentialsId: "KUBE_${environment.toUpperCase()}_TILLER_TOKEN", variable: 'TILLER_TOKEN')]) {
    sh """
      pwd
      cp ./kubeconfig-base.yaml ${kubeconfig}
      export KUBECONFIG=/tmp/kubeconfig
      set +x; kubectl config set-credentials kube.${environment}.healthgrades.io --token ${TILLER_TOKEN}; set -x;
      kubectl config use-context kube.${environment}.healthgrades.io
    """.stripIndent().trim()
  }
}

def helm_lint(Map args) {
  echo "Linting ${args.chart}"
  sh "helm lint charts/${args.chart}"
}

def helm_diff(Map args) {
  echo "Diffing ${args.chart}"
  ansiColor('xterm') {
    sh "KUBECONFIG=${args.kubeconfig} ./scripts/helm_wrapper.sh -e ${args.env} -p ${args.platform} -c ${args.chart} diff ${args.release} --suppress-secrets"
  }
}

def helm_install(Map args) {
  echo "Installing ${args.chart}"
  sh "KUBECONFIG=${args.kubeconfig} ./scripts/helm_wrapper.sh -e ${args.env} -p ${args.platform} -c ${args.chart} upgrade ${args.release}"
}

// Decrypts a given helm chart/env combination for the duration of the closure
// and attempts to clean up after itself
def withDecryptedValues(Map args, closure) {
  withCredentials([
    // Leverages the environment specific Jenkins Credential to procure IAM permissions.
    [$class: 'AmazonWebServicesCredentialsBinding', credentialsId: "exp-${args.env}-helm"]
  ]) {
    echo "Decrypting the chart ${args.chart} in ${args.env}"
    sh "./scripts/process_secrets.sh dec -e ${args.env} -p ${args.platform} -c ${args.chart}"

    // Just echo out the changes to ensure that is what we are wanting
    sh "git clean -dXn"

    // Attempt the callback and cleanup after self
    try {
      closure.call()
    } finally {
      echo "Attempting to cleanup secrets"
      // Also need to ensure that the .gitignore values for the `.dec` files are removed.
      sh "git clean -dXf ./values/*/${args.env}/"

      sh "git clean -dXn"
    }
  }
}

def getDeployedCharts(environment) {
  def helmListOutput = helmList(environment)
  def list = delimitedStringToList(helmListOutput, true, '\t')

  return list.collect { chart ->
    return [
      chart: chart[4].trim().tokenize('-').init().join('-'),
      env: environment,
      release: chart[0].trim(),
      namespace: chart[5].trim(),
      version: chart[4].trim().tokenize('-').last()
    ]
  }
}

def helmList(environment) {
  docker.withRegistry("https://${params.AWS_ECR_REGISTRY}", "${params.JENKINS_ECR_CREDENTIALS}") {
    docker.image("${params.AWS_ECR_REGISTRY}/${params.CLI_TOOLS_IMAGE_NAME}:${params.CLI_TOOLS_IMAGE_TAG}").inside {
      echo "Rendering a kubeconfig for helm to use"
      render_helmconfig(environment, '/tmp/kubeconfig')

      withCredentials([string(credentialsId: "KUBE_${environment.toUpperCase()}_TILLER_TOKEN", variable: 'TILLER_TOKEN')]) {
        def output = sh(returnStdout: true, script: "KUBECONFIG='/tmp/kubeconfig' helm ls").trim()
        return output
      }
    }
  }
}

def delimitedStringToList(delimitedString, boolean skipFirstRow, delimiter) {
  def lines = delimitedString.split('\n')
  if (skipFirstRow) {
    lines = lines.drop(1)
  }
  def rows = lines.collect {it.tokenize(delimiter)}
  return rows
}

def doesReleaseExistOnServer(release, namespace, deployedCharts) {
  def exists = deployedCharts.findAll { chart ->
    chart.release == release
  }
  return exists.size() > 0
}

def doesReleaseExistInProject(release, chartDeployments) {
  def exists = chartDeployments.charts.findAll { chart ->
    chart.name == release
  }
  return exists.size() > 0
}

def getChartVersion(chartName) {
  def file = "./charts/${chartName}/Chart.yaml"
  def data = readYaml file: file
  return data.version
}

// NOTE: This is essentially "dead" code now, as we don't use this to determine what we want to run Helm or Helmfile against.
//  the idea was that this would have been the way of dynamically determining what charts had changed and auto-run the charts
//  At the time, we had a terrible time with the Jenkins box as it was shared infra and we could not get the things we needed
//  added there.

// Requirements & Expectations:
// - Proper credentials matching the pattern: exp-dev, exp-test, exp-prod that map to AWS credentials capable of using the KMS decrypt process.


def processAllCharts(changedCharts, environment, chartDeployments, callback) {
  docker.withRegistry("https://${params.AWS_ECR_REGISTRY}", "${params.JENKINS_ECR_CREDENTIALS}") {
    docker.image("${params.AWS_ECR_REGISTRY}/${params.CLI_TOOLS_IMAGE_NAME}:${params.CLI_TOOLS_IMAGE_TAG}").inside {
      echo "Rendering a kubeconfig for helm to use"
      render_helmconfig(environment, '/tmp/kubeconfig')

      echo "Performing dry-run installation on cluster"
      changedCharts.each { changedChart ->
        def chartInfos = getChartInfos(changedChart, environment, chartDeployments)
        chartInfos.each { chartInfo ->
          withDecryptedValues(chartInfo) {
            callback.call(chartInfo)
          }
        }
      }
    }
  }
}

def isMaster() {
  return env.BRANCH_NAME == 'master'
}

//Find the most recent successful build that had a changeset associated with it
@NonCPS
def lastSuccessfulCommit(build) {
  echo "Looking up commits from build ${build.number} ${build.result} - ${build.changeSets.size()}"
  if (build.result == 'SUCCESS' && build.changeSets.size() == 1) {
    return build.changeSets[0].items[0].commitId
  }

  return build.previousBuild ? lastSuccessfulCommit(build.previousBuild) : ""
}

def determineCommitSpread(explicitCommit) {
  // If given an explicit commit, always use it
  if (explicitCommit != '') {
    return explicitCommit
  }

  // Feature branches are compared to origin/master
  if (!isMaster()) {
    // ... will give us the changes that have occurred in the current branch compared to `origin/master`
    // without this, we would also see the changes that had been merged into origin/master and deployed
    return 'origin/master...'
  }

  // Master branch needs to look up last successful build from history to determine changes
  def commitSpread = lastSuccessfulCommit(currentBuild)

  // If we never had a succesful build with changes, we fail.
  // TODO - We could take the very first commit in the repo instead
  if (commitSpread == '') {
    error "Unable to proceed with pipeline, missing proper commit spread. Specificy a parameter if necessary"
  }

  return commitSpread
}

def determineChangedFiles(commitSpread) {
  return [] + sh(returnStdout: true, script: "git diff --find-renames --name-only ${commitSpread}").trim().split('\n').toList()
}

@NonCPS
def determineChangedCharts(files) {
  def nonDeployedCharts = ['common']

  // files = "git diff --find-renames --name-only ${commitSpread}".execute().text.trim().split('\n')
  //   Find all the charts that are changed
  // ex. charts/credentials/templates/snowflake-configmap.yaml => credentials
  def charts = files.
    findAll{ fullPath -> fullPath =~ /^charts/ }.
    collect{ fullPath -> fullPath.split('/') }.
    collect{ pathSegements -> pathSegements[1] }

  //Add values yaml changes
  // ex. values/edp/prod/us-east-1/streamsets/secrets-tls.yaml => streamsets
  charts += files.
       findAll{ fullPath -> fullPath =~ /^values.*yaml$/ }.
       collect{ fullPath -> fullPath.split('/') }.
       findAll{ pathSegements -> pathSegements.size() == 6 }.
       collect{ pathSegements -> pathSegements[4] }

  charts.unique()

  // Remove the non-deployable charts
  nonDeployedCharts.each { nonDeployable ->
    charts.remove(nonDeployable)
  }

  return charts
}
