pipeline {
    agent any
    environment { 
        GITHUB_URL = '' 
        COMMIT_MESSAGE = '' 
        BRANCH_NAME = ''
        UI_FOLDER = ''
        PROJECT_NAME = ''
        PROJECT_PATH = ''
        PACKAGE_NAME = ''
        PACKAGE_VERSION = ''
        OUTPUT_PATH = "${env.OUTPUT_PATH}"
        UIPCLI_API = credentials('SECRET_KEY_UIPATH')
        UIPAPP_ID = "${env.UIPAPP_ID}"
        ORCHESTRATOR_URL = "${env.ORCHESTRATOR_URL}"
        ORCHESTRATOR_TENANT = "${env.ORCHESTRATOR_TENANT}"
        UIPCLI_PATH = "${env.UIPCLI_PATH}"
        UI_ORGANIZACION = "${env.UI_ORGANIZACION}"
    }
    
    stages {
        stage('Get Commit, Branch, Folder, Project name and Repo URL') {
            steps {
                script {
                    
                    // Obtener la URL del repositorio
                    GITHUB_URL = powershell( script: 'git config --get remote.origin.url', returnStdout: true ).trim()
                    
                    // Obtener el mensaje del último commit
                    COMMIT_MESSAGE = powershell( script: 'git log -1 --pretty=format:"%B"', returnStdout: true ).trim()
                        // Separar el texto por '#'
                    def parts = COMMIT_MESSAGE.split("#", 2) // Limitar a 2 partes
                        // Asignar a variables antes y después del '#'
                    UI_FOLDER = parts[0]
                    PROJECT_NAME = parts.length > 1 ? parts[1] : "${UI_FOLDER}"
                    
                    // Obtener la rama que contiene el commit 
                    def commitHash = powershell( script: 'git rev-parse HEAD', returnStdout: true ).trim() 
                    BRANCH_NAME = powershell( script: "git branch -r --contains ${commitHash}", returnStdout: true ).trim()
                    
                    // Comando de PowerShell para eliminar la carpeta si existe 
                    def deleteFolderCommand = """ if (Test-Path -Path '${OUTPUT_PATH}') { Remove-Item -Path '${OUTPUT_PATH}' -Recurse -Force } """ 
                    // Ejecutar el comando de PowerShell 
                    powershell script: deleteFolderCommand, returnStdout: true
                }
            }
        }
        
        stage('Clone repositorio') {
            steps {
                script {
                    bat """
                    git clone ${GITHUB_URL} ${OUTPUT_PATH}
                    """
                }
            }
        }
        
        stage('Analizar codigo') {
            steps {
                bat """
                "${UIPCLI_PATH}" package analyze "${OUTPUT_PATH}\\${PROJECT_NAME}\\project.json" --analyzerTraceLevel "Verbose" --resultPath "${OUTPUT_PATH}\\Workflow-Analysis.json" --treatWarningsAsErrors
                """
            }
        }


        stage('Version Management') {
            steps {
                script {
                    // Obtiene el número total de commits
                    def commitCount = powershell(
                        script: 'git rev-list --count HEAD',
                        returnStdout: true
                    ).trim()

                    // Construye la nueva versión
                    def newVersion = "1.0.${commitCount}"
                    
                    echo "Nueva versión configurada: ${newVersion}"

                    // Escapa correctamente las comillas y las rutas
                    def updateVersionCommand = """
                    \$projectPath = '${OUTPUT_PATH}\\${PROJECT_NAME}\\project.json'
                    (Get-Content \$projectPath) -replace '"projectVersion": ".*",', '"projectVersion": "${newVersion}",' | Set-Content \$projectPath
                    """

                    // Ejecutar el comando de PowerShell
                    powershell script: updateVersionCommand, returnStdout: true

                    echo "Nueva versión configurada: ${newVersion}"

                    // Asigna la nueva versión como variable de entorno para las siguientes etapas
                    env.PACKAGE_VERSION = newVersion
                }
            }
        }



        stage('Empaquetar proyecto UiPath') {
            steps {
                script {
                    PROJECT_PATH = "${OUTPUT_PATH}\\${PROJECT_NAME}\\project.json"
                    bat """
                    "${UIPCLI_PATH}" package pack "${PROJECT_PATH}" -o "${OUTPUT_PATH}"
                    """
                    def directoryPath = "${OUTPUT_PATH}"  // Cambia esto por la ruta del directorio donde esperas el archivo
                    def fileExists = false
                    def maxAttempts = 10  // Número máximo de intentos
                    def attempt = 0
                    def waitTime = 10  // Tiempo de espera entre intentos en segundos

                    while (!fileExists && attempt < maxAttempts) {
                        def files = powershell(
                            script: "Get-ChildItem -Path '${directoryPath}' -Filter '*.nupkg'",
                            returnStdout: true
                        ).trim()

                        if (files) {
                            fileExists = true
                            echo "Archivo .nupkg encontrado: ${files}"
                        } else {
                            attempt++
                            echo "Intento ${attempt}: Esperando ${waitTime} segundos..."
                            sleep(time: waitTime, unit: 'SECONDS')
                        }
                    }
                    PACKAGE_NAME = powershell( script: "Get-ChildItem -Path ${OUTPUT_PATH} -Filter '*.nupkg' | Select-Object -ExpandProperty Name", returnStdout: true ).trim()
                    echo "nombre ${PACKAGE_NAME}"
            }
        }
    }


        stage('Desplegar en Orchestrator') {
            steps {
                bat """
                "${UIPCLI_PATH}" package deploy "${OUTPUT_PATH}\\${PACKAGE_NAME}" "${ORCHESTRATOR_URL}" "${ORCHESTRATOR_TENANT}" --applicationId "${UIPAPP_ID}" --applicationSecret "${UIPCLI_API}" --applicationScope "OR.Assets OR.BackgroundTasks OR.Execution OR.Folders OR.Jobs OR.Machines.Read OR.Monitoring OR.Robots.Read OR.Settings.Read OR.TestSetExecutions OR.TestSets OR.TestSetSchedules OR.Users.Read" -o "${UI_FOLDER}" --accountForApp "${UI_ORGANIZACION}"
                """
            }
        }
        
        stage('Print Variables') { 
            steps { 
                echo "Repositorio URL: ${GITHUB_URL}" 
                echo "Último mensaje de commit: ${COMMIT_MESSAGE}" 
                echo "Nombre de la rama actual: ${BRANCH_NAME}" 
                echo "Nombre de la rama actual: ${UI_FOLDER}" 
                echo "Nombre de la rama actual: ${PROJECT_NAME}" 
            } 
        }
    }
    post {

        success {

            echo '¡Despliegue exitoso !'

        }

        failure {

            echo 'Hubo un error en el despliegue.'

        }

    }

}
