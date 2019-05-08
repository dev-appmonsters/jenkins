def build_push_image(service_dir, svc_name) {

  withCredentials([file(credentialsId: 'gcloud-key', variable: 'CI_GOOGLE_SA_JSON_FILE')]){
    set_image_tag(service_dir)

    dir(service_dir){
      def image_tag = image_tags[service_dir]
      def image_url = "${cr_registry_base}/${ci_google_project_id}/${svc_name}"
      echo "build_push_image ${image_tag}"

      def image_cr_status = sh(returnStdout: true, script: "gcloud container images list-tags ${cr_registry_base}/${ci_google_project_id}/${svc_name} \
                                --format=\"json\" | jq '.[].tags | select(index(\"${image_tag}\"))'").trim()
      echo "image_status ${image_cr_status}"

      if (skip_test == "No" || image_cr_status == null || image_cr_status == '' || image_cr_status.size() <= 0){
        stage("Pull image ${svc_name}"){
          sh "${gcloud} auth activate-service-account ${gcloud_login_sa_id} --key-file=${CI_GOOGLE_SA_JSON_FILE} && \
              cat ${CI_GOOGLE_SA_JSON_FILE} | docker login -u _json_key --password-stdin https://${cr_registry_base} && \
              docker pull ${image_url}:latest || true && \
              docker tag ${image_url}:latest fandom-dev-start_jenkins_${svc_name}:latest || true"
        }
      }

      if(image_cr_status == null || image_cr_status == '' || image_cr_status.size() <= 0){
        stage("build using cache & tag image ${svc_name}"){
          sh "docker build --cache-from=${image_url}:latest \
              --build-arg NODE_ENV=development \
              --build-arg ASDF_CODE_VERSION=${asdf_commit} \
              --build-arg SCHEMAS_CODE_VERSION=${scheme_commit} \
              -t ${image_url}:${image_tag}  \
              -f Dockerfile . && \
              docker tag ${image_url}:${image_tag} ${image_url}:latest && \
              docker tag ${image_url}:${image_tag} fandom-dev-start_jenkins_${svc_name}:latest"
        }
        stage("push image ${svc_name}"){
          withCredentials([file(credentialsId: 'gcloud-key', variable: 'CI_GOOGLE_SA_JSON_FILE')]){
            sh "${gcloud} auth activate-service-account ${gcloud_login_sa_id} --key-file=${CI_GOOGLE_SA_JSON_FILE} && \
                cat ${CI_GOOGLE_SA_JSON_FILE} | docker login -u _json_key --password-stdin https://${cr_registry_base} && \
                docker push ${image_url}:${image_tag} && \
                docker push ${image_url}:latest"
          }
        }
      }
    }
  }
}


def set_image_tag(service_dir){
  def git_dir_hash = sh(returnStdout: true, script: "git rev-parse --short HEAD:${service_dir}").trim()
  image_tags[service_dir] = "${git_dir_hash}-${asdf_commit}-${scheme_commit}"
}

def deploy_helm (chart_dir, chart_name, kube_cluster, kube_namespace, kube_region, kube_context, google_projects, params) {
  dir(chart_dir){
    sh "${gcloud} container clusters get-credentials ${kube_cluster} --region=${kube_region} --project ${google_projects} && \
        helm upgrade ${chart_name} \
        --kube-context=${kube_context} \
        --debug \
        --install . --namespace=${kube_namespace} \
        ${params}"
  }
}

def deploy_configs (chart_dir, kube_cluster, kube_namespace, kube_region, kube_context, google_projects) {
  dir(chart_dir){
    sh "${gcloud} container clusters get-credentials ${kube_cluster} --region=${kube_region} --project ${google_projects} && \
        make configs-deploy-${env.BRANCH_NAME}"
  }
}

def get_deployed_image_tag(svc, kube_cluster, kube_region, google_projects) {
  tag = sh(returnStdout: true, script: "${gcloud} container clusters get-credentials ${kube_cluster} --region=${kube_region} --project ${google_projects} && \
    kubectl get deployment -l app.kubernetes.io/name=${svc} -o json | jq '.items[] | select(.metadata.name == \"${svc}\")' | jq '.spec.template.spec.containers[] | select(.name == \"${svc}\")' | jq .image | tr -d '\"'  | cut -d':' -f2").trim()
  return tag
}

def deploy_svc (service_dir, svc_name, kube_cluster, kube_namespace, kube_region, kube_context, google_projects){
  def image_tag = image_tags[service_dir]
  def deployed_image_tag = get_deployed_image_tag(svc_name, kube_cluster, kube_region, google_projects)

  echo "*** The deployed tag is:  ${deployed_image_tag} ***\n*** The latest tag is: ${image_tag} ***"

  if (deployed_image_tag != image_tag){
    echo "Different tags for ${svc_name}"
  } else {
    echo "Same tags for ${svc_name}"
    return
  }

  def image_url = "${cr_registry_base}/${ci_google_project_id}/${svc_name}"
  deploy_helm(
    "${service_dir}/k8s/${svc_name}/",
    svc_name,
    kube_cluster,
    kube_namespace,
    kube_region,
    kube_context,
    google_projects,
    "--set=image.repository=${image_url} --set=image.tag=${image_tag}"
  )
}


def label_slave = "mypod-${UUID.randomUUID().toString()}"
podTemplate(
    label: label_slave,
    containers: [
      containerTemplate(
        name: 'jenkins-dind',
        image: 'r3dio/custom-dind:v9',
        alwaysPullImage: true,
        privileged: true,
        ttyEnabled: true,
        command: 'dockerd-entrypoint.sh',
        resourceRequestCpu: '3000m',
        resourceRequestMemory: '4000Mi',
      )
    ],
    volumes: [emptyDirVolume(mountPath: '/var/lib/docker', memory: false)],
    nodeSelector: 'cloud.google.com/gke-preemptible: "true"',
    slaveConnectTimeout: 9000,
    // idleMinutes: 100
  )
{
  node(label_slave) {
    properties(
      [
        parameters(
          [
            choice(name: 'skip_test', choices:"No\nYes", description: "Do you whish to Run test cases ?" )
          ]
        )
      ]
    )
    container('jenkins-dind'){

      environment {
        image_tags = [:]
        cr_registry_base = "fake"
        ci_google_project_id = "fake"
        gcloud_login_sa_id = "fake"
        gcloud_login_sa_json_jenkins_cred_id = "fake"
        gcloud = "fake"
        asdf_commit = "fake"
        scheme_commit = "fake"
      }

      parameters {
        booleanParam(defaultValue: true, description: 'Wether to skip testing or not', name: 'skipTest')
      }

      stage('Assign variable'){
        image_tags = [:]
        cr_registry_base = "us.gcr.io"
        ci_google_project_id = "fandom-ci"
        gcloud_login_sa_id = "jenkins-ci@fandom-ci.iam.gserviceaccount.com"
        gcloud_login_sa_json_jenkins_cred_id = "gcloud-key"
        gcloud = "/root/google-cloud-sdk/bin/gcloud"
        // def userInput = input(
        //     id: 'userInput', message: 'This is PRODUCTION!', parameters: [
        //     [$class: 'BooleanParameterDefinition', defaultValue: false, description: '', name: 'Please confirm you sure to proceed']
        // ])
      }

      stage('Get cluster credentials'){
       withCredentials([file(credentialsId: "${gcloud_login_sa_json_jenkins_cred_id}", variable: 'CI_GOOGLE_SA_JSON_FILE')]){
          sh "${gcloud} auth activate-service-account ${gcloud_login_sa_id} --key-file=${CI_GOOGLE_SA_JSON_FILE}"
        }
      }

      stage('Build asdf & schemas'){
      git (url: 'git@bitbucket.org:fandomsportsv2/fandom-dev-start.git', branch: env.BRANCH_NAME , credentialsId: 'bitbucket_ssh_key')
        withCredentials([file(credentialsId: 'gcloud-key', variable: 'CI_GOOGLE_SA_JSON_FILE')]){

          asdf_commit = sh(returnStdout: true, script: 'git rev-parse --short HEAD:common/asdf').trim()
          scheme_commit = sh(returnStdout: true, script: 'git rev-parse --short HEAD:common/schemas').trim()

          stage("pull image asdf & schemas"){
            sh "${gcloud} auth activate-service-account ${gcloud_login_sa_id} --key-file=${CI_GOOGLE_SA_JSON_FILE} && \
                cat ${CI_GOOGLE_SA_JSON_FILE} | docker login -u _json_key --password-stdin https://${cr_registry_base} && \
                docker pull ${cr_registry_base}/${ci_google_project_id}/asdf:${asdf_commit} || true && \
                docker pull ${cr_registry_base}/${ci_google_project_id}/schemas:${scheme_commit} || true"
          }

          dir('common') {
            dir('asdf') {
              def image_url = "${cr_registry_base}/${ci_google_project_id}/asdf"
              sh "docker build --cache-from=${image_url}:${asdf_commit} \
                  --build-arg NODE_ENV=development \
                  -t ${image_url}:${asdf_commit}  \
                  -f Dockerfile ."
              sh "docker tag ${image_url}:${asdf_commit} asdf:latest"
              sh "docker tag ${image_url}:${asdf_commit} asdf:${asdf_commit}"
              sh "docker push ${image_url}:${asdf_commit}"
            }
          }

          dir('common') {
            dir('schemas') {
              dir('v3'){
                def image_url = "${cr_registry_base}/${ci_google_project_id}/schemas"
                sh "docker build --cache-from=${image_url}:${scheme_commit} \
                  --build-arg NODE_ENV=development \
                  -t ${image_url}:${scheme_commit}  \
                  -f Dockerfile ."
                sh "docker tag ${image_url}:${scheme_commit} schemas:latest"
                sh "docker tag ${image_url}:${scheme_commit} schemas:${scheme_commit}"
                sh "docker push ${image_url}:${scheme_commit}"
              }
            }
          }

        }
      }

      stage('Build ALL Docker images'){
        parallel(
          scrapper: {
            build_push_image('scrapper','scrapper')
          },
          sportradar: {
            build_push_image('sportradar','sportradar')
          },
          websocket: {
            build_push_image('websocket','websocket')
          },
          chat: {
            build_push_image('chat','chat')
          },
          profile: {
            build_push_image('profile','profile')
          },
          post: {
            build_push_image('post','post')
          },
          admin_ui: {
            build_push_image('fandom-admin-ui','fandom-admin-ui')
          },
        )
        parallel(
          content: {
            build_push_image('content','content')
          },
          purchase: {
            build_push_image('in-app-purchase', 'purchase')
          },
          economy: {
            build_push_image('economy','economy')
          },
          feed: {
            build_push_image('feed','feed')
          },
          fight: {
            build_push_image('fight','fight')
          },
          follower: {
            build_push_image('follower','follower')
          },
          gamelayer: {
            build_push_image('gamelayer','gamelayer')
          },
        )
        parallel(
          fabric: {
            build_push_image('fabric','fabric')
          },
          identity: {
            build_push_image('identity','identity')
          },
          informer: {
            build_push_image('informer','informer')
          },
          mobileAggregatorApi: {
            build_push_image('mobile-aggregator-api','mobile-aggregator-api')
          },
          notificationFeed: {
            build_push_image('notification-feed','notification-feed')
          },
          referral: {
            build_push_image('referral','referral')
          }
        )
      }

      if (skip_test == "No"){
        stage('Run integration test'){
          sh "make services-up && make set-up-db && make test"
        }
      }

        def kube_cluster = ""
        def kube_namespace = ""
        def kube_region = "us-central1"
        def kube_context = ""
        def google_project = ""
        def skip_helm_upgrade = false

        switch (env.BRANCH_NAME) {
          case 'staging':
            kube_cluster = "fandom-staging"
            kube_context = "gke_fandom-dev-241610_us-central1_fandom-staging"
            google_project = "fandom-dev-241610"
            break
          case 'rc1':
            kube_cluster = "fandom-rc1"
            kube_context = "gke_fandom-rc1_us-central1_fandom-rc1"
            google_project = "fandom-rc1"
            break
          case 'rc2':
            kube_cluster = "fandom-rc2"
            kube_context = "gke_fandom-rc2_us-central1_fandom-rc2"
            google_project = "fandom-rc2"
            break
          case 'production':
            kube_cluster = "production-fandom"
            kube_context = "gke_fandom-production-241312_us-central1_production-fandom"
            google_project = "fandom-production-241312"
            break
          default:
            skip_helm_upgrade = true
        }

      if (skip_helm_upgrade != true){
        stage('HELM upgrade image') {

          switch(env.BRANCH_NAME) {
            case 'staging':
              withCredentials([file(credentialsId:'gcloud-key-staging', variable: 'STAGING_GOOGLE_SA_JSON_FILE')]){
                sh "${gcloud} auth activate-service-account jenkins@fandom-dev-241610.iam.gserviceaccount.com --key-file=${STAGING_GOOGLE_SA_JSON_FILE}"
              }
              break
            case 'rc1':
              withCredentials([file(credentialsId:'gcloud-key-rc1', variable: 'RC1_GOOGLE_SA_JSON_FILE')]){
                sh "${gcloud} auth activate-service-account jenkins@fandom-rc1.iam.gserviceaccount.com --key-file=${RC1_GOOGLE_SA_JSON_FILE}"
              }
              break
            case 'rc2':
              withCredentials([file(credentialsId:'gcloud-key-rc2', variable: 'RC2_GOOGLE_SA_JSON_FILE')]){
                sh "${gcloud} auth activate-service-account jenkins@fandom-rc2.iam.gserviceaccount.com --key-file=${RC2_GOOGLE_SA_JSON_FILE}"
              }
              break
            case 'production':
              withCredentials([file(credentialsId:'gcloud-key-production', variable: 'PRODUCTION_GOOGLE_SA_JSON_FILE')]){
                sh "${gcloud} auth activate-service-account jenkins@fandom-production-241312.iam.gserviceaccount.com --key-file=${PRODUCTION_GOOGLE_SA_JSON_FILE}"
              }
              break
          }
          // deploy_svc (service_dir, svc_name, kube_cluster, kube_namespace, kube_region, kube_context, google_projects)
          configs: {
            deploy_configs('deployment/charts/', kube_cluster, kube_namespace, kube_region, kube_context, google_project)
          }

          purchase: {
            deploy_svc('in-app-purchase','purchase', kube_cluster, kube_namespace, kube_region, kube_context, google_project)
          }

          adminui: {
            deploy_svc('fandom-admin-ui','fandom-admin-ui', kube_cluster, kube_namespace, kube_region, kube_context, google_project)
          }

          scrapper: {
            deploy_svc('scrapper','scrapper', kube_cluster, kube_namespace, kube_region, kube_context, google_project)
          }
          post: {
            deploy_svc('post','post', kube_cluster, kube_namespace, kube_region, kube_context, google_project)
          }
          sportradar: {
            deploy_svc('sportradar','sportradar', kube_cluster, kube_namespace, kube_region, kube_context, google_project)
          }
          websocket: {
            deploy_svc('websocket','websocket', kube_cluster, kube_namespace, kube_region, kube_context, google_project)
          }
          profile: {
            deploy_svc('profile','profile', kube_cluster, kube_namespace, kube_region, kube_context, google_project)
            // sh "${gcloud} container clusters get-credentials ${kube_cluster} --region=${kube_region} --project ${google_project} && \
            //   kubernetes-run default ${kube_context} ./node_modules/.bin/typeorm migration:run --template=profile"
          }
          chat: {
            deploy_svc('chat','chat', kube_cluster, kube_namespace, kube_region, kube_context, google_project)
          }
          content: {
            deploy_svc('content','content', kube_cluster, kube_namespace, kube_region, kube_context, google_project)
          }
          economy: {
            deploy_svc('economy','economy', kube_cluster, kube_namespace, kube_region, kube_context, google_project)
            // sh "${gcloud} container clusters get-credentials ${kube_cluster} --region=${kube_region} --project ${google_project} && \
            //   kubernetes-run default ${kube_context} ./node_modules/.bin/typeorm migration:run --template=economy"
          }
          feed: {
            deploy_svc('feed','feed', kube_cluster, kube_namespace, kube_region, kube_context, google_project)
          }
          fight: {
            deploy_svc('fight','fight', kube_cluster, kube_namespace, kube_region, kube_context, google_project)
            // sh "${gcloud} container clusters get-credentials ${kube_cluster} --region=${kube_region} --project ${google_project} && \
            //   kubernetes-run default ${kube_context} ./node_modules/.bin/typeorm migration:run --template=fight"
          }
          fabric: {
            deploy_svc('fabric','fabric', kube_cluster, kube_namespace, kube_region, kube_context, google_project)
          }
          follower: {
            deploy_svc('follower','follower', kube_cluster, kube_namespace, kube_region, kube_context, google_project)
            // sh "${gcloud} container clusters get-credentials ${kube_cluster} --region=${kube_region} --project ${google_project} && \
            //   kubernetes-run default ${kube_context} ./node_modules/.bin/typeorm migration:run --template=follower"
          }
          gamelayer: {
            deploy_svc('gamelayer','gamelayer', kube_cluster, kube_namespace, kube_region, kube_context, google_project)
            // sh "${gcloud} container clusters get-credentials ${kube_cluster} --region=${kube_region} --project ${google_project} && \
            //   kubernetes-run default ${kube_context} ./node_modules/.bin/typeorm migration:run --template=gamelayer"
          }
          identity: {
            deploy_svc('identity','identity', kube_cluster, kube_namespace, kube_region, kube_context, google_project)
            // sh "${gcloud} container clusters get-credentials ${kube_cluster} --region=${kube_region} --project ${google_project} && \
            //   kubernetes-run default ${kube_context} ./node_modules/.bin/typeorm migration:run --template=identity"
          }
          informer: {
            deploy_svc('informer','informer', kube_cluster, kube_namespace, kube_region, kube_context, google_project)
          }
          mobileAggregatorApi: {
            deploy_svc('mobile-aggregator-api','mobile-aggregator-api', kube_cluster, kube_namespace, kube_region, kube_context, google_project)
          }
          notificationFeed: {
            deploy_svc('notification-feed','notification-feed', kube_cluster, kube_namespace, kube_region, kube_context, google_project)
          }
          referral: {
            deploy_svc('referral','referral', kube_cluster, kube_namespace, kube_region, kube_context, google_project)
          }
        }
      }
    }
  }
}
