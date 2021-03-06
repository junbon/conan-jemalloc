project = "conan-jemalloc"

conan_remote = "ess-dmsc-local"
conan_user = "ess-dmsc"
conan_pkg_channel = "stable"

images = [
  'centos7': [
    'name': 'essdmscdm/centos7-build-node:3.0.0',
    'sh': '/usr/bin/scl enable devtoolset-6 -- /bin/bash -e'
  ],
  'debian9': [
    'name': 'essdmscdm/debian9-build-node:2.0.0',
    'sh': 'bash -e'
  ],
  'ubuntu1804': [
    'name': 'essdmscdm/ubuntu18.04-build-node:1.1.0',
    'sh': 'bash -e'
  ]
]

base_container_name = "${project}-${env.BRANCH_NAME}-${env.BUILD_NUMBER}"

def get_pipeline(image_key) {
  return {
    node('docker') {
      def container_name = "${base_container_name}-${image_key}"
      try {
        def image = docker.image(images[image_key]['name'])
        def custom_sh = images[image_key]['sh']
        def container = image.run("\
          --name ${container_name} \
          --tty \
          --cpus=2 \
          --memory=4GB \
          --network=host \
          --env http_proxy=${env.http_proxy} \
          --env https_proxy=${env.https_proxy} \
          --env local_conan_server=${env.local_conan_server} \
        ")

        stage("${image_key}: Checkout") {
          sh """docker exec ${container_name} ${custom_sh} -c \"
            git clone \
              --branch ${env.BRANCH_NAME} \
              https://github.com/ess-dmsc/${project}.git
          \""""
        }  // stage

        stage("${image_key}: Conan setup") {
          withCredentials([
            string(
              credentialsId: 'local-conan-server-password',
              variable: 'CONAN_PASSWORD'
            )
          ]) {
            sh """docker exec ${container_name} ${custom_sh} -c \"
              set +x
              conan remote add \
                --insert 0 \
                ${conan_remote} ${local_conan_server}
              conan user \
                --password '${CONAN_PASSWORD}' \
                --remote ${conan_remote} \
                ${conan_user} \
                > /dev/null
            \""""
          }  // withCredentials
        }  // stage

        stage("${image_key}: Package") {
          sh """docker exec ${container_name} ${custom_sh} -c \"
            cd ${project}
            conan create . ${conan_user}/${conan_pkg_channel} \
              --settings jemalloc:build_type=Release \
              --options jemalloc:shared=False \
              --build=outdated
          \""""

          sh """docker exec ${container_name} ${custom_sh} -c \"
            cd ${project}
            conan create . ${conan_user}/${conan_pkg_channel} \
              --settings jemalloc:build_type=Release \
              --options jemalloc:shared=True \
              --build=outdated
          \""""
        }  // stage

        stage("${image_key}: Upload") {
          sh """docker exec ${container_name} ${custom_sh} -c \"
            upload_conan_package.sh ${project}/conanfile.py \
              ${conan_remote} \
              ${conan_user} \
              ${conan_pkg_channel}
          \""""
        }  // stage
      } finally {
        sh "docker stop ${container_name}"
        sh "docker rm -f ${container_name}"
      }  // finally
    }  // node
  }  // return
}  // def

def get_macos_pipeline() {
  return {
    node('macos') {
      cleanWs()
      dir("${project}") {
        stage("macOS: Checkout") {
          checkout scm
        }  // stage

        stage("macOS: Conan setup") {
          withCredentials([
            string(
              credentialsId: 'local-conan-server-password',
              variable: 'CONAN_PASSWORD'
            )
          ]) {
            sh "conan user \
              --password '${CONAN_PASSWORD}' \
              --remote ${conan_remote} \
              ${conan_user} \
              > /dev/null"
          }  // withCredentials
        }  // stage

        stage("macOS: Package") {
          sh "conan create . ${conan_user}/${conan_pkg_channel} \
            --settings jemalloc:build_type=Release \
            --options jemalloc:shared=False \
            --build=outdated"

          sh "conan create . ${conan_user}/${conan_pkg_channel} \
            --settings jemalloc:build_type=Release \
            --options jemalloc:shared=True \
            --build=outdated"
        }  // stage

        stage("macOS: Upload") {
          sh "upload_conan_package.sh conanfile.py \
            ${conan_remote} \
            ${conan_user} \
            ${conan_pkg_channel}"
        }  // stage
      }  // dir
    }  // node
  }  // return
}  // def

node {
  checkout scm

  def builders = [:]
  for (x in images.keySet()) {
    def image_key = x
    builders[image_key] = get_pipeline(image_key)
  }
  builders['macOS'] = get_macos_pipeline()
  parallel builders

  // Delete workspace when build is done.
  cleanWs()
}
