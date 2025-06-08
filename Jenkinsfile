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
        
        stage('[ZAP] Baseline passive-scan') {
            steps {
                // Utwórz katalog na wyniki
                sh 'mkdir -p results/'
                
                // Uruchomienie kontenera Juice Shop
                sh '''
                    docker run --name juice-shop -d --rm \
                        -p 3000:3000 \
                        bkimminich/juice-shop
                    sleep 5
                '''

                // Uruchomienie kontenera ZAP
                sh '''
                    docker run -d --name zap ghcr.io/zaproxy/zaproxy:stable \
                    sleep 2000
                '''
                
                // Dodanie folderów do kontenera ZAP
                sh '''
                    docker exec zap mkdir -p /zap/wrk/reports
                '''
                
                // Skopiuj plik passive_scan do kontenera ZAP
                sh 'docker cp zap/passive_scan.yaml zap:/zap/wrk/passive_scan.yaml'

                // Wykonaj skan
                sh '''
                    docker exec zap bash -c "
                        zap.sh -cmd -addonupdate;
                        zap.sh -cmd -addoninstall communityScripts;
                        zap.sh -cmd -addoninstall pscanrulesAlpha;
                        zap.sh -cmd -addoninstall pscanrulesBeta;
                        zap.sh -cmd -autorun /zap/wrk/passive_scan.yaml
                        "
                '''
            }
            
            post {
                always {
                    sh '''
                        docker cp zap:/zap/wrk/reports/zap_html_report.html ${WORKSPACE}/results/zap_html_report.html || true
                        docker stop zap || true
                        docker rm zap || true
                        docker stop juice-shop || true
                    '''
                    archiveArtifacts artifacts: 'results/*.html, results/*.xml', allowEmptyArchive: true
                }
            }
        }
        
        stage('OSV Scanner') {
            steps {
                // Utwórz katalog na wyniki
                sh 'mkdir -p results/'

                // Pobierz OSV-Scanner
                sh '''
                    if ! command -v osv-scanner &> /dev/null; then
                        curl -sSL https://github.com/google/osv-scanner/releases/download/v1.0.0/osv-scanner-linux-amd64 -o /usr/local/bin/osv-scanner
                        chmod +x /usr/local/bin/osv-scanner
                    fi
                '''

                // Uruchom skanowanie
                sh '''
                    if [ -f package-lock.json ]; then
                          osv-scanner scan --lockfile package-lock.json > results/osv_scan_report.json || true
                    else
                        echo "package-lock.json not found. Skipping OSV scan."
                    fi
                '''
            }

            post {
                always {
                    archiveArtifacts artifacts: 'results/osv_scan_report.json', allowEmptyArchive: true
                }
            }
        }
    }
}
