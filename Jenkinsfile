@Library('flexy') _

// rename build
def userId = currentBuild.rawBuild.getCause(hudson.model.Cause$UserIdCause)?.userId
if (userId) {
  currentBuild.displayName = userId
}

def RETURNSTATUS = "default"

pipeline {
  agent none

  parameters {
        string(name: 'BUILD_NUMBER', defaultValue: '', description: 'Build number of job that has installed the cluster.')
        choice(choices: ["cluster-density","node-density","node-density-heavy","pod-density","pod-density-heavy","max-namespaces","max-services"], name: 'WORKLOAD', description: '''Type of kube-burner job to run''')
        booleanParam(name: 'WRITE_TO_FILE', defaultValue: false, description: 'Value to write to google sheet (will run https://mastern-jenkins-csb-openshift-qe.apps.ocp-c1.prod.psi.redhat.com/job/scale-ci/job/e2e-benchmarking-multibranch-pipeline/job/write-scale-ci-results)')

        string(name:'JENKINS_AGENT_LABEL',defaultValue:'oc45',description:
        '''
        scale-ci-static: for static agent that is specific to scale-ci, useful when the jenkins dynamic agen
 isn't stable<br>
        4.y: oc4y || mac-installer || rhel8-installer-4y <br/>
            e.g, for 4.8, use oc48 || mac-installer || rhel8-installer-48 <br/>
        3.11: ansible-2.6 <br/>
        3.9~3.10: ansible-2.4 <br/>
        3.4~3.7: ansible-2.4-extra || ansible-2.3 <br/>
        '''
        )

        string(name: 'VARIABLE', defaultValue: '1000', description: '''This variable configures parameter needed for each type of workload. By default 1000. <br>
            pod-density: This will export PODS env variable; set to 200 * num_workers, work up to 250 * num_workers. Creates as many "sleep" pods as configured in this environment variable.<br>
            cluster-density: This will export JOB_ITERATIONS env variable; set to 4 * num_workers. This variable sets the number of iterations to perform (1 namespace per iteration). <br>
            max-namespaces: This will export NAMESPACE_COUNT env variable; set to ~30 * num_workers. The number of namespaces created by Kube-burner.  <br>
            max-services: This will export SERVICE_COUNT env variable; set to 200 * num_workers, work up to 250 * num_workers. Creates n-replicas of an application deployment (hello-openshift) and a service in a single namespace.  <br>
            node-density: This will export PODS_PER_NODE env variable; set to 200, work up to 250. Creates as many "sleep" pods as configured in this variable - existing number of pods on node. <br>
            node-density-heavy: This will export PODS_PER_NODE env variable; set to 200, work up to 250. Creates this number of applications proportional to the calculated number of pods / 2 <br>
            Read here for detail of each variable: <br>
            https://github.com/cloud-bulldozer/e2e-benchmarking/blob/master/workloads/kube-burner/README.md <br>
            ''')

        separator(name: "NODE_DENSITY_JOB_INFO", sectionHeader: "Node Density Job Options", sectionHeaderStyle: """
				font-size: 14px;
				font-weight: bold;
				font-family: 'Orienta', sans-serif;
			""")
        string(name: 'NODE_COUNT', defaultValue: '3', description: 'Number of nodes to be used in your cluster for this workload. Should be the number of worker nodes on your cluster')
        text(name: 'ENV_VARS', defaultValue: '', description:'''<p>
               Enter list of additional (optional) Env Vars you'd want to pass to the script, one pair on each line. <br>
               e.g.<br>
               SOMEVAR1='env-test'<br>
               SOMEVAR2='env2-test'<br>
               ...<br>
               SOMEVARn='envn-test'<br>
               </p>'''
            )
        string(name: 'E2E_BENCHMARKING_REPO', defaultValue:'https://github.com/cloud-bulldozer/e2e-benchmarking', description:'You can change this to point to your fork if needed.')
        string(name: 'E2E_BENCHMARKING_REPO_BRANCH', defaultValue:'master', description:'You can change this to point to a branch on your fork if needed.')
    }

  stages {
    stage('Run Kube-Burner Test'){
      agent { label params['JENKINS_AGENT_LABEL'] }
      steps{
        deleteDir()
        checkout([
          $class: 'GitSCM',
          branches: [[name: params.E2E_BENCHMARKING_REPO_BRANCH ]],
          doGenerateSubmoduleConfigurations: false,
          userRemoteConfigs: [[url: params.E2E_BENCHMARKING_REPO ]
          ]])

        copyArtifacts(
            filter: '',
            fingerprintArtifacts: true,
            projectName: 'ocp-common/Flexy-install',
            selector: specific(params.BUILD_NUMBER),
            target: 'flexy-artifacts'
        )
        script {
          buildinfo = readYaml file: "flexy-artifacts/BUILDINFO.yml"
          currentBuild.displayName = "${currentBuild.displayName}-${params.BUILD_NUMBER}"
          currentBuild.description = "Copying Artifact from Flexy-install build <a href=\"${buildinfo.buildUrl}\">Flexy-install#${params.BUILD_NUMBER}</a>"
          buildinfo.params.each { env.setProperty(it.key, it.value) }
        }
        script{
          ansiColor('xterm') {
          RETURNSTATUS = sh(returnStatus: true, script: '''
          # Get ENV VARS Supplied by the user to this job and store in .env_override
          echo "$ENV_VARS" > .env_override
          # Export those env vars so they could be used by CI Job
          set -a && source .env_override && set +a
          mkdir -p ~/.kube
          cp $WORKSPACE/flexy-artifacts/workdir/install-dir/auth/kubeconfig ~/.kube/config
          oc config view
          oc projects
          ls -ls ~/.kube/
          env
          cd workloads/kube-burner
          wget https://www.python.org/ftp/python/3.8.12/Python-3.8.12.tgz
          tar -zxvf Python-3.8.12.tgz
          cd Python-3.8.12
          newdirname=~/.localpython
          if [ -d "$newdirname" ]; then
            echo "Directory already exists"
          else
            mkdir -p $newdirname
            ./configure --prefix=$HOME/.localpython
            make
            make install
          fi
          /home/jenkins/.localpython/bin/python3 --version
          python3 -m pip install virtualenv
          python3 -m virtualenv venv3 -p $HOME/.localpython/bin/python3
          source venv3/bin/activate
          python --version
          cd ..
          if [[ $WORKLOAD == "cluster-density" ]]; then
            export JOB_ITERATIONS=$VARIABLE
          elif [[ $WORKLOAD == "pod-density" ]] || [[ $WORKLOAD == "pod-density-heavy" ]]; then
            export PODS=$VARIABLE
          elif [[ $WORKLOAD == "max-namespaces" ]]; then
            export NAMESPACE_COUNT=$VARIABLE
          elif [[ $WORKLOAD == "max-services" ]]; then
            export SERVICE_COUNT=$VARIABLE
          elif [[ $WORKLOAD == "node-density" ]] || [[ $WORKLOAD == "node-density-heavy" ]]; then
            export PODS_PER_NODE=$VARIABLE
          fi
          ./run.sh

          ''')
          }
        }
        script{
            def status = "FAIL"
            if( RETURNSTATUS.toString() == "0") {
                status = "PASS"
            }else {
                currentBuild.result = "FAILURE"
            }
           if(params.WRITE_TO_FILE == true) {
            build job: 'scale-ci/e2e-benchmarking-multibranch-pipeline/write-scale-ci-results', parameters: [string(name: 'BUILD_NUMBER', value: BUILD_NUMBER),text(name: "ENV_VARS", value: ENV_VARS),string(name: 'CI_JOB_ID', value: BUILD_ID), string(name: 'CI_JOB_URL', value: BUILD_URL), string(name: 'JENKINS_AGENT_LABEL', value: JENKINS_AGENT_LABEL), string(name: "CI_STATUS", value: "${status}"), string(name: "JOB", value: WORKLOAD)]
           }
       }
      }

    }
 }
}