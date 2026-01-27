node {

  /*************** Job Parameters + Push Trigger ******************/
  properties([
    // ADDED: GitHub push trigger so webhook can start this job
    pipelineTriggers([githubPush()]),

    parameters([
      string(name: 'GIT_USER',        defaultValue: 'samaharshareddy', description: 'GitHub org/user'),
      string(name: 'BASE_GIT_URL',    defaultValue: 'https://github.com/samaharshareddy', description: 'Base Git URL (no trailing slash)'),
      string(name: 'REPO_NAMES',      defaultValue: 'ubi-sales-api',   description: 'Comma-separated repo names (no .git)'),
      // CHANGED: keep parameter for manual runs, but we’ll override with pushed branch
      choice(name: 'BRANCH',          choices: ['main', 'master', 'uat', 'feature/dev','feature/dc'] , description: 'Git branch to build (fallback if push branch not detected)'),
      choice(name: 'BUILD_MODE',      choices: ['single', 'bundle'],   description: 'single: one image per repo; bundle: one image with all JARs'),
      booleanParam(name: 'RUN_SONAR', defaultValue: false,             description: 'Run mvn sonar:sonar'),
      booleanParam(name: 'FAIL_ON_MISSING_BRANCH', defaultValue: false, description: 'Fail if a repo branch is missing'),
      // Add these if you use them later:
      //string(name: 'IMAGE_NAME',      defaultValue: 'myapp',           description: 'Docker image repository name'),
      //string(name: 'HARBOR_REGISTRY', defaultValue: 'harbor.local/library', description: 'Harbor registry/repo'),
      //string(name: 'GIT_CREDENTIALS', defaultValue: '', description: 'Optional Git credentials ID')
    ])
  ])

  // Scripted pipeline: set environment variables like this
  env.CENTRAL_DOCKER_DIR  = "/opt/mule-docker"
  env.CENTRAL_DOCKERFILE  = "/var/lib/jenkins/workspace"
  env.JAR_PATH            = "/var/lib/jenkins/workspace/jars"

  // ADDED: central list of allowed branches
  def ALLOWED_BRANCHES = ['main','master','uat','feature/dev','feature/dc']

  // ADDED: resolve pushed branch (webhook/SCM), fallback to parameter
  boolean buildAllowed = true
  stage('Resolve Branch') {
    echo 'Resolving effective branch to build...'
    // Generic Webhook Trigger plugin sets env.gh_branch like: refs/heads/feature/dev
    def inferred = (env.gh_branch ?: '').trim()
    if (inferred) {
      inferred = inferred.replace('refs/heads/','')
      echo "Detected branch from webhook payload: ${inferred}"
    } else if ((env.GIT_BRANCH ?: '').trim()) {
      // When job is defined 'Pipeline script from SCM', Jenkins sets env.GIT_BRANCH like origin/feature/dev
      inferred = env.GIT_BRANCH.replaceFirst(/^origin\//,'').replaceFirst(/^refs\/heads\//,'')
      echo "Detected branch from GIT_BRANCH: ${inferred}"
    } else {
      inferred = params.BRANCH  // fallback for manual runs
      echo "Falling back to parameter BRANCH: ${inferred}"
    }
    env.EFFECTIVE_BRANCH = inferred

    if (!ALLOWED_BRANCHES.contains(env.EFFECTIVE_BRANCH)) {
      echo "[SKIP] Branch '${env.EFFECTIVE_BRANCH}' is not in allowed list ${ALLOWED_BRANCHES}. Exiting."
      currentBuild.result = 'NOT_BUILT'
      buildAllowed = false
    } else {
      echo "Allowed branch: ${env.EFFECTIVE_BRANCH}"
    }
  }

  // Exit early if not allowed branch
  if (!buildAllowed) { return }

  try {
    stage('Pipeline Start') {
      echo 'Pipeline Started'
      env.LANG = 'en_US.UTF-8'
      sh 'rm -rf work docker || true'
      sh 'mkdir -p jars'
    }

    stage('Parse Input') {
      // Split on commas, trim, remove empties, unique
      List<String> repos = params.REPO_NAMES
        .split(',')
        .collect { it.trim() }
        .findAll { it }
        .unique()

      // Validate repo names: letters, digits, dot, dash, underscore
      def invalid = repos.findAll { !(it ==~ /[A-Za-z0-9._-]+/) }
      if (invalid) {
        error "Invalid repository names: ${invalid.join(', ')}. Allowed: letters, digits, dot, dash, underscore."
      }

      // Build URLs (either BASE_GIT_URL or GIT_USER pattern)
      List<String> urls = repos.collect { "${params.BASE_GIT_URL}/${it}.git" }
      // Or: List<String> urls = repos.collect { "https://github.com/${params.GIT_USER}/${it}.git" }

      // Save for later stages
      env.REPO_LIST = repos.join(',')
      env.URL_LIST  = urls.join(',')

      echo "Repos: ${repos}"
      echo "URLs:  ${urls}"
    }

    // ---------- SonarQube Analysis (parallel) ----------
    stage('SonarQube Analysis (parallel)') {
      if (!params.RUN_SONAR) {
        echo "RUN_SONAR=false. Skipping SonarQube analysis."
        // keep your early return behavior
        return
      }

      def repos = env.REPO_LIST.split(',').collect { it.trim() }.findAll { it }
      if (repos.isEmpty()) {
        echo 'No repositories to analyze (REPO_LIST empty)'
        return
      }

      def sonarBranches = [:]

      for (String repo in repos) {
        def repoName = repo

        sonarBranches[repoName] = {
          dir("work/${repoName}") {
            if (!fileExists('pom.xml')) {
              echo "[SKIP] ${repoName} has no pom.xml, skipping Sonar."
              return
            }

            // Unique, stable project key per repo
            def projectKey = "${params.GIT_USER}_${repoName}".replaceAll('[^A-Za-z0-9_.:-]', '_')

            withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
              sh """
                mvn -B -ntp sonar:sonar \
                  -Dsonar.projectKey=${projectKey} \
                  -Dsonar.host.url=http://54.191.55.239:9000 \
                  -Dsonar.login=\${SONAR_TOKEN}
              """
            }
          }
        }
      }

      parallel sonarBranches
    }

    stage('Checkout & Build (parallel)') {
      def repos = env.REPO_LIST.split(',').collect { it.trim() }.findAll { it }
      if (repos.isEmpty()) { error 'No repositories to build (REPO_LIST empty)' }

      def branches = [:]

      for (String repo in repos) {
        def repoName = repo // capture per-iteration value for the closure

        branches[repoName] = {
          dir("work/${repoName}") {
            // Try checkout of ONLY the selected branch; handle missing branch
            def ok = false
            try {
              // CHANGED: use EFFECTIVE_BRANCH resolved from push/webhook/SCM/param
              checkoutRepo(params.BASE_GIT_URL, repoName, env.EFFECTIVE_BRANCH, params.GIT_CREDENTIALS)
              ok = true
            } catch (Throwable e) {
              def msg = "Branch '${env.EFFECTIVE_BRANCH}' not found or checkout failed for repo '${repoName}'."
              if (params.FAIL_ON_MISSING_BRANCH) {
                error msg
              } else {
                echo "[SKIP] ${msg}"
              }
            }

            if (!ok) {
              echo "[SKIP] Build skipped for '${repoName}' due to missing branch."
              return
            }

            // Build (skip tests for speed; adjust as needed)
            sh 'mvn -B -DskipTests clean package'

            // Collect JAR for later
            def jarPath = sh(
              returnStdout: true,
              script: """bash -lc "ls -1 target/*.jar | grep -Ev '(sources|javadoc)' | head -n1 || true" """
            ).trim()

            if (!jarPath) { error "No JAR produced for ${repoName}" }

            sh """mkdir -p "${env.WORKSPACE}/jars" """
            sh """cp "${jarPath}" "${env.WORKSPACE}/jars/${repoName}.jar" """

            echo "Built ${repoName}: ${jarPath} -> jars/${repoName}.jar"
          }
        }
      }

      parallel branches
    }

    // ----------- Docker Image Creation -----------------
    stage('Determine Image Version') {
      script {
        def latest = sh(
            script: """
                docker images --format '{{.Tag}}' ${params.REPO_NAMES} \
                | grep '^v' \
                | sed 's/v//' \
                | sort -n \
                | tail -1
            """,
            returnStdout: true
        ).trim()

        if (!latest) {
            env.IMAGE_VERSION = "v1"
        } else {
            env.IMAGE_VERSION = "v${latest.toInteger() + 1}"
        }

        env.FULL_IMAGE = "${params.REPO_NAMES}:${env.IMAGE_VERSION}"

        echo "Resolved version: ${env.IMAGE_VERSION}"
      }
    }

    stage('Docker Build') {
      sh '''
        cd ..
        docker build -t "$FULL_IMAGE" .
        docker tag $FULL_IMAGE nagireddy77/petclinic:$IMAGE_VERSION
      '''
    }

    stage('Docker Push') {
      withCredentials([usernamePassword(
          credentialsId: 'docker-hub',
          usernameVariable: 'DOCKER_USER',
          passwordVariable: 'DOCKER_PASS'
      )]) {

          sh """
              echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
              docker push nagireddy77/petclinic:$IMAGE_VERSION
              docker logout
          """
      }
    }

  } finally {
    echo 'Pipeline finished (finally block).'
    // cleanWs(deleteDirs: true, notFailBuild: true)
  }
}

/**************** Helper(s) ****************/
def checkoutRepo(baseUrl, repo, branch, gitCredsId) {
  def repoUrl = "${baseUrl}/${repo}.git"
  echo "Cloning ${repoUrl} @ ${branch}"
  if (gitCredsId?.trim()) {
    checkout([$class: 'GitSCM',
      branches: [[name: "*/${branch}"]],
      extensions: [[$class: 'CloneOption', shallow: true, depth: 1]],
      userRemoteConfigs: [[url: repoUrl, credentialsId: gitCredsId]]
    ])
  } else {
    checkout([$class: 'GitSCM',
      branches: [[name: "*/${branch}"]],
      extensions: [[$class: 'CloneOption', shallow: true, depth: 1]],
      userRemoteConfigs: [[url: repoUrl]]
    ])
  }
}
