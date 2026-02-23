pipeline {
    agent { label params.AGENT_LABEL }   // запуск на инструментальном сервере

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
        KEY_PATH = '/home/ubuntu/Nikita_Alecsentsev.pem'   // путь к ключу на агенте
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
                        . ~/openrc.sh
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
                    // Получаем IP созданного сервера (имя из шаблона: Nikita-ML-Server)
                    env.TARGET_IP = sh(
                        script: """
                            source . ~/openrc.sh
                            openstack server list --name Nikita-ML-Server -f value -c Networks | awk -F'=' '{print \$2}'
                        """,
                        returnStdout: true
                    ).trim()
                    echo "Target server IP: ${TARGET_IP}"
                }
            }
        }

        stage('Deploy Application on Target Server') {
            steps {
                script {
                    // Копируем ключ (если необходимо) и выполняем подготовку на удалённом сервере
                    sh """
                        ssh -o StrictHostKeyChecking=no -i $KEY_PATH ${REMOTE_USER}@${TARGET_IP} '
                            set -e
                            # Клонирование репозитория с приложением
                            git clone $APP_REPO ~/app || true
                            cd ~/app
                            # Создание виртуального окружения и установка зависимостей
                            python3 -m venv .venv
                            source .venv/bin/activate
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
                        source .venv/bin/activate
                        export LOG_LEVEL=INFO
                        export WEBSOCKET_HOST=127.0.0.1
                        export BUSINESS_HTTP_BASE=http://127.0.0.1:8000
                        export BUSINESS_BIND_HOST=127.0.0.1
                        export BUSINESS_BIND_PORT=8000
                        export DASH_HOST=0.0.0.0
                        export DASH_PORT=${DASH_PORT}

                        # Очистка занятых портов
                        for p in 8000 8050 8051 8092 8093 8094 8095; do
                            fuser -k \${p}/tcp 2>/dev/null || true
                        done

                        mkdir -p run_output

                        # Запуск компонентов
                        python Simulator/simulator.py > run_output/simulator.log 2>&1 &
                        echo \$! > run_output/simulator.pid
                        sleep 3

                        python Reciever/reciever.py > run_output/reciever.log 2>&1 &
                        echo \$! > run_output/reciever.pid
                        sleep 3

                        python Business/business.py > run_output/business.log 2>&1 &
                        echo \$! > run_output/business.pid
                        sleep 3

                        if [ "${DASH_MODE}" = "test" ]; then
                            python GUI/dash_app_test.py > run_output/dash.log 2>&1 &
                        else
                            python GUI/dash_app_prod.py > run_output/dash.log 2>&1 &
                        fi
                        echo \$! > run_output/dash.pid
                        sleep 5

                        # Проверка health-эндпоинтов (если реализованы)
                        curl -fsS http://127.0.0.1:8000/healthz > run_output/health_business.json || true
                        curl -fsS http://127.0.0.1:${DASH_PORT}/healthz > run_output/health_dash.json || true

                        # Окно ручной проверки (5 минут) – пропускаем для автоматизации
                        # Сразу собираем артефакты, но сначала дадим поработать 30 сек
                        sleep 30

                        # Остановка процессов
                        kill \$(cat run_output/simulator.pid) 2>/dev/null || true
                        kill \$(cat run_output/reciever.pid) 2>/dev/null || true
                        kill \$(cat run_output/business.pid) 2>/dev/null || true
                        kill \$(cat run_output/dash.pid) 2>/dev/null || true
                        sleep 2

                        # Упаковка артефактов
                        tar -czf artifacts.tgz run_output Reciever/*.csv Business/*.csv 2>/dev/null || tar -czf artifacts.tgz run_output
                    '
                """
            }
        }

        stage('Collect Artifacts') {
            steps {
                sh """
                    scp -i $KEY_PATH ${REMOTE_USER}@${TARGET_IP}:~/app/artifacts.tgz . || true
                """
                archiveArtifacts artifacts: 'artifacts.tgz'
            }
        }
    }

    post {
        always {
            // Дополнительная очистка портов на агенте (необязательно)
            sh 'for p in 8000 8050 8051; do fuser -k \${p}/tcp 2>/dev/null || true; done'
        }
    }
}
