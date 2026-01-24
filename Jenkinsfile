pipeline {
    agent any
    
    // ===== Zmienne globalne =====
    environment {
        // Credentials
        GITHUB_TOKEN = credentials('github-token')
        
        // SF CLI PATH
        PATH = "C:\\Program Files\\sf\\client\\bin;${env.PATH}"
        
        // Zmienne z webhooków
        PR_BRANCH = "${env.pr_branch ?: 'SIT'}"
        PR_NUMBER = "${env.pr_number ?: env.issue_number}"
        PR_ACTION = "${env.pr_action}"
        COMMENT_BODY = "${env.comment_body}"
        SOURCE_BRANCH = "${env.pr_head_branch}"
        REPO_NAME = "${env.repo_name}"
        
        // Zmienne konfiguracyjne
        SF_ALIAS = "SIT"
        DEPLOY_DIR = "force-app/main/default"
        
        // Zmienne statusu
        IS_APPROVED = 'false'
        IS_DEPLOYED = 'false'
        IS_VALIDATED = 'false'
        
        // Akumulator wiadomości
        OUTPUT_MESSAGE = ""
    }
    
    stages {
        
        // ===== STAGE 0: Filtrowanie triggerów =====
        stage('Filter Triggers') {
            steps {
                script {
                    def allowedActions = ['opened', 'synchronize', 'created', 'submitted']
                    
                    echo "========================================"
                    echo "Webhook received!"
                    echo "Action: ${env.pr_action}"
                    echo "PR Number: ${env.pr_number ?: env.issue_number}"
                    echo "Comment: ${env.comment_body}"
                    echo "========================================"
                    
                    if (!(env.pr_action in allowedActions)) {
                        echo "⏭️ Skipping pipeline - action '${env.pr_action}' not in allowed list"
                        currentBuild.result = 'NOT_BUILT'
                        error("Action not in allowed list")
                    }
                    
                    echo "✅ Action '${env.pr_action}' is allowed - continuing pipeline"
                }
            }
        }
        
        // ===== STAGE 1: Sprawdzenie zależności =====
        stage('Check Dependencies') {
            steps {
                script {
                    env.OUTPUT_MESSAGE += "🔍 **Checking dependencies...**\\n\\n"
                    
                    try {
                        echo "Checking Git..."
                        def gitVersion = bat(script: '@git --version', returnStdout: true).trim()
                        echo "✅ Git installed: ${gitVersion}"
                        env.OUTPUT_MESSAGE += "✅ Git: ${gitVersion}\\n"
                    } catch (Exception gitError) {
                        echo "❌ Git not found!"
                        env.OUTPUT_MESSAGE += "❌ Git not found\\n"
                        error("Git is required")
                    }
                    
                    try {
                        echo "Checking Salesforce CLI..."
                        def sfVersion = bat(script: '@sf --version', returnStdout: true).trim()
                        echo "✅ Salesforce CLI found: ${sfVersion}"
                        env.OUTPUT_MESSAGE += "✅ Salesforce CLI: ${sfVersion}\\n"
                    } catch (Exception sfError) {
                        echo "❌ SF CLI not found"
                        env.OUTPUT_MESSAGE += "❌ SF CLI not found\\n"
                        error("Salesforce CLI is required")
                    }
                    
                    env.OUTPUT_MESSAGE += "\\n---\\n\\n"
                }
            }
        }
        
        // ===== STAGE 2: Klonowanie repo i merge =====
        stage('Clone and Merge') {
            steps {
                script {
                    env.OUTPUT_MESSAGE += "📥 **Cloning repository and merging branches...**\\n\\n"
                    
                    try {
                        echo "Cleaning workspace..."
                        deleteDir()
                        
                        echo "Cloning repository: ${REPO_NAME}"
                        bat """
                            git clone https://${GITHUB_TOKEN}@github.com/${REPO_NAME}.git .
                        """
                        env.OUTPUT_MESSAGE += "✅ Repository cloned\\n"
                        
                        echo "Checking out branch: ${PR_BRANCH}"
                        bat """
                            git checkout ${PR_BRANCH}
                        """
                        env.OUTPUT_MESSAGE += "✅ Checked out to: ${PR_BRANCH}\\n"
                        
                        // Merge source branch (jeśli istnieje)
                        if (SOURCE_BRANCH && SOURCE_BRANCH != '' && SOURCE_BRANCH != 'null') {
                            echo "Merging ${SOURCE_BRANCH} into ${PR_BRANCH}"
                            def mergeResult = bat(
                                script: "git merge origin/${SOURCE_BRANCH} --no-commit --no-ff",
                                returnStatus: true
                            )
                            
                            if (mergeResult != 0) {
                                env.OUTPUT_MESSAGE += "⚠️ **MERGE CONFLICT**\\n"
                                error("Merge conflict detected")
                            } else {
                                env.OUTPUT_MESSAGE += "✅ Merged successfully: ${SOURCE_BRANCH} → ${PR_BRANCH}\\n"
                            }
                        } else {
                            echo "No source branch to merge (comment trigger)"
                            env.OUTPUT_MESSAGE += "ℹ️ No branch merge needed (comment trigger)\\n"
                        }
                        
                    } catch (Exception e) {
                        env.OUTPUT_MESSAGE += "❌ Error during clone/merge: ${e.message}\\n"
                        throw e
                    }
                    
                    env.OUTPUT_MESSAGE += "\\n---\\n\\n"
                }
            }
        }
        
        // ===== STAGE 3: Authenticate to Salesforce =====
        stage('Authenticate to Salesforce') {
            steps {
                script {
                    env.OUTPUT_MESSAGE += "🔐 **Authenticating to Salesforce...**\\n\\n"
                    
                    try {

                        withCredentials([
                            file(credentialsId: 'salesforce-jwt-key', variable: 'SF_SERVER_KEY'),
                            string(credentialsId: 'salesforce-client-id', variable: 'SF_CLIENT_ID'),
                            string(credentialsId: 'salesforce-username', variable: 'SF_USERNAME')
                        ]) {
                            echo "Authenticating with JWT..."
                            
                            def authResult = bat(
                                script: """
                                    sf org login jwt ^
                                    --client-id %SF_CLIENT_ID% ^
                                    --jwt-key-file "%SF_SERVER_KEY%" ^
                                    --username %SF_USERNAME% ^
                                    --instance-url https://login.salesforce.com ^
                                    --alias ${SF_ALIAS} ^
                                    --set-default ^
                                    --json
                                """,
                                returnStatus: true
                            )
                            
                            if (authResult == 0) {
                                env.OUTPUT_MESSAGE += "✅ **Authenticated successfully to Salesforce (JWT)**\\n"
                            } else {
                                env.OUTPUT_MESSAGE += "❌ **Authentication failed (JWT)**\\n"
                                error("Salesforce JWT authentication failed")
                            }
                        }
                    
                        
                    } catch (Exception e) {
                        env.OUTPUT_MESSAGE += "❌ Auth error: ${e.message}\\n"
                        env.OUTPUT_MESSAGE += "\\n**💡 Troubleshooting:**\\n"
                        env.OUTPUT_MESSAGE += "- Check if 'salesforce-auth-url' credential exists in Jenkins\\n"
                        env.OUTPUT_MESSAGE += "- Verify the auth URL is valid (run: `sf org display --verbose`)\\n"
                        env.OUTPUT_MESSAGE += "- Ensure the org is still active\\n"
                        throw e
                    }
                    
                    env.OUTPUT_MESSAGE += "\\n---\\n\\n"
                }
            }
        }
        
        // ===== STAGE 4: Sprawdzenie approvals =====
        stage('Check Approvals') {
            steps {
                script {
                    env.OUTPUT_MESSAGE += "✔️ **Checking Pull Request approvals...**\\n\\n"
                    
                    try {
                        def prNum = env.pr_number ?: env.issue_number
                        
                        if (!prNum || prNum == 'null') {
                            echo "No PR number - skipping approval check"
                            env.OUTPUT_MESSAGE += "ℹ️ No PR number - cannot check approvals\\n"
                            env.IS_APPROVED = 'false'
                            env.OUTPUT_MESSAGE += "\\n---\\n\\n"
                            return
                        }
                        
                        echo "Checking approvals for PR #${prNum}"
                        def apiUrl = "https://api.github.com/repos/${REPO_NAME}/pulls/${prNum}/reviews"
                        
                        def response = bat(
                            script: "@curl -s -H \"Authorization: token ${GITHUB_TOKEN}\" -H \"Accept: application/vnd.github.v3+json\" ${apiUrl}",
                            returnStdout: true
                        ).trim()
                        
                        def reviews = readJSON text: response
                        def approvedReviews = reviews.findAll { it.state == 'APPROVED' }
                        
                        if (approvedReviews.size() > 0) {
                            env.IS_APPROVED = 'true'
                            env.OUTPUT_MESSAGE += "✅ **PR IS APPROVED** (${approvedReviews.size()} approval(s))\\n"
                            echo "✅ Found ${approvedReviews.size()} approval(s)"
                        } else {
                            env.IS_APPROVED = 'false'
                            env.OUTPUT_MESSAGE += "⚠️ **PR NOT APPROVED** - deployment will be blocked\\n"
                            echo "⚠️ No approvals found"
                        }
                        
                    } catch (Exception e) {
                        env.OUTPUT_MESSAGE += "❌ Error checking approvals: ${e.message}\\n"
                        env.IS_APPROVED = 'false'
                    }
                    
                    env.OUTPUT_MESSAGE += "\\n---\\n\\n"
                }
            }
        }
        
        // ===== STAGE 5: Validate lub Deploy =====
        stage('Validate or Deploy') {
            steps {
                script {
                    env.OUTPUT_MESSAGE += "🚀 **Salesforce Validation/Deployment...**\\n\\n"
                    
                    try {
                        def action = null
                        
                        // Wykryj komendę z komentarza
                        if (COMMENT_BODY && COMMENT_BODY != '' && COMMENT_BODY != 'null') {
                            def commentLower = COMMENT_BODY.toLowerCase().trim()
                            if (commentLower == 'validate') action = 'validate'
                            if (commentLower == 'deploy') action = 'deploy'
                        }
                        
                        if (!action) {
                            env.OUTPUT_MESSAGE += "ℹ️ No valid command found in comment\\n"
                            env.OUTPUT_MESSAGE += "**Valid commands:** `Validate` or `Deploy`\\n"
                            echo "No valid command - skipping validation/deployment"
                            return
                        }
                        
                        env.OUTPUT_MESSAGE += "📋 **Command detected:** ${action.toUpperCase()}\\n\\n"
                        
                        // Zbuduj komendę
                        def deployCmd = ""
                        
                        if (action == 'validate') {
                            deployCmd = "sf project deploy start --source-dir ${DEPLOY_DIR} --target-org ${SF_ALIAS} --test-level RunLocalTests --dry-run --wait 30 --json"
                            env.OUTPUT_MESSAGE += "⏳ Running **validation** (dry-run)...\\n"
                            echo "Running VALIDATION"
                        } else if (action == 'deploy') {
                            // Sprawdź approval przed deploymentem
                            if (env.IS_APPROVED != 'true') {
                                env.OUTPUT_MESSAGE += "❌ **DEPLOY BLOCKED - PR NOT APPROVED!**\\n"
                                env.OUTPUT_MESSAGE += "Please get approval before deploying\\n"
                                error("Deploy requires PR approval")
                            }
                            
                            deployCmd = "sf project deploy start --source-dir ${DEPLOY_DIR} --target-org ${SF_ALIAS} --test-level RunLocalTests --wait 30 --json"
                            env.OUTPUT_MESSAGE += "⏳ Running **deployment**...\\n"
                            echo "Running DEPLOYMENT"
                        }
                        
                        // Wykonaj deployment/validation
                        echo "Executing: ${deployCmd}"
                        def deployResult = bat(
                            script: "${deployCmd} > deployment-result.json 2>&1",
                            returnStatus: true
                        )
                        
                        // Wyświetl zawartość w logach
                        echo "=== DEPLOYMENT/VALIDATION RESULT ==="
                        bat "type deployment-result.json"
                        echo "===================================="
                        
                        // Zapisz status dla następnego stage
                        env.DEPLOY_EXIT_CODE = "${deployResult}"
                        
                        if (deployResult == 0) {
                            env.OUTPUT_MESSAGE += "✅ Command completed successfully\\n"
                        } else {
                            env.OUTPUT_MESSAGE += "⚠️ Command finished with errors (exit code: ${deployResult})\\n"
                        }
                        
                    } catch (Exception e) {
                        env.OUTPUT_MESSAGE += "❌ Execution failed: ${e.message}\\n"
                        throw e
                    }
                    
                    env.OUTPUT_MESSAGE += "\\n---\\n\\n"
                }
            }
        }
        
        // ===== STAGE 6: Parsowanie rezultatu =====
        stage('Parse Deployment Result') {
            steps {
                script {
                    env.OUTPUT_MESSAGE += "📊 **Analyzing result...**\\n\\n"
                    
                    try {
                        if (!fileExists('deployment-result.json')) {
                            env.OUTPUT_MESSAGE += "ℹ️ No deployment/validation was executed\\n"
                            env.OUTPUT_MESSAGE += "\\n---\\n\\n"
                            return
                        }
                        
                        def resultContent = readFile('deployment-result.json').trim()
                        
                        // Parsuj JSON
                        def resultJson
                        try {
                            resultJson = readJSON text: resultContent
                        } catch (Exception jsonError) {
                            env.OUTPUT_MESSAGE += "⚠️ Could not parse JSON result\\n"
                            env.OUTPUT_MESSAGE += "```\\n${resultContent}\\n```\\n"
                            return
                        }
                        
                        def status = resultJson.result?.status ?: resultJson.status
                        def deployId = resultJson.result?.id ?: 'N/A'
                        
                        env.OUTPUT_MESSAGE += "**Status:** ${status}\\n"
                        env.OUTPUT_MESSAGE += "**Deploy ID:** ${deployId}\\n\\n"
                        
                        // Analiza statusu
                        if (status == 'Succeeded') {
                            env.IS_DEPLOYED = 'true'
                            env.IS_VALIDATED = 'true'
                            env.OUTPUT_MESSAGE += "✅ **SUCCESS!**\\n"
                            
                            // Dodatkowe info o sukcesie
                            if (resultJson.result?.deployedSource) {
                                def deployed = resultJson.result.deployedSource.size()
                                env.OUTPUT_MESSAGE += "📦 Deployed ${deployed} component(s)\\n"
                            }
                            
                        } else if (status == 'Failed') {
                            env.IS_DEPLOYED = 'false'
                            env.IS_VALIDATED = 'false'
                            env.OUTPUT_MESSAGE += "❌ **FAILED**\\n\\n"
                            
                            // Szczegóły błędów
                            if (resultJson.result?.details?.componentFailures) {
                                env.OUTPUT_MESSAGE += "**Errors:**\\n"
                                resultJson.result.details.componentFailures.each { failure ->
                                    env.OUTPUT_MESSAGE += "- ${failure.fileName}: ${failure.problem}\\n"
                                }
                            }
                        } else {
                            env.OUTPUT_MESSAGE += "⚠️ **Status:** ${status}\\n"
                        }
                        
                    } catch (Exception e) {
                        env.OUTPUT_MESSAGE += "❌ Error parsing result: ${e.message}\\n"
                    }
                    
                    env.OUTPUT_MESSAGE += "\\n---\\n\\n"
                }
            }
        }
        
        // ===== STAGE 7: Merge PR =====
        stage('Merge Pull Request') {
            when {
                expression { 
                    return env.IS_DEPLOYED == 'true' && env.IS_APPROVED == 'true' 
                }
            }
            steps {
                script {
                    env.OUTPUT_MESSAGE += "🔀 **Merging Pull Request...**\\n\\n"
                    
                    try {
                        def prNum = env.pr_number ?: env.issue_number
                        
                        if (!prNum || prNum == 'null') {
                            env.OUTPUT_MESSAGE += "⚠️ No PR number - cannot merge\\n"
                            env.OUTPUT_MESSAGE += "\\n---\\n\\n"
                            return
                        }
                        
                        echo "Merging PR #${prNum}"
                        def apiUrl = "https://api.github.com/repos/${REPO_NAME}/pulls/${prNum}/merge"
                        
                        def mergeResponse = bat(
                            script: "@curl -s -X PUT -H \"Authorization: token ${GITHUB_TOKEN}\" -H \"Accept: application/vnd.github.v3+json\" -d \"{\\\"commit_title\\\":\\\"Merged by Jenkins CI/CD\\\",\\\"merge_method\\\":\\\"squash\\\"}\" ${apiUrl}",
                            returnStdout: true
                        ).trim()
                        
                        echo "Merge response: ${mergeResponse}"
                        
                        def mergeResult = readJSON text: mergeResponse
                        
                        if (mergeResult.merged == true) {
                            env.OUTPUT_MESSAGE += "✅ **PR #${prNum} merged successfully!**\\n"
                            env.OUTPUT_MESSAGE += "Commit SHA: ${mergeResult.sha}\\n"
                        } else {
                            env.OUTPUT_MESSAGE += "⚠️ Merge attempt completed but status unclear\\n"
                            env.OUTPUT_MESSAGE += "Message: ${mergeResult.message ?: 'N/A'}\\n"
                        }
                        
                    } catch (Exception e) {
                        env.OUTPUT_MESSAGE += "❌ Merge error: ${e.message}\\n"
                    }
                    
                    env.OUTPUT_MESSAGE += "\\n---\\n\\n"
                }
            }
        }
        
        // ===== STAGE 8: Archiwizacja =====
        stage('Archive Artifacts') {
            steps {
                script {
                    env.OUTPUT_MESSAGE += "📦 **Archiving artifacts...**\\n\\n"
                    
                    try {
                        // Archiwizuj deployment result
                        if (fileExists('deployment-result.json')) {
                            archiveArtifacts artifacts: 'deployment-result.json', allowEmptyArchive: true
                        }
                        
                        // Archiwizuj kod źródłowy
                        archiveArtifacts artifacts: "${DEPLOY_DIR}/**/*", allowEmptyArchive: true
                        
                        env.OUTPUT_MESSAGE += "✅ Artifacts archived\\n"
                    } catch (Exception e) {
                        env.OUTPUT_MESSAGE += "⚠️ Could not archive: ${e.message}\\n"
                    }
                    
                    env.OUTPUT_MESSAGE += "\\n---\\n\\n"
                }
            }
        }
    }
    
    // ===== POST ACTIONS =====
    post {
        always {
            script {
                echo "========================================"
                echo "Sending final comment to GitHub..."
                echo "========================================"
                
                try {
                    def prNum = env.pr_number ?: env.issue_number
                    
                    if (!prNum || prNum == 'null' || prNum == '') {
                        echo "No PR/Issue number - skipping comment"
                        return
                    }
                    
                    // Przygotuj ostateczną wiadomość
                    def buildStatus = currentBuild.result ?: 'SUCCESS'
                    def statusEmoji = buildStatus == 'SUCCESS' ? '✅' : '❌'
                    
                    def finalMessage = """
## ${statusEmoji} Jenkins CI/CD Pipeline Report

**Build:** [#${BUILD_NUMBER}](${BUILD_URL})
**Status:** ${buildStatus}
**Triggered by:** ${env.pr_action}
**Branch:** ${PR_BRANCH}

---

${env.OUTPUT_MESSAGE}

---

**Summary:**
- Approved: ${env.IS_APPROVED == 'true' ? '✅ Yes' : '⚠️ No'}
- Validated: ${env.IS_VALIDATED == 'true' ? '✅ Yes' : '⚠️ No'}
- Deployed: ${env.IS_DEPLOYED == 'true' ? '✅ Yes' : '❌ No'}

---
*Pipeline executed at: ${new Date()}*
                    """.trim()
                    
                    // Escape dla JSON
                    def escapedMessage = finalMessage
                        .replaceAll('\\\\', '\\\\\\\\')
                        .replaceAll('"', '\\\\"')
                        .replaceAll('\n', '\\\\n')
                        .replaceAll('\r', '')
                    
                    def apiUrl = "https://api.github.com/repos/${REPO_NAME}/issues/${prNum}/comments"
                    
                    bat """
                        @curl -s -X POST -H "Authorization: token ${GITHUB_TOKEN}" -H "Accept: application/vnd.github.v3+json" -d "{\\"body\\":\\"${escapedMessage}\\"}" ${apiUrl}
                    """
                    
                    echo "✅ Comment posted to PR #${prNum}"
                } catch (Exception e) {
                    echo "❌ Failed to post comment: ${e.message}"
                }
            }
        }
        
        success {
            echo "✅ Pipeline completed successfully!"
        }
        
        failure {
            echo "❌ Pipeline failed!"
        }
    }
}
