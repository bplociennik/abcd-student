pipeline {
    agent any
    options {
        skipDefaultCheckout(true)
    }
    stages {
        stage('Code checkout from GitHub') {
            steps {
                script {
                    cleanWs()
                    git credentialsId: 'github-pat', url: 'https://github.com/bplociennik/abcd-student', branch: 'main'
                }
            }
        }
        stage('Example') {
            steps {
                echo 'Hello!'
                sh 'ls -la'
            }
        }
        
        stage('[ZAP] Baseline passive-scan') {
            steps {
                // Utwórz katalog na wyniki
                sh 'mkdir -p results/'
                
                // Odpalenie aplikacji Juice Shop
                sh '''
                    docker run --name juice-shop -d --rm \
                        -p 3000:3000 \
                        bkimminich/juice-shop
                    sleep 5
                '''

                // Uruchomienie pasywnego skanowania ZAP
                sh '''
                    docker run --name zap \
                        --add-host=host.docker.internal:host-gateway \
                        -v zap/passive_scan.yaml:/zap/wrk/:rw
                        -t ghcr.io/zaproxy/zaproxy:stable bash -c \
                        "zap.sh -cmd -addonupdate; zap.sh -cmd -addoninstall 
                        communityScripts -addoninstall pscanrulesAlpha -addoninstall 
                        pscanrulesBeta -autorun /zap/wrk/passive_scan.yaml" \
                        || true
                '''
            }

            post {
                always {
                    sh '''
                        docker cp zap:/zap/wrk/reports/zap_html_report.html ${WORKSPACE}/results/zap_html_report.html
                        docker cp zap:/zap/wrk/reports/zap_xml_report.xml ${WORKSPACE}/results/zap_xml_report.xml
                        docker stop zap juice-shop
                        docker rm zap
                    '''
                }
            }
        }
    }
}
