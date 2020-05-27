@Library('dynatrace@master') _
@Library('keptn-library@master') _
import sh.keptn.Keptn
def keptn = new sh.keptn.Keptn()
import groovy.json.JsonSlurper
import groovy.json.JsonSlurperClassic

pipeline {
  agent {
    label 'kubegit'
  }
  environment {
    APP_NAME = "carts"
    KEPTN_PROJECT = "sockshop-qg"
    KEPTN_SERVICE = "carts"
    KEPTN_STAGE = "hardening"
    JMETER_VUCOUNT = 1
    JMETER_LOOPCOUNT = 4000
  }
  stages {
    
    stage('Keptn Init') {
      steps{
        script {
          keptn.keptnInit project:"sockshop-qg", service:"carts", stage:"hardening", monitoring: "dynatrace", shipyard:'keptn/carts-shipyard.yaml'
          keptn.keptnAddResources('keptn/carts-sli.yaml','dynatrace/sli.yaml')
          keptn.keptnAddResources('keptn/carts-slo.yaml','slo.yaml')
          keptn.keptnAddResources('keptn/dynatrace.conf.yaml','dynatrace/dynatrace.conf.yaml')
        }
      }
    }

    stage('Warm up') {
      steps {
        //checkout scm
        echo "Waiting for the service to start..."
        container('kubectl') {
          script {
            def status = waitForDeployment (
              deploymentName: "${env.APP_NAME}",
              environment: 'staging'
            )
            if(status !=0 ){
              currentBuild.result = 'FAILED'
              error "Deployment did not finish before timeout."
            }
          }
        }
        container('jmeter') {
          script {
            def status = executeJMeter ( 
              scriptName: "jmeter/${env.APP_NAME}_perfcheck.jmx",
              resultsDir: "PerfCheck_Warmup_${env.APP_NAME}_${BUILD_NUMBER}",
              serverUrl: "${env.APP_NAME}.staging", 
              serverPort: 80,
              checkPath: '/health',
              vuCount: 1,
              loopCount: 10,
              LTN: "PerfCheck_Warmup_${BUILD_NUMBER}",
              funcValidation: false,
              avgRtValidation: 4000
            )
            if (status != 0) {
              currentBuild.result = 'FAILED'
              error "Performance check failed."
            }
          }
        }
        
        echo "Waiting for a minute to not skew data in DT"
        sleep(60)
      }
    }
    
    stage('Send Test Start to DT') {
        steps {
          container("curl") {
            script {
        
                def tagMatchRules = [[ meTypes: [[meType: 'SERVICE']],
                                          tags : [[context: 'CONTEXTLESS', key: 'keptn_project', value: KEPTN_PROJECT],
                                                  [context: 'CONTEXTLESS', key: 'keptn_service', value: KEPTN_SERVICE],
                                                  [context: 'CONTEXTLESS', key: 'keptn_stage', value: KEPTN_STAGE]]
                                    ]];
                // Push some Jenkins OOTB info to DT.
                def customProps = [ 
                    "Test Type": "Load",
                    "Test Provider": "Jmeter",
                    "Test Parameters": "[vuCount: ${env.JMETER_VUCOUNT}] [loopCount: ${env.JMETER_LOOPCOUNT}]"
                ];
                
                def notification = new pushDynatraceInfoEvent();
                notification.call(title: "Test Start on ${env.KEPTN_PROJECT}/${env.KEPTN_SERVICE}", source: 'Jenkins', description: 'Starting load test.', tagRule: tagMatchRules, customProperties: customProps);
            }
          }
      }
    
    } // end stage

    stage('Run Performance Test') {
      steps {
        script {
            env.testStartTime = get_timestamp()
            keptn.markEvaluationStartTime
        }
        container('jmeter') {
          script {
              
              def status = executeJMeter ( 
                scriptName: "jmeter/${env.APP_NAME}_perfcheck.jmx",
                resultsDir: "PerfCheck_${env.APP_NAME}_${BUILD_NUMBER}",
                serverUrl: "${env.APP_NAME}.staging", 
                serverPort: 80,
                checkPath: '/health',
                vuCount: env.JMETER_VUCOUNT.toInteger(),
                loopCount: env.JMETER_LOOPCOUNT.toInteger(),
                LTN: "PerfCheck_${BUILD_NUMBER}",
                funcValidation: false,
                avgRtValidation: 4000
              )
              if (status != 0) {
                currentBuild.result = 'FAILED'
                error "Performance check failed."
              }
            }
        }
        script {
            env.testEndTime = get_timestamp()
        }

        //echo "Waiting for a minute so data can be processed in Dynatrace"
        //sleep(60)
      }
    }
    
    stage('Send Test Stop to DT') {
            steps {
              container("curl") {
                script {
            
                    def tagMatchRules = [[ meTypes: [[meType: 'SERVICE']],
                                             tags : [[context: 'CONTEXTLESS', key: 'keptn_project', value: KEPTN_PROJECT],
                                                      [context: 'CONTEXTLESS', key: 'keptn_service', value: KEPTN_SERVICE],
                                                      [context: 'CONTEXTLESS', key: 'keptn_stage', value: KEPTN_STAGE]]
                                        ]];
                    // Push some Jenkins OOTB info to DT.
                    def customProps = [ 
                        "Test Type": "Load",
                        "Test Provider": "Jmeter",
                        "Test Parameters": "[vuCount: ${env.JMETER_VUCOUNT}] [loopCount: ${env.JMETER_LOOPCOUNT}]"
                    ];
                    
                    def notification = new pushDynatraceInfoEvent();
                    notification.call(title: "Test Stop on ${env.KEPTN_PROJECT}/${env.KEPTN_SERVICE}", source: 'Jenkins', description: 'Starting load test.', tagRule: tagMatchRules, customProperties: customProps);
                }
             }
          }
        
        } // end stage
  
    stage('Evaluate Build with Keptn') {
      steps {
        script {
          def keptnContext = keptn.sendStartEvaluationEvent starttime:"", endtime:"" 
          echo "Open Keptns Bridge: ${keptn_bridge}/trace/${keptnContext}"
        }
      }
    }
        /*stage('Send Evaluation Result to DT') {
            steps {
              container("curl") {
                script {
            
                    def tagMatchRules = [[ meTypes: [[meType: 'SERVICE']],
                                             tags : [[context: 'CONTEXTLESS', key: 'keptn_project', value: KEPTN_PROJECT],
                                                      [context: 'CONTEXTLESS', key: 'keptn_service', value: KEPTN_SERVICE],
                                                      [context: 'CONTEXTLESS', key: 'keptn_stage', value: KEPTN_STAGE]]
                                        ]];
                    // Push some Jenkins OOTB info to DT.
                    env.bridge_evalUrl = "${env.KEPTN_BRIDGE}/project/${env.KEPTN_PROJECT}/${env.KEPTN_SERVICE}/${env.keptn_context}/${env.keptn_evaluationContext}"
                    def customProps = [ 
                        "Evaluation Result": "${env.evaluationResult}", 
                        "Evaluation Score": "${env.evaluationScore}", 
                        "Jenkins Build Tag": "${BUILD_TAG}", 
                        "Jenkins Build URL": "${BUILD_URL}",
                        "Evaluation Details": "${env.bridge_evalUrl}"
                    ];
                    
                    def notification = new pushDynatraceInfoEvent();
                    notification.call(title: "Keptn Quality Gates Evaluation: ${env.evaluationResult}", source: 'Jenkins', description: 'Evaluated build score for this build.', tagRule: tagMatchRules, customProperties: customProps);
                }
             }
          }
        
        }*/ // end stage
        stage('Pipeline Quality Gate') {
            steps {
          
                script {
                    def result = keptn.waitForEvaluationDoneEvent setBuildResult:true, waitTime:waitTime
                    echo "${result}"
                    //if The Keptn result is "Fail", fail the build.
                    /*echo env.evaluationResult
                    if (env.evaluationResult == "fail") {
                        echo "[pipeline] FAILING PIPELINE Because Keptn Failed the Build";
                        echo "Check the Keptn Bridge for more details: ${env.bridge_evalUrl}"
                        //currentBuild.currentResult = 'FAILURE';
                        error('Failing pipeline because keptn failed the build...');
                    }*/
                }
            }
        }// end stage
    }// end stages
} // end pipeline
def get_timestamp(){
    DATE_TAG = java.time.LocalDate.now()
    DATETIME_TAG = java.time.LocalDateTime.now()
    echo "${DATETIME_TAG}"
                
    return DATETIME_TAG
}