def INTEGRATION_FOLDER = "base/integration" 
def INTEGRATION_FILE_PATTERN = "*.T2"

def SYSOUT_FOLDER = "output" 
def SYSOUT_FILE_PATTERN = "*.out" 
def JENKINS_AGENT_LABEL = "agent"

def isFolderExist(folderName) {
  folderExist = fileExists file: "${folderName}"
  return folderExist
}

def loadFiles(filePattern,excludes="") {
    return findFiles(glob: "${filePattern}", excludes: "${excludes}")
}

def listOfFiles
def listOfSysOutFiles
def userInputParams
def reviewYesOrNo = false
def emailAddress 

pipeline {
    agent { label 'SC47-agent' }
    
	parameters {
		string (
			name: 'APPLICATION_LOAD_VALUE',
			defaultValue: '',
			description: 'Enter the application load value')
	}

    stages {

        stage('Git Checkout') {
            steps {
                script {
                    cleanWs()
                    checkout([$class: 'GitSCM',branches: [[name: "main"]],doGenerateSubmoduleConfigurations: false,
                    gitTool: 'Default',
                    submoduleCfg: [],userRemoteConfigs: [[url: "https://github.com/ashokhein/upwork-tasks"]]   ])   
                }
            }
        }

        stage('Load Integration Files') {
            steps {
                script {
                    isIntegrationFolderExist = isFolderExist("${WORKSPACE}/${INTEGRATION_FOLDER}")
                    if(!isIntegrationFolderExist) {
                        println "Integration folder doesn't exist"
                        currentBuild.result = 'FAILURE'
                        error("Job failure to due to integration folder doesn't exist.")
                    }
                    listOfFiles = loadFiles("**/${INTEGRATION_FOLDER}/${INTEGRATION_FILE_PATTERN}")
                    println "List of fileName"
                    listOfFiles.each { String fileMetaData ->
                        println "${fileMetaData.name}"
                    }
                }
            }
        }

        stage('Exec JCL Script') {
            agent {
                label "${JENKINS_AGENT_LABEL}"
            }            
            steps {
                script {
                    failedScript = false 
                    listOfFiles.each { String fileMetaData ->
                        println "Executing ${fileMetaData.name}"
                        def jcl = """
                            HELLO  JOB ,
                            MSGCLASS=H,MSGLEVEL=(1,1),TIME=(,4),REGION=0M
                            STEP0001 EXEC PGM=IEBGENER
                            SYSIN    DD ${WORKSPACE}/${fileMetaData.path}
                            SYSPRINT DD ${APPLICATION_LOAD_VALUE}
                            SYSUT2   DD SYSOUT=*
                        """   
                        try {
                            //new JCLExec().text(jcl).confDir("${WORKSPACE}").execute()
                            // jclExec.saveOutput('SYSPRINT',  new File("${WORKSPACE}/output/${fileMetaData.name}_sysprint.out"), "UTF-8")                                             
                            
                            //For Testing Output
                            writeFile file: "${WORKSPACE}/${SYSOUT_FOLDER}/${fileMetaData.name}_sysprint.out", text: "${jcl}"
                        }catch(err) {
                            failedScript = true
                            println "${err}"
                        }
                   }
                    if(!failedScript) {
                        stash includes: "**/${SYSOUT_FOLDER}/**", name: 'sysoutput'                               
                    }              
                }
            }
        }        

        stage('Print SysOut') {
            steps {
                script {
                    unstash 'sysoutput'
                    isSysOutFolderExist = isFolderExist("${WORKSPACE}/${SYSOUT_FOLDER}")
                    if(!isSysOutFolderExist) {
                        println "Sys out folder doesn't exist"
                        currentBuild.result = 'FAILURE'
                        error("Job failure to due to sys out folder doesn't exist.")                        
                    }
                    listOfSysOutFiles = loadFiles("**/${SYSOUT_FOLDER}/${SYSOUT_FILE_PATTERN}")
                    listOfSysOutFiles.each { String fileMetaData ->
                        def sysOut = readFile file: "${WORKSPACE}/${fileMetaData.path}", encoding: "UTF-8"
                        println sysOut
                    }
                }
            }
        }

        stage('User Review') {
            steps {
                script { 
                    try {
                        timeout(time: 10, unit: 'MINUTES') { // change to a convenient timeout for you
                            userInputParams = input(message: 'Review', 
                            parameters: [
                                booleanParam(defaultValue: false, description: 'Does the output look fine?',name: 'Yes'),
                                string(defaultValue: "", description: 'Email Address',name: 'Email')
                            ])
                        }
                    } catch(err) { // timeout reached or input false
                        println "User input Timeout or Aborted"
                        userInputParams = false
                    }
                }
            }
        }

        stage('Analysing the User Input') {    
            when {
                expression { userInputParams  } 
            }                     
            steps {
                script { 
                    if(userInputParams.Yes != null && userInputParams.Yes != "" && userInputParams.Yes ) {
                        reviewYesOrNo = true
                    }
                    
                    //Validate the Email
                    if(userInputParams.Email != null && userInputParams.Email != "") {
                        emailAddress = userInputParams.Email
                    } else {
                        if(reviewYesOrNo) {
                            currentBuild.result = 'FAILURE'
                            error("Email address is mandatory if you choose output is 'YES'")                                 
                        }
                    }
                }
            }
        }    

        stage('Sending Email') {
            when {
                expression { reviewYesOrNo  } 
            }            
            steps {
                script { 
                    println "Sending Email to ${emailAddress}"
                    // emailext body: 'Test Message', subject: 'Test Subject',to: "${emailAddress}"                    
                }
            }
        }    

        stage('Reverting Git Commit') {
            when {
                expression { !reviewYesOrNo  } 
            }            
            steps {
                script { 
                    println "Reverting GitCommit"
                }
            }
        } 
    }
}
