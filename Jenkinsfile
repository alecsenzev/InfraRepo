pipeline {
    agent { label params.AGENT_LABEL }

    options {
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
    }

    parameters {
        string(name: 'AGENT_LABEL', defaultValue: 'Nikita_Alecsentsev', description: 'Jenkins agent label')
        choice(name: 'DASH_MODE', choices: ['test', 'prod'], description: 'Which Dash UI to run')
        string(name: 'DASH_PORT', defaultValue: '8051', description: 'Dash port (test=8050, prod=8051)')
    }

    environment {
        STACK_NAME = 'Nikita-stack'
        HEAT_TEMPLATE = 'server.yaml'
        KEY_PATH = '/home/ubuntu/Nikita_Alecsentsev.pem'
        REMOTE_USER = 'debian'
        APP_REPO = 'https://github.com/alecsenzev/RestoringValuesSPbPU.git'
    }

    stages {
        stage('Checkout InfraRepo') {
            steps {
                git branch: 'main', url: 'https://github.com/alecsenzev/InfraRepo.git'
            }
        }

        stage('Create Infrastructure via Heat') {
            steps {
                script {
                    sh '''
                        . /home/ubuntu/openrc.sh
                        # Удалить предыдущий стек, если существует
                        openstack stack show $STACK_NAME && openstack stack delete $STACK_NAME --yes --wait || true
                        # Создать новый стек
                        openstack stack create -t $HEAT_TEMPLATE --wait $STACK_NAME
                    '''
                }
            }
        }

        stage('Get Target Server IP') {
            steps {
                script {
                    // Получаем IP созданного сервера (ищем по части имени)
                    env.TARGET_IP = sh(
                        script: """
                            . /home/ubuntu/openrc.sh
                            openstack server list --name ".*${STACK_NAME}.*" -f value -c Networks | head -1 | awk -F'=' '{print \$2}'
                        """,
                        returnStdout: true
                    ).trim()
                    
                    if (env.TARGET_IP.isEmpty()) {
                        // Если не нашли, пробуем получить через stack show
                        echo "Поиск по имени не дал результатов, пробуем альтернативный метод..."
                        env.TARGET_IP = sh(
                            script: """
                                . /home/ubuntu/openrc.sh
                                openstack stack show $STACK_NAME -f value -c outputs | grep -o '"ip":[^,]*' | cut -d'"' -f4 || true
                            """,
                            returnStdout: true
                        ).trim()
                    }
                    
                    if (env.TARGET_IP.isEmpty()) {
                        error "Не удалось получить IP адрес сервера"
                    }
                    echo "Target server IP: ${TARGET_IP}"
                }
            }
        }

        stage('Deploy Application on Target Server') {
            steps {
                script {
                    sh """
                        ssh -o StrictHostKeyChecking=no -i $KEY_PATH ${REMOTE_USER}@${TARGET_IP} '
                            set -e
                            echo "=== Клонирование репозитория ==="
                            git clone $APP_REPO ~/app || (cd ~/app && git pull)
                            cd ~/app
                            
                            echo "=== Создание виртуального окружения ==="
                            python3 -m venv .venv
                            . .venv/bin/activate
                            
                            echo "=== Установка зависимостей ==="
                            pip install -U pip
                            pip install -r requirements.txt
                        '
                    """
                }
            }
        }

        stage('Run Application & Health Checks') {
            steps {
                sh """
                    ssh -i $KEY_PATH ${REMOTE_USER}@${TARGET_IP} '
                        cd ~/app
                        . .venv/bin/activate
                        
                        # Очистка переменных окружения
                        export LOG_LEVEL=INFO
                        export WEBSOCKET_HOST=127.0.0.1
                        export BUSINESS_HTTP_BASE=http://127.0.0.1:8000
                        export BUSINESS_BIND_HOST=127.0.0.1
                        export BUSINESS_BIND_PORT=8000
                        export DASH_HOST=0.0.0.0
                        export DASH_PORT=${DASH_PORT}

                        echo "=== Очистка занятых портов ==="
                        for p in 8000 8050 8051 8092 8093 8094 8095; do
                            sudo fuser -k \${p}/tcp 2>/dev/null || true
                        done

                        mkdir -p run_output

                        echo "=== Запуск компонентов ==="
                        # Запуск симулятора
                        python Simulator/simulator.py > run_output/simulator.log 2>&1 &
                        echo \$! > run_output/simulator.pid
                        echo "Симулятор запущен с PID: \$(cat run_output/simulator.pid)"
                        sleep 3

                        # Запуск receiver
                        python Reciever/reciever.py > run_output/reciever.log 2>&1 &
                        echo \$! > run_output/reciever.pid
                        echo "Receiver запущен с PID: \$(cat run_output/reciever.pid)"
                        sleep 3

                        # Запуск business
                        python Business/business.py > run_output/business.log 2>&1 &
                        echo \$! > run_output/business.pid
                        echo "Business запущен с PID: \$(cat run_output/business.pid)"
                        sleep 3

                        # Запуск Dash
                        if [ "${DASH_MODE}" = "test" ]; then
                            python GUI/dash_app_test.py > run_output/dash.log 2>&1 &
                        else
                            python GUI/dash_app_prod.py > run_output/dash.log 2>&1 &
                        fi
                        echo \$! > run_output/dash.pid
                        echo "Dash запущен с PID: \$(cat run_output/dash.pid)"
                        sleep 5

                        echo "=== Проверка health-эндпоинтов ==="
                        curl -fsS http://127.0.0.1:8000/healthz > run_output/health_business.json || echo "Business health check failed" > run_output/health_business.json
                        curl -fsS http://127.0.0.1:${DASH_PORT}/healthz > run_output/health_dash.json || echo "Dash health check failed" > run_output/health_dash.json

                        echo "=== Приложение работает, сбор данных 30 секунд ==="
                        sleep 30

                        echo "=== Остановка процессов ==="
                        kill \$(cat run_output/simulator.pid) 2>/dev/null || true
                        kill \$(cat run_output/reciever.pid) 2>/dev/null || true
                        kill \$(cat run_output/business.pid) 2>/dev/null || true
                        kill \$(cat run_output/dash.pid) 2>/dev/null || true
                        sleep 2

                        echo "=== Упаковка артефактов ==="
                        tar -czf artifacts.tgz run_output Reciever/*.csv Business/*.csv 2>/dev/null || tar -czf artifacts.tgz run_output
                        echo "Артефакты упакованы в artifacts.tgz"
                    '
                """
            }
        }

        stage('Collect Artifacts') {
            steps {
                sh """
                    scp -i $KEY_PATH ${REMOTE_USER}@${TARGET_IP}:~/app/artifacts.tgz . || echo "No artifacts found"
                """
                archiveArtifacts artifacts: 'artifacts.tgz', fingerprint: true, allowEmptyArchive: true
            }
        }
    }

    post {
        always {
            // Очистка портов на агенте
            sh '''
                for p in 8000 8050 8051; do
                    fuser -k ${p}/tcp 2>/dev/null || true
                done
            '''
            echo "Сборка завершена. Статус: ${currentBuild.result ?: 'SUCCESS'}"
        }
        
        success {
            echo "✅ Сборка успешно выполнена! Артефакты сохранены."
        }
        
        failure {
            echo "❌ Сборка завершилась с ошибкой. Проверьте логи выше."
        }
    }
}
