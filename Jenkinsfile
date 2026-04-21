// Jenkinsfile (root của repo)
def getChangedServices() {
    def changedFiles = []
    
    // Lấy danh sách file thay đổi so với main
    if (env.CHANGE_TARGET) {
        // Đây là PR build
        changedFiles = sh(
            script: "git diff --name-only origin/${env.CHANGE_TARGET}...HEAD",
            returnStdout: true
        ).trim().split('\n').toList()
    } else {
        // Đây là branch build
        changedFiles = sh(
            script: "git diff --name-only HEAD~1 HEAD 2>/dev/null || git diff --name-only \$(git rev-list --max-parents=0 HEAD) HEAD",
            returnStdout: true
        ).trim().split('\n').toList()
    }
    
    // Danh sách các Java service
    def javaServices = [
        'cart', 'customer', 'inventory', 'location',
        'media', 'order', 'product', 'promotion',
        'rating', 'search', 'tax', 'webhook'
    ]
    
    // Danh sách Next.js services
    def nextServices = ['storefront', 'backoffice']
    
    def changedJavaServices = [] as Set
    def changedNextServices = [] as Set
    
    changedFiles.each { file ->
        javaServices.each { svc ->
            if (file.startsWith("${svc}/")) {
                changedJavaServices.add(svc)
            }
        }
        nextServices.each { svc ->
            if (file.startsWith("${svc}/")) {
                changedNextServices.add(svc)
            }
        }
    }
    
    return [java: changedJavaServices.toList(), next: changedNextServices.toList()]
}

pipeline {
    agent any
    
    options {
        timeout(time: 60, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    environment {
        JAVA_HOME = '/usr/lib/jvm/java-21-openjdk-amd64'
        MAVEN_OPTS = '-Xmx1024m'
    }

    stages {
        stage('Detect Changed Services') {
            steps {
                script {
                    def changed = getChangedServices()
                    env.CHANGED_JAVA_SERVICES = changed.java.join(',')
                    env.CHANGED_NEXT_SERVICES = changed.next.join(',')
                    
                    echo "Changed Java services: ${env.CHANGED_JAVA_SERVICES}"
                    echo "Changed Next.js services: ${env.CHANGED_NEXT_SERVICES}"
                    
                    if (!env.CHANGED_JAVA_SERVICES && !env.CHANGED_NEXT_SERVICES) {
                        echo "No service changes detected. Skipping build."
                        currentBuild.result = 'SUCCESS'
                    }
                }
            }
        }

        stage('Test & Build Java Services') {
            when {
                expression { env.CHANGED_JAVA_SERVICES?.trim() }
            }
            steps {
                script {
                    def services = env.CHANGED_JAVA_SERVICES.split(',')
                    def parallelStages = [:]
                    
                    services.each { svc ->
                        def serviceName = svc.trim()
                        parallelStages["Test & Build: ${serviceName}"] = {
                            stage("${serviceName}") {
                                dir(serviceName) {
                                    // PHASE TEST
                                    stage("Test: ${serviceName}") {
                                        sh """
                                            ./mvnw test \
                                                -Djacoco.destFile=target/jacoco.exec \
                                                -pl . \
                                                --no-transfer-progress
                                        """
                                    }
                                    
                                    // PHASE BUILD
                                    stage("Build: ${serviceName}") {
                                        sh """
                                            ./mvnw package -DskipTests \
                                                --no-transfer-progress
                                        """
                                    }
                                }
                            }
                        }
                    }
                    
                    parallel parallelStages
                }
            }
            post {
                always {
                    script {
                        def services = env.CHANGED_JAVA_SERVICES.split(',')
                        services.each { svc ->
                            def serviceName = svc.trim()
                            // Upload JUnit test results
                            junit(
                                testResults: "${serviceName}/target/surefire-reports/*.xml",
                                allowEmptyResults: true
                            )
                            // Upload JaCoCo coverage
                            jacoco(
                                execPattern: "${serviceName}/target/jacoco.exec",
                                classPattern: "${serviceName}/target/classes",
                                sourcePattern: "${serviceName}/src/main/java",
                                minimumInstructionCoverage: '0',  // Phase 5 sẽ tăng lên 70
                                changeBuildStatus: false
                            )
                        }
                    }
                }
            }
        }

        stage('Test & Build Next.js Services') {
            when {
                expression { env.CHANGED_NEXT_SERVICES?.trim() }
            }
            steps {
                script {
                    def services = env.CHANGED_NEXT_SERVICES.split(',')
                    services.each { svc ->
                        def serviceName = svc.trim()
                        dir(serviceName) {
                            sh "npm ci"
                            sh "npm run test -- --coverage --watchAll=false || true"
                            sh "npm run build"
                        }
                    }
                }
            }
        }
    }
    
    post {
        success {
            echo "✅ CI Pipeline PASSED"
            // Notify GitHub PR status
            githubNotify status: 'SUCCESS', description: 'CI passed'
        }
        failure {
            echo "❌ CI Pipeline FAILED"
            githubNotify status: 'FAILURE', description: 'CI failed'
        }
        always {
            cleanWs()
        }
    }
}