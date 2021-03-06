# -*- mode: ruby -*-
# vi: set ft=ruby :

# The Docker registry from where we pull the OpenShift Docker image
DOCKER_REGISTRY="docker.io"

# The name of the OpenShift image available on dockerhub.
IMAGE_NAME="openshift/origin"
# Tag of the OpenShift image available on dockerhub.
#IMAGE_TAG="v1.2.0"
IMAGE_TAG="v1.3.0-alpha.2"

# The public IP address the VM created by Vagrant will get.
# You will use this IP address to connect to OpenShift web console.
PUBLIC_ADDRESS="10.1.2.2"

REQUIRED_PLUGINS = %w(vagrant-service-manager)
errors = []

def message(name)
  "#{name} plugin is not installed, run `vagrant plugin install #{name}` to install it."
end
# Validate and collect error message if plugin is not installed
REQUIRED_PLUGINS.each { |plugin| errors << message(plugin) unless Vagrant.has_plugin?(plugin) }
unless errors.empty?
  msg = errors.size > 1 ? "Errors: \n* #{errors.join("\n* ")}" : "Error: #{errors.first}"
  fail Vagrant::Errors::VagrantError.new, msg
end

# The following steps are executed to provision OpenShift Origin:
# 1. Create a private network and set the Vagrant Box IP to *10.1.2.2*. If you want a different IP then change the **PUBLIC_ADDRESS** variable.
# 2. Pull latest the *openshift/origin* container from docker hub and tag it.
# 3. Provide required configuration details for OpenShift web console and API.
# For More info about Openshift, please refer to `offical documents
# <https://docs.openshift.org/latest/welcome/index.html>`_.

Vagrant.configure(2) do |config|
  config.vm.box = 'projectatomic/adb'

  config.vm.provider "virtualbox" do |v, override|
    v.memory = 5120
    v.cpus   = 4
    v.customize ["modifyvm", :id, "--cpus", "4"]
    v.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
  end

  config.vm.provider "libvirt" do |v, override|
    v.driver = "kvm"
    v.memory = 5120
    v.cpus   = 4
  end

  config.vm.provider "openstack" do |os|
    # Specify OpenStack authentication information
    os.openstack_auth_url = ENV['OS_AUTH_URL']
    os.tenant_name = ENV['OS_TENANT_NAME']
    os.username = ENV['OS_USERNAME']
    os.password = ENV['OS_PASSWORD']

    # Specify instance information
    os.server_name = ENV['OS_INSTANCE']
    os.flavor = ENV['OS_FLAVOR']
    os.image = ENV['OS_IMAGE']
    os.floating_ip_pool = ENV['OS_FLOATING_IP_POOL']

    os.security_groups = ['default', ENV['OS_SECURITY_GROUP']]
    os.keypair_name = ENV['OS_KEYPAIR_NAME']
    config.ssh.private_key_path = ENV['OS_PRIVATE_KEYPATH']
    config.ssh.username = 'vagrant'
  end

  config.vm.network "private_network", ip: "#{PUBLIC_ADDRESS}"

  config.vm.provision "shell", run: "always", inline: <<-SHELL
#    sudo yum update -y --nogpgcheck adb-utils 1>/dev/null
    sudo yum install -y http://cbs.centos.org/kojifiles/work/tasks/7131/97131/adb-utils-1.8.1-1.el7.noarch.rpm 1>/dev/null
    sudo sed -i.orig -e "s/METRICS=false/METRICS=true/" /etc/sysconfig/openshift_option

    sudo curl -s -L https://github.com/tnozicka/adb-utils/raw/update-jenkins-next-templates/services/openshift/templates/adb/jenkins-ephemeral-next-template.json > /opt/adb/openshift/templates/adb/jenkins-ephemeral-template.json
    sudo sed 's/"name": "jenkins-ephemeral-next"/"name": "jenkins"/' /opt/adb/openshift/templates/adb/jenkins-ephemeral-template.json > /opt/adb/openshift/templates/adb/jenkins-template.json

    sudo curl -s -L https://github.com/redhat-kontinuity/catapult/raw/master/catapult_os_template.json > /opt/adb/openshift/templates/adb/catapult-template.json

    sudo DOCKER_REGISTRY=#{DOCKER_REGISTRY} IMAGE_TAG=#{IMAGE_TAG} IMAGE_NAME=#{IMAGE_NAME} /usr/bin/sccli openshift
  SHELL

  config.vm.provision "shell", inline: <<-SHELL
    sudo setsebool -P virt_sandbox_use_fusefs 1
  SHELL

  config.vm.provision "shell", run: "always", privileged: false, inline: <<-SHELL
    oc delete project sample-project 1>/dev/null 2>&1
    oc new-project catapult --display-name="Catapult" 1>/dev/null

    echo "Please visit https://hawcular-metrics.centos7-adb.#{PUBLIC_ADDRESS}.xip.io/ (or any other https site inside cluster) and accept the ssl certificate for the metrics to work in console."
    echo
    echo "You can now access OpenShift console on: https://#{PUBLIC_ADDRESS}:8443/console"
    echo
    echo "Configured basic user: openshift-dev, Password: devel"
    echo
    echo "Configured cluster admin user: admin, Password: admin"
    echo
    echo "To use OpenShift CLI, run:"
    echo "$ vagrant ssh"
    echo "$ oc ..."
    echo
    echo "To browse the OpenShift API documentation, follow this link:"
    echo "http://openshift3swagger-claytondev.rhcloud.com"
    echo
    echo "Then enter this URL:"
    echo https://#{PUBLIC_ADDRESS}:8443/swaggerapi/oapi/v1
    echo "."
    echo
    echo "We will now pull some builder images to make your experience smoother for the first time."
    docker pull docker.io/redhatdevelopers/catapult
    docker pull docker.io/openshift/jenkins-1-centos7:dev
    docker pull docker.io/redhatdistortion/maven-jenkins-slave
    docker pull registry.access.redhat.com/jboss-eap-7/eap70-openshift
    echo "Done pulling images"
  SHELL
end
