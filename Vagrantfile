# -*- mode: ruby -*-
# vi: set ft=ruby :

conf = {
# Generic parameters used across all ONAP components
  'public_net_id'       => '00000000-0000-0000-0000-000000000000',
  'key_name'            => 'ecomp_key',
  'pub_key'             => '',
  'nexus_repo'          => 'https://nexus.onap.org/content/sites/raw',
  'nexus_docker_repo'   => 'nexus3.onap.org:10001',
  'nexus_username'      => 'docker',
  'nexus_password'      => 'docker',
  'dmaap_topic'         => 'AUTO',
  'artifacts_version'   => '1.0.0',
  'docker_version'      => '1.0-STAGING-latest',
  'gerrit_branch'       => 'master',
# Parameters for DCAE instantiation
  'dcae_zone'           => 'iad4',
  'dcae_state'          => 'vi',
  'openstack_tenant_id' => '',
  'openstack_username'  => '',
  'openstack_api_key'   => '',
  'openstack_password'  => '',
  'nexus_repo_root'     => 'https://nexus.onap.org',
  'nexus_url_snapshot'  => 'https://nexus.onap.org/content/repositories/snapshots',
  'gitlab_branch'       => 'master',
  'build_image'         => 'True',
  'odl_version'         => '0.5.3-Boron-SR3',
  'compile_repo'        => 'False'
}

Vagrant.require_version ">= 1.8.6"

# Determine the OS for the host computer
module OS
    def OS.windows?
        (/cygwin|mswin|mingw|bccwin|wince|emx/ =~ RUBY_PLATFORM) != nil
    end

    def OS.mac?
        (/darwin/ =~ RUBY_PLATFORM) != nil
    end

    def OS.unix?
        !OS.windows?
    end

    def OS.linux?
        OS.unix? and not OS.mac?
    end
end

if OS.windows?
    puts "Vagrant launched from windows. This configuration has not fully tested."
end

# Determine the provider used
provider = (ENV['VAGRANT_DEFAULT_PROVIDER'] || :virtualbox).to_sym
puts "Using #{provider} provider"

vd_conf = ENV.fetch('VD_CONF', 'etc/settings.yaml')
if File.exist?(vd_conf)
  require 'yaml'
  user_conf = YAML.load_file(vd_conf)
  conf.update(user_conf)
end

deploy_mode = ENV.fetch('DEPLOY_MODE', 'individual')
sdc_volume='vol1-sdc-data.vdi'

Vagrant.configure("2") do |config|

  if ENV['http_proxy'] != nil and ENV['https_proxy'] != nil and ENV['no_proxy'] != nil
    if not Vagrant.has_plugin?('vagrant-proxyconf')
      system 'vagrant plugin install vagrant-proxyconf'
      raise 'vagrant-proxyconf was installed but it requires to execute again'
    end
    config.proxy.http     = ENV['http_proxy']
    config.proxy.https    = ENV['https_proxy']
    config.proxy.no_proxy = ENV['no_proxy']
  end

  config.vm.box = 'ubuntu/trusty64'
  if provider == :libvirt
    config.vm.box = 'sputnik13/trusty64'
  end
  if provider == :openstack
    config.vm.box = nil
    config.ssh.username = 'ubuntu'
    if not Vagrant.has_plugin?('vagrant-openstack-provider')
      system 'vagrant plugin install vagrant-openstack-provider'
      raise 'vagrant-openstack-provider was installed but it requires to execute again'
    end
  end
  #config.vm.provision "docker"
  config.vm.synced_folder './opt', '/opt/', create: true
  config.vm.synced_folder './lib', '/var/onap/', create: true
  config.vm.synced_folder '~/.m2', '/root/.m2/', create: true

  config.vm.provider :virtualbox do |v|
    v.customize ["modifyvm", :id, "--memory", 4 * 1024]
  end
  config.vm.provider :libvirt do |v|
    v.memory = 4 * 1024
    v.nested = true
  end
  config.vm.provider :openstack do |v|

    v.openstack_auth_url               = ENV.fetch('OS_AUTH_URL', '')
    v.tenant_name                      = ENV.fetch('OS_TENANT_NAME', '')
    v.username                         = ENV.fetch('OS_USERNAME', '')
    v.password                         = ENV.fetch('OS_PASSWORD', '')
    v.region                           = ENV.fetch('OS_REGION_NAME', '')
    v.identity_api_version             = ENV.fetch('OS_IDENTITY_API_VERSION', '')
    v.domain_name                      = ENV.fetch('OS_PROJECT_DOMAIN_ID', '')
    v.project_name                     = ENV.fetch('OS_PROJECT_NAME', '')

    v.floating_ip_pool                 = ENV.fetch('OS_FLOATING_IP_POOL', '')
    v.floating_ip_pool_always_allocate = (ENV['OS_FLOATING_IP_ALWAYS_ALLOCATE'] == 'true')
    v.image                            = ENV.fetch('OS_IMAGE', '')
    v.security_groups                  = [ENV.fetch('OS_SEC_GROUP', '')]
    v.flavor                           = 'm1.medium'
    v.networks                         = ENV.fetch('OS_NETWORK', '')
  end

  case deploy_mode

  when 'all-in-one'

    config.vm.define :all_in_one do |all_in_one|
      all_in_one.vm.hostname = 'all-in-one'
      all_in_one.vm.network :private_network, ip: '192.168.50.3'
      all_in_one.vm.provider "virtualbox" do |v|
        v.customize ["modifyvm", :id, "--memory", 12 * 1024]
        unless File.exist?(sdc_volume)
           v.customize ['createhd', '--filename', sdc_volume, '--size', 20 * 1024]
        end
        v.customize ['storageattach', :id, '--storagectl', 'SATAController', '--port', 1, '--device', 0, '--type', 'hdd', '--medium', sdc_volume]
      end
      all_in_one.vm.provider "libvirt" do |v|
        v.memory = 12 * 1024
        v.nested = true
        v.storage :file, path: sdc_volume, bus: 'sata', device: 'vdb', size: '2G'
      end
      all_in_one.vm.provider "openstack" do |v|
        v.server_name = 'all-in-one'
        v.flavor = 'm1.xlarge'
      end
      all_in_one.vm.provision 'shell' do |s|
        s.path = 'postinstall.sh'
        s.args = ['mr', 'sdc', 'aai', 'mso', 'robot', 'vid', 'sdnc', 'portal', 'dcae', 'policy', 'appc', 'vfc']
        s.env = conf
      end
    end

  when 'individual'

    config.vm.define :dns do |dns|
      dns.vm.hostname = 'dns'
      dns.vm.network :private_network, ip: '192.168.50.3'
      dns.vm.provider "virtualbox" do |v|
        v.customize ["modifyvm", :id, "--memory", 1 * 1024]
      end
      dns.vm.provider "libvirt" do |v|
        v.memory = 1 * 1024
        v.nested = true
      end
      dns.vm.provider "openstack" do |v|
        v.server_name = 'dns'
        v.flavor = 'm1.small'
      end
      dns.vm.provision 'shell' do |s|
        s.path = 'postinstall.sh'
        s.env = conf
      end 
    end

    config.vm.define :mr do |mr|
      mr.vm.hostname = 'message-router'
      mr.vm.network :private_network, ip: '192.168.50.4'
      mr.vm.provider "openstack" do |v|
        v.server_name = 'message-router'
      end
      mr.vm.provision 'shell' do |s|
        s.path = 'postinstall.sh'
        s.args = ['mr']
        s.env = conf
      end
    end

    config.vm.define :sdc do |sdc|
      sdc.vm.hostname = 'sdc'
      sdc.vm.network :private_network, ip: '192.168.50.5'
      sdc.vm.provider "virtualbox" do |v|
        unless File.exist?(sdc_volume)
           v.customize ['createhd', '--filename', sdc_volume, '--size', 20 * 1024]
        end
        v.customize ['storageattach', :id, '--storagectl', 'SATAController', '--port', 1, '--device', 0, '--type', 'hdd', '--medium', sdc_volume]
      end
      sdc.vm.provider "libvirt" do |v|
        v.storage :file, path: sdc_volume, bus: 'sata', device: 'vdb', size: '2G'
      end
      sdc.vm.provider "openstack" do |v|
        v.server_name = 'sdc'
      end
      sdc.vm.provision 'shell' do |s|
        s.path = 'postinstall.sh'
        s.args = ['sdc']
        s.env = conf
      end
    end
  
    config.vm.define :aai do |aai|
      aai.vm.hostname = 'aai'
      aai.vm.network :private_network, ip: '192.168.50.6'
      aai.vm.provider "openstack" do |v|
        v.server_name = 'aai'
      end
      aai.vm.provision 'shell' do |s| 
        s.path = 'postinstall.sh'
        s.args = ['aai']
        s.env = conf
      end 
    end
  
    config.vm.define :mso do |mso|
      mso.vm.hostname = 'mso'
      mso.vm.network :private_network, ip: '192.168.50.7'
      mso.vm.provider "openstack" do |v|
        v.server_name = 'mso'
      end
      mso.vm.provision 'shell' do |s| 
        s.path = 'postinstall.sh'
        s.args = ['mso']
        s.env = conf
      end 
    end
  
    config.vm.define :robot do |robot|
      robot.vm.hostname = 'robot'
      robot.vm.network :private_network, ip: '192.168.50.8'
      robot.vm.provider "openstack" do |v|
        v.server_name = 'robot'
      end
      robot.vm.provision 'shell' do |s|
        s.path = 'postinstall.sh'
        s.args = ['robot']
        s.env = conf
      end
    end
  
    config.vm.define :vid do |vid|
      vid.vm.hostname = 'vid'
      vid.vm.network :private_network, ip: '192.168.50.9'
      vid.vm.provider "openstack" do |v|
        v.server_name = 'vid'
      end
      vid.vm.provision 'shell' do |s|
        s.path = 'postinstall.sh'
        s.args = ['vid']
        s.env = conf
      end
    end
  
    config.vm.define :sdnc do |sdnc|
      sdnc.vm.hostname = 'sdnc'
      sdnc.vm.network :private_network, ip: '192.168.50.10'
      sdnc.vm.provider "openstack" do |v|
        v.server_name = 'sdnc'
      end
      sdnc.vm.provision 'shell' do |s|
        s.path = 'postinstall.sh'
        s.args = ['sdnc']
        s.env = conf
      end
    end
  
    config.vm.define :portal do |portal|
      portal.vm.hostname = 'portal'
      portal.vm.network :private_network, ip: '192.168.50.11'
      portal.vm.provider "openstack" do |v|
        v.server_name = 'portal'
      end
      portal.vm.provision 'shell' do |s|
        s.path = 'postinstall.sh'
        s.args = ['portal']
        s.env = conf
      end
    end
  
    config.vm.define :dcae do |dcae|
      dcae.vm.hostname = 'dcae'
      dcae.vm.network :private_network, ip: '192.168.50.12'
      dcae.vm.provider "openstack" do |v|
        v.server_name = 'dcae'
      end
      dcae.vm.provision 'shell' do |s|
        s.path = 'postinstall.sh'
        s.args = ['dcae']
        s.env = conf
      end
    end
  
    config.vm.define :policy do |policy|
      policy.vm.hostname = 'policy'
      policy.vm.network :private_network, ip: '192.168.50.13'
      policy.vm.provider "openstack" do |v|
        v.server_name = 'policy'
      end
      policy.vm.provision 'shell' do |s|
        s.path = 'postinstall.sh'
        s.args = ['policy']
        s.env = conf
      end
    end
  
    config.vm.define :appc do |appc|
      appc.vm.hostname = 'appc'
      appc.vm.network :private_network, ip: '192.168.50.14'
      appc.vm.provider "openstack" do |v|
        v.server_name = 'appc'
      end
      appc.vm.provision 'shell' do |s|
        s.path = 'postinstall.sh'
        s.args = ['appc']
        s.env = conf
      end
    end

    config.vm.define :vfc do |vfc|
      vfc.vm.hostname = 'vfc'
      vfc.vm.network :private_network, ip: '192.168.50.15'
      vfc.vm.provider "openstack" do |v|
        v.server_name = 'vfc'
      end
      vfc.vm.provision 'shell' do |s|
        s.path = 'postinstall.sh'
        s.args = ['vfc']
        s.env = conf
      end
    end

  when 'testing'

    config.vm.define :testing do |testing|
      test_suite = ENV.fetch('TEST_SUITE', '*')
      test_case = ENV.fetch('TEST_CASE', '*')

      testing.vm.hostname = 'testing'
      testing.vm.network :private_network, ip: '192.168.50.3'
      testing.vm.synced_folder './tests', '/var/onap_tests/', create: true
      testing.vm.provider "virtualbox" do |v|
        v.customize ["modifyvm", :id, "--memory", 2 * 1024]
      end
      testing.vm.provider "libvirt" do |v|
        v.memory = 2 * 1024
        v.nested = true
      end
      testing.vm.provider "openstack" do |v|
        v.server_name = 'testing'
        v.flavor      = 'm1.small'
      end
      testing.vm.provision 'shell' do |s|
        s.path = 'unit_testing.sh'
        s.args = [test_suite, test_case]
        s.env = conf
      end
    end

  end
end
