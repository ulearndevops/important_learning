https://github.com/ulearners
pipeline {
    agent any
    parameters {
        choice(name: 'CLOUD_PROVIDER', choices: ['AWS', 'Azure'], description: 'Select Cloud Provider')
        activeChoiceParam('REGION') {
            description('Select Region based on Cloud Provider')
            groovyScript {
                script('''
                    if (params.CLOUD_PROVIDER == "AWS") {
                        return ["us-east-1", "us-west-2", "eu-central-1"]
                    } else if (params.CLOUD_PROVIDER == "Azure") {
                        return ["East US", "West US", "North Europe"]
                    } else {
                        return ["Default Region"]
                    }
                ''')
                fallbackScript('"Default Region"')
            }
        }
    }
    stages {
        stage('Show Parameters') {
            steps {
                script {
                    echo "Selected Cloud Provider: ${params.CLOUD_PROVIDER}"
                    echo "Selected Region: ${params.REGION}"
                }
            }
        }
    }
}
