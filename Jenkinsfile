#!/usr/bin/env groovy
/*
 * Build Maven Modules
 */

pipeline {
      agent any

      environment {
        MODULE = 'vulnadoava'
        MODULE_ENV_NAME = 'dev'
        ECR_ENV_NAME = 'dev'
        ECR_REGION = 'us-east-1'
        PROJECT = 'cbc'
        VERSION = "${PROJECT}-${MODULE_ENV_NAME}"
        SONAR_HOST = "https://sonarqube.cloudbees.com"
      }

      options {
        disableConcurrentBuilds()
        skipStagesAfterUnstable()
        ansiColor('xterm')
      }

      stages {
        stage('Start') {
          steps {
            cleanWs()
            script {
              currentBuild.description = "${env.GIT_BRANCH} ${env.GIT_COMMIT}"
            }
          }
        }

        stage('Checkout') {
          steps {
            deleteDir()
            checkout scm
          }
        }

        stage('Pre-build') {
          steps {
            script {
              withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: "aws_${ECR_ENV_NAME}_creds"]]) {
                ACCOUNT = sh (script: 'aws sts get-caller-identity --query Account --output text', returnStdout: true).trim()
                ECR_REPO_URI = sh (script: "echo ${ACCOUNT}.dkr.ecr.${env.ECR_REGION}.amazonaws.com/${env.PROJECT}/${env.MODULE}", returnStdout: true).trim()
                ECR_REPO_LOGIN = sh (script: "aws ecr get-login-password --region ${env.ECR_REGION} | docker login --username AWS --password-stdin $ECR_REPO_URI || true", returnStdout: true)
                GIT_SHORT_HASH = env.GIT_COMMIT.take(7)
                TARGET = sh (script: "echo ${ECR_REPO_URI}:${VERSION}", returnStdout: true).trim()
                TARGET_GIT_SHORT_HASH = sh (script: "echo ${ECR_REPO_URI}:${GIT_SHORT_HASH}", returnStdout: true).trim()
              }
            }
          }
        }

        stage('Build') {
          steps {
            withCredentials([[
                $class: 'AmazonWebServicesCredentialsBinding',
                credentialsId: "aws_${ECR_ENV_NAME}_creds"],

                [$class: 'StringBinding',
                credentialsId: "sonarqube_token",
                variable: "SONAR_LOGIN"],

                [$class: 'FileBinding',
                credentialsId: 'maven-settings-xml',
                variable: 'SETTINGS_XML']]) {
                    sh "mkdir -p .m2 || true"
                    sh "cp $SETTINGS_XML .m2/settings.xml"
                    sh "docker build -t $TARGET --build-arg PROJECT_KEY=${env.MODULE} --build-arg SONAR_HOST=${env.SONAR_HOST} --build-arg SONAR_LOGIN=${SONAR_LOGIN} ."  
                }
          }
        }
      }
}
