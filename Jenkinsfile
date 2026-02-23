node('Nikita_label') {
    echo "Starting deployment..."
    
    stage('Deploy') {
        sh '''
            echo "=== DEPLOYMENT SCRIPT ==="
            echo "Target server: 192.168.199.161"
            echo "User: debian"
            echo "Key: /home/ubuntu/Nikita_Alecsentsev.pem"
            
            # Test connection
            ssh -i /home/ubuntu/Nikita_Alecsentsev.pem -o StrictHostKeyChecking=no debian@192.168.199.161 "echo 'Connection successful'"
            
            echo "=== DEPLOYMENT COMPLETE ==="
        '''
    }
}
