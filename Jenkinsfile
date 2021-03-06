#!groovy

pipeline
{
  agent none

  stages
  {
    stage("check")
    {
      agent
      {
        docker
        {
          image 'dealii/indent'
        }
      }

      post { cleanup { cleanWs() } }

      stages
      {
        stage("permission")
        {
          steps
          {
            githubNotify context: 'CI', description: 'need ready to test label and /rebuild',  status: 'PENDING'
            // For /rebuild to work you need to:
            // 1) select "issue comment" to be delivered in the github webhook setting
            // 2) install "GitHub PR Comment Build Plugin" on Jenkins
            // 3) in project settings select "add property" "Trigger build on pr comment" with
            //    the phrase ".*/rebuild.*" (without quotes)
            sh '''
               wget -q -O - https://api.github.com/repos/dealii/dealii/issues/${CHANGE_ID}/labels | grep 'ready to test' || \
               { echo "This commit will only be tested when it has the label 'ready to test'. Trigger a rebuild by adding a comment that contains '/rebuild'..."; exit 1; }
            '''
            githubNotify context: 'CI', description: 'running tests...',  status: 'PENDING'
          }
        }

        stage("indent")
        {
          steps
          {
            // We can not use 'indent' because we are missing the master branch:
            // "fatal: ambiguous argument 'master': unknown revision or path"
            sh '''
               ./contrib/utilities/indent-all
            '''
            sh 'git diff > changes.diff'
            archiveArtifacts artifacts: 'changes.diff', fingerprint: true
            sh '''
               git diff --exit-code || \
               { echo "Please check indentation, see artifacts at the top right!"; exit 1; }
            '''
            githubNotify context: 'indent', description: '',  status: 'SUCCESS'
          }
        }
      }
    }

    stage('Build and Test')
    {
      parallel
      {

        stage('gcc-serial')
        {
          agent
          {
            docker
            {
              image 'tjhei/candi:v9.0.1-r4'
            }
          }
          post { cleanup { cleanWs() } }

          steps
          {
          timeout(time: 6, unit: 'HOURS')
          {
            sh "echo \"building on node ${env.NODE_NAME}\""
            sh '''#!/bin/bash
               export NP=`grep -c ^processor /proc/cpuinfo`
	       export TEST_TIME_LIMIT=1200
               echo $NP
               mkdir -p /home/dealii/build
               cd /home/dealii/build
               cmake -G "Ninja" \
                 -D DEAL_II_CXX_FLAGS='-Werror' \
                 -D CMAKE_BUILD_TYPE=Debug \
                 -D DEAL_II_WITH_MPI=OFF \
                 -D DEAL_II_UNITY_BUILD=ON \
                 $WORKSPACE/
               time ninja -j $NP
               time ninja setup_tests
               time ctest --output-on-failure -DDESCRIPTION="CI-$JOB_NAME" -j $NP
            '''
          }
          }
        }


        stage('gcc-mpi')
        {
          agent
          {
            docker
            {
              image 'tjhei/candi:v9.0.1-r4'
            }
          }
          post { cleanup { cleanWs() } }

          steps
          {
          timeout(time: 6, unit: 'HOURS')
          {
            sh "echo \"building on node ${env.NODE_NAME}\""
            sh '''#!/bin/bash
                export NP=`grep -c ^processor /proc/cpuinfo`
                mkdir -p /home/dealii/build
                cd /home/dealii/build
                cmake -G "Ninja" \
                  -D DEAL_II_CXX_FLAGS='-Werror' \
                  -D CMAKE_BUILD_TYPE=Debug \
                  -D DEAL_II_WITH_MPI=ON \
                  -D DEAL_II_UNITY_BUILD=OFF \
                  $WORKSPACE/
                time ninja -j $NP
                time ninja setup_tests
                time ctest -R "all-headers|multigrid/transfer" --output-on-failure -DDESCRIPTION="CI-$JOB_NAME" -j $NP
            '''
          }
          }
        }

      }
    }

    stage("finalize")
    {
      agent none

      steps
      {
        githubNotify context: 'CI', description: 'OK',  status: 'SUCCESS'
      }
    }
  }
}
