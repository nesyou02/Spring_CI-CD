pipeline {
  agent any

  tools {
    jdk   'jdk17'
    maven 'maven3'
  }
  environment {
    DOCKER_CRED = 'dockerhub'
    SONAR_TOKEN = credentials('sonar-token')
    SONAR_URL   = 'http://localhost:9000'
    NEXUS_URL   = 'http://172.16.159.134:8081/repository/maven-snapshots/'
  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }
    stage('Build & Test') {
        sh 'mvn clean package -B'
    stage('SonarQube Analysis') {
        catchError(buildResult: 'UNSTABLE', stageResult: 'UNSTABLE') {
          withSonarQubeEnv('demo') {
    sh "mvn sonar:sonar -Dsonar.projectKey=demo -Dsonar.host.url=$SONAR_URL -Dsonar.login=$SONAR_TOKEN"
            sh "mvn sonar:sonar -Dsonar.host.url=${SONAR_URL} -Dsonar.login=${SONAR_TOKEN}"
          }
        }
    stage('Quality Gate') {
          timeout(time: 2, unit: 'MINUTES') {
            script {
              def qg = waitForQualityGate()
              echo "Quality Gate: ${qg.status}"
              if (qg.status != 'OK') {
                currentBuild.result = 'UNSTABLE'
              }
            }
    stage('Docker Login, Build & Push') {
stage('Trivy Scan') {      steps {        withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {          sh "docker pull $DOCKER_USER/demoapp:${GIT_COMMIT}";          sh "trivy image --exit-code 1 --severity HIGH,CRITICAL $DOCKER_USER/demoapp:${GIT_COMMIT}";        }      }    }
        withCredentials([usernamePassword(
          credentialsId: DOCKER_CRED,
          usernameVariable: 'DOCKER_USER',
          passwordVariable: 'DOCKER_PASS'
        )]) {
          sh '''
            echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
            docker build -t $DOCKER_USER/demoapp:${GIT_COMMIT} .
            docker push $DOCKER_USER/demoapp:${GIT_COMMIT}
            docker tag $DOCKER_USER/demoapp:${GIT_COMMIT} $DOCKER_USER/demoapp:latest
            docker push $DOCKER_USER/demoapp:latest
          '''
    stage('Deploy to Nexus') {
  steps {
    echo '📦 Déploiement du JAR vers Nexus (maven-snapshots)'
    withCredentials([usernamePassword(
      credentialsId: 'nexus-credentials',
      usernameVariable: 'NEXUS_USER',
      passwordVariable: 'NEXUS_PASS'
    )]) {
      // Crée un settings.xml temporaire avec les credentials
      sh '''cat > settings.xml <<EOF
<settings>
  <servers>
    <server>
      <id>nexus</id>
      <username>$NEXUS_USER</username>
      <password>$NEXUS_PASS</password>
    </server>
  </servers>
</settings>
EOF'''

      // Utilise le nouveau settings.xml et corrige la syntaxe du repository
      sh 'mvn deploy -B -s settings.xml -DaltDeploymentRepository=nexus::http://172.16.159.134:8081/repository/maven-snapshots/'
}
  post {
    success  { echo '✅ Pipeline terminé avec succès' }
    unstable { echo '⚠️ Pipeline instable (vérifier les logs)' }
    failure  { echo '❌ Pipeline échoué' }
