/*
 * =====================================================
 * SALESFORCE CI/CD PIPELINE - JENKINSFILE
 * =====================================================
 * 
 * Purpose: Automated validation, approval, and deployment pipeline for Salesforce metadata
 * Trigger: GitHub webhooks (PR comments, PR events)
 * Author: Created for educational/production use
 * Last Updated: 2026-01-26
 * 
 * Workflow:
 * 1. Filter webhook triggers (ignore Jenkins bot comments)
 * 2. Check dependencies (Git, SF CLI)
 * 3. Clone repository and merge branches
 * 4. Authenticate to Salesforce using JWT
 * 5. Check PR approvals
 * 6. Validate or Deploy based on comment command
 * 7. Parse deployment results
 * 8. Merge PR (if approved and deployed)
 * 9. Archive artifacts
 * 10. Post results to GitHub PR
 */

pipeline {
    agent any  // Run on any available Jenkins agent
    
    // ===== ENVIRONMENT VARIABLES =====
    // Global variables accessible throughout the entire pipeline
    environment {
        // GitHub Personal Access Token for API calls
        GITHUB_TOKEN = credentials('github-token')
        
        // Add Salesforce CLI to system PATH for command execution
        PATH = "C:\\Program Files\\sf\\client\\bin;${env.PATH}"
        
        // === Webhook Variables ===
        // These are populated by Generic Webhook Trigger plugin from GitHub payload
        PR_BRANCH = "${env.pr_branch ?: 'SIT'}"           // Target branch (default: SIT)
        PR_NUMBER = "${env.pr_number ?: env.issue_number}" // Pull Request number
        PR_ACTION = "${env.pr_action}"                     // Webhook action (opened, created, etc.)
        COMMENT_BODY = "${env.comment_body}"               // Comment text from PR
        SOURCE_BRANCH = "${env.pr_head_branch}"            // Source branch (feature branch)
        REPO_NAME = "${env.repo_name}"                     // Repository full name (owner/repo)
        
        // === Salesforce Configuration ===
        SF_ALIAS = "SIT"                                   // Salesforce org alias
        DEPLOY_DIR = "force-app/main/default"              // Directory containing Salesforce metadata
        
        // === Pipeline Status Flags ===
        // Track whether each stage completed successfully
        IS_APPROVED = 'false'   // PR has required approvals
        IS_DEPLOYED = 'false'   // Deployment completed
        IS_VALIDATED = 'false'  // Validation completed
        
    }
    
    stages {
        
        // =====================================================
        // STAGE 0: FILTER TRIGGERS
        // =====================================================
        // Purpose: 
        // - Prevent infinite loops by ignoring Jenkins bot comments
        // - Only allow specific webhook actions (opened, created, etc.)
        // - Log incoming webhook data for debugging
        // =====================================================
        stage('Filter Triggers') {
            steps {
                script {
                    env.OUTPUT_MESSAGE = "" 

                    // === Log Webhook Information ===
                    echo "========================================"
                    echo "Webhook received!"
                    echo "Action: ${env.pr_action}"
                    echo "PR Number: ${env.pr_number ?: env.issue_number}"
                    echo "Comment: ${env.comment_body}"
                    echo "Comment Author: ${env.comment_author}"
                    echo "========================================"

                    // === Debug Comment Content ===
                    // Check if comment contains keywords to detect Jenkins bot
                    echo "DEBUG: comment_body length = ${env.comment_body?.length()}"
                    echo "DEBUG: contains 'Pipeline Report'? = ${env.comment_body?.contains('Pipeline Report')}"
                    echo "DEBUG: contains 'Jenkins'? = ${env.comment_body?.contains('Jenkins')}"
                    
                    // === Prevent Infinite Loop ===
                    // Jenkins posts a comment → GitHub triggers webhook → Jenkins posts again → LOOP!
                    // Solution: Detect Jenkins-generated comments and abort
                    if (env.comment_body?.contains('Pipeline Report')) {
                        echo "Skipping - this is a Jenkins bot comment"
                        currentBuild.result = 'ABORTED'
                        error('Jenkins bot comment detected - aborting to prevent infinite loop')
                    }
                    
                    // === Filter Webhook Actions ===
                    // Only process specific GitHub webhook actions
                    def allowedActions = ['opened', 'synchronize', 'created', 'submitted']
                    
                    if (!(env.pr_action in allowedActions)) {
                        echo "Skipping pipeline - action '${env.pr_action}' not in allowed list"
                        currentBuild.result = 'NOT_BUILT'
                        error("Action not in allowed list")
                    }
                    
                    echo "Action '${env.pr_action}' is allowed - continuing pipeline"
                }
            }
        }
        
        // =====================================================
        // STAGE 1: CHECK DEPENDENCIES
        // =====================================================
        // Purpose:
        // - Verify Git is installed and accessible
        // - Verify Salesforce CLI is installed and accessible
        // - Log versions for troubleshooting
        // =====================================================
        stage('Check Dependencies') {
            steps {
                script {
                    env.OUTPUT_MESSAGE += "**Checking dependencies...**\n\n"
                    
                    // === Check Git Installation ===
                    try {
                        echo "Checking Git..."
                        // @ prefix suppresses command echo in Windows batch
                        def gitVersion = bat(script: '@git --version', returnStdout: true).trim()
                        echo "Git installed: ${gitVersion}"
                        env.OUTPUT_MESSAGE += "Git: ${gitVersion}\n"
                    } catch (Exception gitError) {
                        echo "Git not found!"
                        env.OUTPUT_MESSAGE += "Git not found\n"
                        error("Git is required")  // Stop pipeline if Git not found
                    }
                    
                    // === Check Salesforce CLI Installation ===
                    try {
                        echo "Checking Salesforce CLI..."
                        def sfVersion = bat(script: '@sf --version', returnStdout: true).trim()
                        echo "Salesforce CLI found: ${sfVersion}"
                        env.OUTPUT_MESSAGE += "Salesforce CLI: ${sfVersion}\n"
                    } catch (Exception sfError) {
                        echo "SF CLI not found"
                        env.OUTPUT_MESSAGE += "SF CLI not found\n"
                        error("Salesforce CLI is required")  // Stop pipeline if SF CLI not found
                    }
                    
                    // Add separator for GitHub comment formatting
                    env.OUTPUT_MESSAGE += "\n---\n\n"
                }
            }
        }
        
        // =====================================================
        // STAGE 2: CLONE AND MERGE
        // =====================================================
        // Purpose:
        // - Clean workspace to ensure fresh start
        // - Clone repository from GitHub
        // - Checkout target branch (e.g., SIT)
        // - Merge source branch if PR event (not comment)
        // =====================================================
        stage('Clone and Merge') {
            steps {
                script {
                    env.OUTPUT_MESSAGE += "**Cloning repository and merging branches...**\n\n"
                    
                    try {
                        // === Clean Workspace ===
                        // Remove all files from previous builds to avoid conflicts
                        echo "Cleaning workspace..."
                        deleteDir()
                        
                        // === Clone Repository ===
                        // Use GitHub token for authentication
                        echo "Cloning repository: ${REPO_NAME}"
                        bat """
                            git clone https://${GITHUB_TOKEN}@github.com/${REPO_NAME}.git .
                        """
                        env.OUTPUT_MESSAGE += "Repository cloned\n"
                        
                        // === Checkout Target Branch ===
                        // Switch to the branch where changes will be deployed (e.g., SIT)
                        echo "Checking out branch: ${PR_BRANCH}"
                        bat """
                            git checkout ${PR_BRANCH}
                        """
                        env.OUTPUT_MESSAGE += "Checked out to: ${PR_BRANCH}\n"
                        
                        // === Merge Source Branch (if exists) ===
                        // For PR events, merge feature branch into target branch
                        // For comment triggers, SOURCE_BRANCH is null
                        if (SOURCE_BRANCH && SOURCE_BRANCH != '' && SOURCE_BRANCH != 'null') {
                            echo "Merging ${SOURCE_BRANCH} into ${PR_BRANCH}"
                            // --no-commit: Don't create merge commit yet
                            // --no-ff: Always create merge commit (no fast-forward)
                            def mergeResult = bat(
                                script: "git merge origin/${SOURCE_BRANCH} --no-commit --no-ff",
                                returnStatus: true  // Return exit code instead of throwing error
                            )
                            
                            // Check if merge was successful (exit code 0)
                            if (mergeResult != 0) {
                                env.OUTPUT_MESSAGE += "**MERGE CONFLICT**\n"
                                error("Merge conflict detected")  // Stop pipeline on conflict
                            } else {
                                env.OUTPUT_MESSAGE += "Merged successfully: ${SOURCE_BRANCH} → ${PR_BRANCH}\n"
                            }
                        } else {
                            // Comment-triggered builds don't have a source branch to merge
                            echo "No source branch to merge (comment trigger)"
                            env.OUTPUT_MESSAGE += "No branch merge needed (comment trigger)\n"
                        }
                        
                    } catch (Exception e) {
                        env.OUTPUT_MESSAGE += "Error during clone/merge: ${e.message}\n"
                        throw e  // Re-throw to fail the build
                    }
                    
                    env.OUTPUT_MESSAGE += "\n---\n\n"
                }
            }
        }
        
        // =====================================================
        // STAGE 3: AUTHENTICATE TO SALESFORCE
        // =====================================================
        // Purpose:
        // - Authenticate to Salesforce org using JWT flow
        // - Use server key, client ID, and username from Jenkins credentials
        // - Set authenticated org as default for subsequent commands
        // =====================================================
        stage('Authenticate to Salesforce') {
            steps {
                script {
                    env.OUTPUT_MESSAGE += "**Authenticating to Salesforce...**\n\n"
                    
                    try {
                        // === Load Credentials from Jenkins ===
                        // - salesforce-jwt-key: Private key file (server.key)
                        // - salesforce-client-id: Connected App Consumer Key
                        // - salesforce-username: Salesforce user email
                        withCredentials([
                            file(credentialsId: 'salesforce-jwt-key', variable: 'SF_SERVER_KEY'),
                            string(credentialsId: 'salesforce-client-id', variable: 'SF_CLIENT_ID'),
                            string(credentialsId: 'salesforce-username', variable: 'SF_USERNAME')
                        ]) {
                            echo "Authenticating with JWT..."
                            
                            // === Execute JWT Login ===
                            // JWT flow provides long-lived authentication without passwords
                            // ^ is Windows batch line continuation character
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
                                returnStatus: true  // Return exit code (0 = success)
                            )
                            
                            // === Verify Authentication ===
                            if (authResult == 0) {
                                env.OUTPUT_MESSAGE += "**Authenticated successfully to Salesforce**\n"
                                
                                // Get org info for verification
                                def orgInfo = bat(script: "@sf org display --json", returnStdout: true).trim()
                                def orgData = readJSON text: orgInfo
                                
                                if (orgData.status == 0) {
                                    def orgId = orgData.result?.id ?: 'Unknown'
                                    def username = orgData.result?.username ?: 'Unknown'
                                    env.OUTPUT_MESSAGE += "- Org ID: ${orgId}\n"
                                    env.OUTPUT_MESSAGE += "- Username: ${username}\n"
                                } else {
                                    env.OUTPUT_MESSAGE += "- Org ID: Could not retrieve\n"
                                }
                            } else {
                                env.OUTPUT_MESSAGE += "**Authentication failed**\n"
                                error("Salesforce JWT authentication failed")
                            }
                        }
                        
                    } catch (Exception e) {
                        env.OUTPUT_MESSAGE += "Authentication error: ${e.message}\n"
                        env.OUTPUT_MESSAGE += "\n**Troubleshooting:**\n"
                        env.OUTPUT_MESSAGE += "- Verify Connected App settings\n"
                        env.OUTPUT_MESSAGE += "- Check JWT key file\n"
                        env.OUTPUT_MESSAGE += "- Confirm username is correct\n"
                        throw e
                    }
                    
                    env.OUTPUT_MESSAGE += "\n---\n\n"
                }
            }
        }
        
        // =====================================================
        // STAGE 4: CHECK APPROVALS
        // =====================================================
        // Purpose:
        // - Query GitHub API for PR approvals
        // - Count number of approving reviews
        // - Set IS_APPROVED flag for later use in merge stage
        // =====================================================
        stage('Check Approvals') {
            steps {
                script {
                    env.OUTPUT_MESSAGE += "**Checking PR approvals...**\n\n"
                    
                    try {
                        def prNum = env.pr_number ?: env.issue_number
                        echo "Checking approvals for PR #${prNum}"
                        
                        // === Query GitHub Reviews API ===
                        // Get all reviews for this PR
                        def apiUrl = "https://api.github.com/repos/${REPO_NAME}/pulls/${prNum}/reviews"
                        def reviewsJson = bat(
                            script: "@curl -s -H \"Authorization: token ${GITHUB_TOKEN}\" ${apiUrl}",
                            returnStdout: true
                        ).trim()
                        
                        // === Parse Reviews ===
                        def reviews = readJSON text: reviewsJson
                        
                        // Count approved reviews
                        def approvalCount = reviews.findAll { it.state == 'APPROVED' }.size()
                        
                        // === Update Status ===
                        if (approvalCount > 0) {
                            env.IS_APPROVED = 'true'
                            env.OUTPUT_MESSAGE += "**${approvalCount} approval(s) found**\n"
                        } else {
                            env.OUTPUT_MESSAGE += "**No approvals found**\n"
                        }
                        
                    } catch (Exception e) {
                        env.OUTPUT_MESSAGE += "Could not check approvals: ${e.message}\n"
                    }
                    
                    env.OUTPUT_MESSAGE += "\n---\n\n"
                }
            }
        }
        
        // =====================================================
        // STAGE 5: VALIDATE OR DEPLOY
        // =====================================================
        // Purpose:
        // - Check comment for "Validate" or "Deploy" keywords
        // - Execute Salesforce validation (check-only deployment)
        // - Execute Salesforce deployment (actual changes to org)
        // - Set appropriate status flag for reporting
        // =====================================================
        stage('Validate or Deploy') {
            steps {
                script {
                    // === Debug Comment Detection ===
                    echo "DEBUG: params.comment_body = ${params.comment_body}"
                    echo "DEBUG: env.comment_body = ${env.comment_body}"
                    
                    // Use env.comment_body (from webhook) or fallback to params
                    def commentText = env.comment_body ?: params.comment_body
                    echo "Checking comment: ${commentText}"
                    
                    // === VALIDATE Command ===
                    // Triggered by comment containing "validate" (case-insensitive)
                    if (commentText?.toLowerCase()?.contains('validate')) {
                        env.OUTPUT_MESSAGE += "**Running VALIDATION...**\n\n"
                        echo "Running VALIDATION"
                        
                        // Execute validation deployment (no changes applied)
                        bat(script: """
                            sf project deploy validate ^
                                --source-dir force-app ^
                                --target-org SIT ^
                                --json > deployment-result.json
                        """, returnStatus: true)

                        // === Set Validation Flag ===
                        // Use currentBuild.description instead of env variable
                        // (env variables have issues in Declarative Pipeline)
                        echo "SETTING IS_VALIDATED via currentBuild.description"
                        currentBuild.description = (currentBuild.description ?: '') + 'VALIDATED '
                        echo "currentBuild.description is now: ${currentBuild.description}"
                        
                        env.OUTPUT_MESSAGE += "**Validation attempted (check details in artifacts)**\n"
                        
                    // === DEPLOY Command ===
                    // Triggered by comment containing "deploy" (case-insensitive)
                    } else if (commentText?.toLowerCase()?.contains('deploy')) {
                        env.OUTPUT_MESSAGE += "**Running DEPLOYMENT...**\n\n"
                        echo "Running DEPLOYMENT"
                        
                        // Execute actual deployment (changes applied to org)
                        bat(script: """
                            sf project deploy start ^
                                --source-dir force-app ^
                                --target-org SIT ^
                                --test-level RunLocalTests ^
                                --json > deployment-result.json
                        """, returnStatus: true)
                        
                        // === Set Deployment Flag ===
                        currentBuild.description = (currentBuild.description ?: '') + 'DEPLOYED '
                        env.OUTPUT_MESSAGE += "**Deployment attempted (check details in artifacts)**\n"
                        
                    } else {
                        // No recognized command in comment
                        echo "No valid command - skipping validation/deployment"
                        env.OUTPUT_MESSAGE += "No validate/deploy command detected\n"
                    }

                    env.OUTPUT_MESSAGE += "\n---\n\n" 

                }
            }
        }
        
        // =====================================================
        // STAGE 6: PARSE DEPLOYMENT RESULT
        // =====================================================
        // Purpose:
        // - Read deployment-result.json file
        // - Extract deployment ID, component counts, test results
        // - Display errors if deployment failed
        // 
        // NOTE: Currently disabled for testing purposes
        //       (force-app contains only org settings, not deployable code)
        // =====================================================
        stage('Parse Deployment Result') {
            steps {
                script {
                    // === DISABLED STAGE ===
                    // This stage is temporarily disabled because the force-app directory
                    // contains organization settings that cannot be deployed between orgs
                    // In a real project, this would parse actual deployment results
                    echo "Stage temporarily disabled due to missing code in force-app directory."
                    
                    /* ORIGINAL CODE (commented out):
                    
                    // Check if deployment result file exists
                    if (fileExists('deployment-result.json')) {
                        def deployResult = readJSON file: 'deployment-result.json'
                        
                        echo "Deployment result status: ${deployResult.status}"
                        
                        // Parse successful deployment
                        if (deployResult.status == 0) {
                            def result = deployResult.result
                            
                            // Determine if this was validation or deployment
                            if (result.checkOnly == true) {
                                env.IS_VALIDATED = 'true'
                                env.OUTPUT_MESSAGE += "\n**Validation Details:**\n"
                            } else {
                                env.IS_DEPLOYED = 'true'
                                env.OUTPUT_MESSAGE += "\n**Deployment Details:**\n"
                            }
                            
                            // Display deployment statistics
                            env.OUTPUT_MESSAGE += "- Deploy ID: `${result.id}`\n"
                            env.OUTPUT_MESSAGE += "- Components: ${result.numberComponentsDeployed ?: 0} deployed, ${result.numberComponentErrors ?: 0} errors\n"
                            env.OUTPUT_MESSAGE += "- Tests: ${result.numberTestsCompleted ?: 0} run, ${result.numberTestErrors ?: 0} failures\n"
                            
                            // Display component failures if any
                            if (result.details?.componentFailures) {
                                env.OUTPUT_MESSAGE += "\n**Errors:**\n"
                                result.details.componentFailures.each { failure ->
                                    env.OUTPUT_MESSAGE += "- ${failure.fileName}: ${failure.problem}\n"
                                }
                            }
                        } else {
                            env.OUTPUT_MESSAGE += "\n**Validation/Deployment failed** - check logs\n"
                        }
                    }
                    */
                }
            }
        }
        
        // =====================================================
        // STAGE 7: MERGE PULL REQUEST
        // =====================================================
        // Purpose:
        // - Automatically merge PR if deployment succeeded and PR is approved
        // - Use GitHub API to squash and merge
        // - Only runs when both IS_DEPLOYED and IS_APPROVED are true
        // =====================================================
        stage('Merge Pull Request') {
            // === Conditional Execution ===
            // Only execute this stage if:
            // 1. Deployment was successful (IS_DEPLOYED == 'true')
            // 2. PR has required approvals (IS_APPROVED == 'true')
            when {
                expression { 
                    return env.IS_DEPLOYED == 'true' && env.IS_APPROVED == 'true' 
                }
            }
            steps {
                script {
                    env.OUTPUT_MESSAGE += "**Merging Pull Request...**\n\n"
                    
                    try {
                        def prNum = env.pr_number ?: env.issue_number
                        
                        // Verify PR number exists
                        if (!prNum || prNum == 'null') {
                            env.OUTPUT_MESSAGE += "No PR number - cannot merge\n"
                            env.OUTPUT_MESSAGE += "\n---\n\n"
                            return
                        }
                        
                        echo "Merging PR #${prNum}"
                        
                        // === Call GitHub Merge API ===
                        // PUT /repos/{owner}/{repo}/pulls/{pull_number}/merge
                        def apiUrl = "https://api.github.com/repos/${REPO_NAME}/pulls/${prNum}/merge"
                        
                        // Execute merge with squash method
                        // Squash combines all commits into one
                        def mergeResponse = bat(
                            script: "@curl -s -X PUT -H \"Authorization: token ${GITHUB_TOKEN}\" -H \"Accept: application/vnd.github.v3+json\" -d \"{\\\"commit_title\\\":\\\"Merged by Jenkins CI/CD\\\",\\\"merge_method\\\":\\\"squash\\\"}\" ${apiUrl}",
                            returnStdout: true
                        ).trim()
                        
                        echo "Merge response: ${mergeResponse}"
                        
                        // === Parse Merge Result ===
                        def mergeResult = readJSON text: mergeResponse
                        
                        if (mergeResult.merged == true) {
                            env.OUTPUT_MESSAGE += "**PR #${prNum} merged successfully!**\n"
                            env.OUTPUT_MESSAGE += "Commit SHA: ${mergeResult.sha}\n"
                        } else {
                            env.OUTPUT_MESSAGE += "Merge attempt completed but status unclear\n"
                            env.OUTPUT_MESSAGE += "Message: ${mergeResult.message ?: 'N/A'}\n"
                        }
                        
                    } catch (Exception e) {
                        env.OUTPUT_MESSAGE += "Merge error: ${e.message}\n"
                    }
                    
                    env.OUTPUT_MESSAGE += "\n---\n\n"
                }
            }
        }
        
        // =====================================================
        // STAGE 8: ARCHIVE ARTIFACTS
        // =====================================================
        // Purpose:
        // - Save deployment results for later review
        // - Archive source code that was deployed
        // - Make artifacts downloadable from Jenkins UI
        // =====================================================
        stage('Archive Artifacts') {
            steps {
                script {
                    env.OUTPUT_MESSAGE += "**Archiving artifacts...**\n\n"
                    
                    try {
                        // === Archive Deployment Result JSON ===
                        // Contains detailed info about validation/deployment
                        if (fileExists('deployment-result.json')) {
                            archiveArtifacts artifacts: 'deployment-result.json', allowEmptyArchive: true
                        }
                        
                        // === Archive Salesforce Metadata ===
                        // Save the code that was deployed for traceability
                        archiveArtifacts artifacts: "${DEPLOY_DIR}/**/*", allowEmptyArchive: true
                        
                        env.OUTPUT_MESSAGE += "Artifacts archived\n"
                    } catch (Exception e) {
                        env.OUTPUT_MESSAGE += "Could not archive: ${e.message}\n"
                    }
                    
                    env.OUTPUT_MESSAGE += "\n---\n\n"
                }
            }
        }
    }
    
        // ===== POST ACTIONS =====
// Purpose: Send pipeline execution report as comment to GitHub PR
// Executes: Always (regardless of build success/failure)
post {
    always {
        script {

            // ===== DEBUG OUTPUT =====
            // Log all relevant variables for troubleshooting (Jenkins console output only)
            echo "============================================"
            echo "POST ACTIONS - DEBUG:"
            echo "comment_body = '${env.comment_body}'"
            echo "contains 'Pipeline Report'? = ${env.comment_body?.contains('Pipeline Report')}"
            echo "currentBuild.result = '${currentBuild.result}'"
            echo "currentBuild.description = '${currentBuild.description}'"
            echo "contains VALIDATED? = ${currentBuild.description?.contains('VALIDATED')}"
            echo "contains DEPLOYED? = ${currentBuild.description?.contains('DEPLOYED')}"
            echo "============================================"

            // ===== CONDITION 1: Was this build triggered by Jenkins bot comment? =====
            // Check if the WEBHOOK was triggered by a comment containing "Pipeline Report"
            // This prevents infinite loop: Build → Comment → Webhook → Build → Comment → ∞
            // Note: check the TRIGGER (comment_body), not the OUTPUT of this build
            def triggeredByBotComment = env.comment_body?.contains('Pipeline Report')
            
            if (triggeredByBotComment) {
                echo "Skipping - this is a Jenkins bot comment"
                echo "Build result: ${currentBuild.result}"
                
                // If build aborted in Stage 0 (detected bot) - DON'T post comment
                // This is the expected behavior - Stage 0 aborts, Post action skips comment
                if (currentBuild.result == 'ABORTED') {
                    echo "Build correctly aborted to prevent loop - no comment needed"
                    return
                }
            }

            // ===== CONDITION 2: Did any stages execute? =====
            // Check if pipeline executed any stages (description is set in Stage 5)
            // If build aborted before stages ran, skip comment (nothing to report)
            def stagesExecuted = currentBuild.description && currentBuild.description != 'null' && currentBuild.description != ''
            
            if (!stagesExecuted && currentBuild.result == 'ABORTED') {
                echo "Build aborted without executing stages - skipping comment"
                return
            }

            // ===== CONDITION 3: Is PR number available? =====
            // Get PR/Issue number from webhook (required for posting comment)
            def prNum = env.pr_number ?: env.issue_number
            
            if (!prNum || prNum == 'null' || prNum == '') {
                echo "No PR/Issue number - cannot post comment"
                return
            }

            // ===== ALL CHECKS PASSED - POST COMMENT TO GITHUB =====
            echo "============================================"
            echo "All checks passed - posting comment to GitHub"
            echo "============================================"
            
            try {
                // ===== PREPARE REPORT =====
                // Determine build status (SUCCESS/FAILURE/ABORTED)
                def buildStatus = currentBuild.result ?: 'SUCCESS'

                // ===== FIX: Handle empty OUTPUT_MESSAGE =====
                // If OUTPUT_MESSAGE is null/empty (early abort), use placeholder
                // This prevents "null" appearing in GitHub comment
                def outputContent = env.OUTPUT_MESSAGE ?: "No stage output (build may have been aborted early)"

                // ===== BUILD MARKDOWN REPORT AS SINGLE LINE =====
                // CRITICAL FIX: Composed with string concatenation (+ "\n") instead of multi-line """..."""
                // Why? Multi-line """...""" creates REAL newlines that break Windows CMD command parsing
                // Here we create ONE LONG STRING with LITERAL \n characters inside
                // GitHub API will interpret \n as actual newlines when rendering markdown
                def finalMessage =
                    "## Jenkins CI/CD Pipeline Report\n\n" +  // Title + spacing
                    "**Build:** [#${BUILD_NUMBER}](${BUILD_URL})\n" +  // Build link
                    "**Status:** ${buildStatus}\n" +  // SUCCESS/FAILURE/ABORTED
                    "**Triggered by:** ${env.pr_action}\n" +  // Webhook action (created, opened, etc.)
                    "**Branch:** ${PR_BRANCH}\n\n" +  // Target branch (SIT)
                    "---\n\n" +  // Separator before stages output
                    "${outputContent}\n\n" +  // ALL stage logs (Checking dependencies..., etc.)
                    "---\n\n" +  // Separator before summary
                    "**Summary:**\n" +  // Summary section
                    "- Approved: ${env.IS_APPROVED == 'true' ? 'Yes' : 'No'}\n" +
                    "- Validated: ${currentBuild.description?.contains('VALIDATED') ? 'Yes' : 'No'}\n" +
                    "- Deployed: ${currentBuild.description?.contains('DEPLOYED') ? 'Yes' : 'No'}\n\n" +
                    "---\n" +  // Final separator
                    "*Pipeline executed at: ${new Date()}*"  // Timestamp

                // ===== ESCAPE MESSAGE FOR JSON =====
                // ONLY escape double quotes (") for valid JSON
                // ONLY remove Windows carriage returns (\r)
                // LET GitHub API interpret literal \n as actual newlines - DO NOT convert them!
                def escapedMessage = finalMessage
                    .replaceAll('"', '\\\\"')  // " → \" (JSON escape)
                    .replaceAll('\r', '')     // Remove \r (Windows line endings)

                // ===== POST COMMENT VIA GITHUB API =====
                // POST /repos/{owner}/{repo}/issues/{issue_number}/comments
                // Note: PR comments use /issues/ endpoint (not /pulls/)
                def apiUrl = "https://api.github.com/repos/${REPO_NAME}/issues/${prNum}/comments"
                
                // ===== WINDOWS CMD FRIENDLY CURL =====
                // @curl = suppress command echo
                // ^ at end of lines = Windows CMD line continuation (treats as ONE command)
                // -d "..." = JSON payload (one long line, no real newlines inside)
                bat """
                    @curl -s -X POST ^
                      -H "Authorization: token ${GITHUB_TOKEN}" ^
                      -H "Accept: application/vnd.github.v3+json" ^
                      -d "{\\"body\\":\\"${escapedMessage}\\"}" ^
                      ${apiUrl}
                """
                
                echo "Comment posted to PR #${prNum}"
            } catch (Exception e) {
                echo "Failed to post comment: ${e.message}"
            }
        }
    }
    
    // ===== SUCCESS BLOCK =====
    // Executes ONLY if pipeline completed successfully
    success {
        echo "Pipeline completed successfully!"
    }
    
    // ===== FAILURE BLOCK =====
    // Executes ONLY if pipeline failed (error() in any stage)
    failure {
        echo "Pipeline failed!"
    }
}