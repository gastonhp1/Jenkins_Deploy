pipeline {
    agent any
    parameters {
        choice(
            name: 'Action',
            choices: ['Build', 'Destroy'],
            description: 'The action to take'
        )
        string(name: 'NODENAME', defaultValue: 'node', description: 'virtual machine hostname',)
        string(name: 'POOLPATH', defaultValue: '/var/kvm/virtual-machines', description: 'virtual machine KVM path',)
        string(name: 'REMOTE_ISO', defaultValue: 'https://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img', description: 'virtual machine remote image',)
        string(name: 'MEMORY', defaultValue: '2048', description: 'virtual machine memory size',)
        string(name: 'VCPU', defaultValue: '4', description: 'virtual machine vcpus',)
    }
    environment {
            GIT_REPO_URL = "https://github.com/gastonhp1/libvirt_VM.git"
            LIBVIRT_DEFAULT_URI = "qemu:///system"
            TF_PLAN_FILE = "libvirt.tfplan"
            BUILD_ID = currentBuild.number.toString()
            CREDENTIALS_ID = "${SSH_USERNAME}"
            TF_VAR_HOSTNAME = "${NODENAME}"
            TF_VAR_POOLPATH = "${POOLPATH}"
            TF_VAR_REMOTE_ISO = "${REMOTE_ISO}"
            TF_VAR_MEMORY = "${MEMORY}"
            TF_VAR_VCPU = "${VCPU}"
            TF_VAR_SSH_CREDENTIALS = credentials('SSH_CREDENTIALS')
            TF_VAR_SSH_AUTHORIZED_KEY = credentials('SSH_AUTHORIZED_KEY')
            }
    stages {
        stage("Checkout SCM") {
            steps {
                git branch: "main", url: env.GIT_REPO_URL
            }
        }
        stage("Setup virtual machine settings") {
            steps {
                script {
                    echo "Setup hostname in cloud-config.yml"
                    def cloudInitContent = """
#cloud-config
# vim: syntax
instance-id: ${TF_VAR_HOSTNAME}-${BUILD_ID}
local-hostname: ${TF_VAR_HOSTNAME}
user: ${TF_VAR_SSH_CREDENTIALS_USR}
password: ${TF_VAR_SSH_CREDENTIALS_PSW}
chpasswd: {expire: false}
ssh_pwauth: True
ssh_authorized_keys:
  - ${TF_VAR_SSH_AUTHORIZED_KEY}
"""
                    writeFile file: "${WORKSPACE}/config/cloud_init.yml", text: cloudInitContent
                    sh "cat ${WORKSPACE}/config/cloud_init.yml"
                }
            }
        }
        stage("Terraform Init") {
            steps {
                script {
                    sh "export LIBVIRT_DEFAULT_URI=${env.LIBVIRT_DEFAULT_URI}"
                    sh "terraform init"
                }
            }
        }
        stage("Terraform Plan") {
            steps {
                script {
                    sh "terraform plan -var='build_id=${BUILD_ID}' -out=${env.TF_PLAN_FILE}"
                }
            }
        }
        stage("Terraform Action") {
            steps {
                script {
                    if (params.Action == 'Build') {
                        sh """
                        terraform apply \
                            -var='vmname=${TF_VAR_HOSTNAME}' \
                            -var='poolpath=${TF_VAR_POOLPATH}' \
                            -var='remote_iso=${TF_VAR_REMOTE_ISO}' \
                            -var='mem=${TF_VAR_MEMORY}' \
                            -var='vcpu=${TF_VAR_VCPU}' \
                            -var='ssh_username=$TF_VAR_SSH_CREDENTIALS_USR' \
                            -var='password=$TF_VAR_SSH_CREDENTIALS_PSW' \
                            -var='build_id=${BUILD_ID}' \
                            --lock=false \
                            -auto-approve
                        """
                    } else if (params.Action == 'Destroy') {
                        sh "terraform destroy -var='build_id=${BUILD_ID}' --lock=false -auto-approve"
                    } else {
                        error "Invalid value for Action: ${params.Action}"
                    }
                }
            }
        }
    }
}