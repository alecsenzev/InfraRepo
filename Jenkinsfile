node('Nikita_label') {
    stage('Deploy') {
        sh '''
            echo "Starting deployment..."
            echo "Target: 192.168.199.94"
            echo "Testing connection..."
            ssh -i /home/ubuntu/Nikita_Alecsentsev.pem -o StrictHostKeyChecking=no debian@192.168.199.161 "echo 'Connection OK'"
            echo "Deployment completed"
        '''
    }
}
