pipeline {
    agent {
        node {
        	//This doesn't really require a specific image. The only dependency on terraform here is for a terraform fmt
            label 'base-with-terraform'
        }
    }
    parameters {
            string(name: 'FW_NAME', defaultValue: 'DEFAULT_NAME', description: 'Name for the fireall rule')
            string(name: 'DESCRIPTION', defaultValue: 'DEFAULT_DESCRIPTION', description: 'Basic description of what the firewall is for')
            string(name: 'TCP_PORTS', defaultValue: '', description: 'Comma delimited list of TCP ports that need to be opened for your GCP resources. (ex., 2376,10443,8080-8082)')
            string(name: 'UDP_PORTS', defaultValue: '', description: 'Comma delimited list of UDP ports that need to be opened for your GCP resources. (ex., 2376,10443,8080-8082)')
            string(name: 'SOURCE_RANGES', defaultValue: '', description: '')
            string(name: 'TARGET_TAGS', defaultValue: '', description: 'Comma delimited list of target tags (eg., windows-core,linux-core,teradici)')
    }
    stages {
        stage ('Checkout Scm') {
            steps {
            	/*
            	As this is a somewhat hardcoded implementation, the git URL needs to be the one hosting the actual terraform file that manages firewalls
            	*/
                checkout([$class: 'GitSCM', branches: [[name: '*/master']], extensions: [], userRemoteConfigs: [[credentialsId: 'github', url: '##GITURL_PLACEHOLDER##']]])
            }
        }

        stage ('Shell Script') {
            steps {
            	//Again, hardcoded for an internal implementation, and thus redacted
                withCredentials([sshUserPrivateKey(credentialsId: 'github', keyFileVariable: '##REDACTED##', usernameVariable: '##REDACTED##')]) {
                    sh '''#https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/compute_firewall
set +x

#Set up id_rsa for git
mkdir -p ~/.ssh
cat ##REDACTED## > ~/.ssh/id_rsa
chmod 700 ~/.ssh/id_rsa

#Run git config commands
echo "Setting up git configs and pulling"
git config --global user.email "kube-constech-jenkins@activision.com"
git config --global user.name "kube-constech-jenkins"

#Long-lived branch strategy for additions to the firewall
git checkout -b jenkins_add_firewall_working_branch origin/jenkins_add_firewall_working_branch
git pull
git status

#Set up a temp file to hold pending changes
echo "Setting up temp file for new firewall rule"
#cd to dir hosting the firewall TF
cd firewall-rules
#Remove fixed placeholders; comment and 3 braces
head -n -4 fw.tf > working_file.tf | sed '${s/$/,/;}'
echo "    }," > to_append.txt
echo "    \"${FW_NAME}\" = {
        description = \"INGRESS - ${DESCRIPTION}\"
        direction = \"INGRESS\"
        priority = 1000
        allow = { \"tcp\" = [${TCP_PORT_LIST}] }
        allow = { \"udp\" = [${UDP_PORT_LIST}] }
        source_ranges = { ${SOURCE_RANGE_LIST} }
        target_tags = [${TARGET_TAG_LIST}]
        lifecycle = { create_before_destroy = true }
      }" >> to_append.txt
echo "    #DO NOT MESS WITH THE LAST 4 LINES
    }
  }" >> to_append.txt

#Moving in pending changes
cat to_append.txt >> working_file.tf
echo "Syncing new rule to fw.tf and formatting"
\\mv working_file.tf fw.tf
/terraform fmt

#Cleanup
rm -f to_append.txt

#DEBUG
#tail -20 fw.tf

#Push to git
git add -A .
git commit -m "Adding new firewall rule via Jenkins job"
git push --set-upstream origin  jenkins_add_firewall_working_branch
'''
                }
            }
        }
    }
}
