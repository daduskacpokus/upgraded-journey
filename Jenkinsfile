pipeline {
  agent {
    kubernetes {
      yaml '''
        apiVersion: v1
        kind: Pod
        metadata:
          labels:
            app: jenkins
        spec:
          serviceAccount: jenkins        
          containers:
            - name: helm
              image: alpine/helm:3.8.2
              tty: true
              command: ["sleep", "365d"]
            - name: docker
              image: docker:19.03.1-dind
              tty: true
              securityContext:
                privileged: true
              env:
                - name: DOCKER_TLS_CERTDIR
                  value: ""
        '''
    }
  }    
  stages {
    stage("Build"){
      steps{
        script{
          try{
            container('docker'){
              sh 'docker build -t daduskacpokus/upgraded-journey .'
            }
          }catch(err) {
            pagerDuty(err.toString())
            currentBuild.result = 'FAILURE'
            error('Build has failed')
          }
        }
      }
    }
    stage("Test"){
      steps{
        script{
          try{
            container('docker'){              
              sh '''
                docker run -d -p 3000:3000 daduskacpokus/upgraded-journey
                CODE=$(wget --spider -S http://localhost:3000 2>&1| grep "HTTP/"| awk '{print $2}')
                test "$CODE" = "200" && echo "Success" || echo "Failure"; exit 11;
              '''
            }
          }catch(err) {
            pagerDuty(err.toString())
            currentBuild.result = 'FAILURE'
            error('Build has failed')
          }
        }
      }
    }
    stage("Publish"){
      steps{
        script{
          try{
            withCredentials([usernamePassword(credentialsId: 'docker_hub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_TOKEN' )]) {
            container('docker'){
              sh '''
                docker login -u $DOCKER_USER -p $DOCKER_TOKEN
                docker push daduskacpokus/upgraded-journey
              '''
              }
            }
          }catch(err) {
            pagerDuty(err.toString())
            currentBuild.result = 'FAILURE'
            error('Build has failed')
          }
        }
      }
    }
    stage("Deploy"){
      steps{
        script{
          withCredentials([usernamePassword(credentialsId: 'aws', usernameVariable : 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY' )]) {
            try{
              container('helm'){
                sh '''
                  apk add --no-cache aws-cli
                  aws eks update-kubeconfig --name testlab1-infra --region us-east-1
                  helm create upgraded-journey && cd upgraded-journey
                  helm upgrade --install app1 . --set image.tag=latest --set image.repository=daduskacpokus/upgraded-journey
                '''
              }
            }catch(err) {
              pagerDuty(err.toString())
              currentBuild.result = 'FAILURE'
              error('Build has failed')
            }
          }
        }
      }
    }
  }
}

public void pagerDuty(String error){
  // create payload
  def incident = """
      {
      	"incident": {
      		"type": "incident",
      		"title": "Incident triggered by Jenkins",
      		"service": {
      			"id": "PRB9DW0",
      			"type": "service_reference"
      		},
      		"body": {
      			"type": "incident_body",
      			"details": "$error"
      		}
      	}
      }
  """
  withCredentials([string(credentialsId: 'pager_duty', variable: 'TOKEN')]) {
    def response = httpRequest customHeaders: [[name: 'From', value: 'user@exmaple.com'],[name: 'Authorization', value: "Token token=$TOKEN"]], 
      acceptType: 'APPLICATION_JSON', 
      contentType: 'APPLICATION_JSON',
      httpMode: 'POST', quiet: true,
      requestBody: incident,
      url: 'https://api.pagerduty.com/incidents'
  }
}