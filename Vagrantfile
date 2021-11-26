# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'yaml'

Vagrant.require_version '>= 1.8.0'
ENV['LC_ALL'] = 'en_US.UTF-8'

vagrant_config = YAML.load_file('config.yaml')

ip_base = vagrant_config['infrast']['ip_base']
apiserver_vip, apiserver_dest_port  = "#{ip_base}100", "8443"
control_plane_endpoint = "#{apiserver_vip}:#{apiserver_dest_port}"
api_servers, minions = [], []

(1..vagrant_config['kubernetes']['masters']).each do |i| 
  api_servers.append({"name" => "k-m-#{i}", "ip" => "#{ip_base}#{10+i}","port":6443})
end
(1..vagrant_config['kubernetes']['minions']).each do |i| 
  minions.append({ "name" => "k-n-#{i}", "ip" => "#{ip_base}#{100+i}"})
end

puts '#########################################################################################'
puts '#'
puts "# ==> Starting HA Kubernetes stacked cluster based on box: #{vagrant_config['local']['box']} using Ansible !"
puts "# ==> Using ip addresses #{ip_base}x, #{vagrant_config['infrast']['masters']} masters and #{vagrant_config['infrast']['minions']} minions."
puts "# ==> KeepAlived with VIP = #{apiserver_vip} on master#1 and master#2, "
puts "# ==> control_plane_endpoint = #{control_plane_endpoint} "
puts '# ==> If minions==0 and you want to be able to schedule Pods on the control-plane node, please taint the nodes manually by: '
puts 'kubectl taint nodes --all node-role.kubernetes.io/master-'
puts '#########################################################################################'

Vagrant.configure('2') do |config|
  config.vm.box = vagrant_config['local']['box']
  config.vm.box_check_update = false

  #config.vm.synced_folder './', '/vagrant', type: 'nfs', nfs_version: 4,  nfs_udp: false

  (1..api_servers.length).each do |i|
    config.vm.define vm_name=api_servers[i-1]['name'] do |v|
      v.vm.hostname = vm_name
      ["libvirt","virtualbox"].each do |p|
        v.vm.provider p do |lb|
          lb.cpus = vagrant_config['infrast']['master_cpu']
          lb.memory = vagrant_config['infrast']['master_mem'] 
        end
      end

      v.vm.network "private_network", ip: api_servers[i-1]['ip']

      ha = {}
      ha['isHANode'] = (i == 1 or i == 2) ? true : false
      if ha['isHANode'] == true
        ha['state'] = (i == 1) ? "MASTER" : "BACKUP"
        ha['haproxy_version '] = vagrant_config['kubernetes']['haproxy_version ']
        ha['keepalived_version '] = vagrant_config['kubernetes']['keepalived_version ']
        ha['api_servers'] = api_servers
        ha['configured'] = false
      end

      v.vm.provision 'ansible' do |a|
        a.verbose = 'v'
        a.playbook = 'kubernetes-setup/k8s.yaml'
        a.extra_vars = {
          KUBERNETES_VERSION: vagrant_config['kubernetes']['version'],
          CALICO_VERSION: vagrant_config['kubernetes']['calico_version'],
          APISERVER_VIP: apiserver_vip,
          APISERVER_DEST_PORT: apiserver_dest_port,
          CONTROL_PLANE_ENDPOINT: control_plane_endpoint,
          ANSIBLE_WAITFOR_TIMEOUT: vagrant_config['ansible']['waitfor_seconds'],
          isControlplane: true,
          ha: ha
        }
      end
    end
  end  

  (1..minions.length).each do |i|
    config.vm.define vm_name= minions[i-1]['name'] do |v|
      v.vm.hostname = vm_name
      ["libvirt","virtualbox"].each do |p|
        v.vm.provider p do |lb|
          lb.cpus = vagrant_config['infrast']['node_cpu']
          lb.memory = vagrant_config['infrast']['node_mem'] 
        end
      end

      v.vm.network "private_network", ip: minions[i-1]['ip']

      v.vm.provision 'ansible' do |a|
        a.verbose = 'v'
        a.playbook = 'kubernetes-setup/k8s.yaml'

        a.extra_vars = {
          KUBERNETES_VERSION: vagrant_config['kubernetes']['version'],
          CONTROL_PLANE_ENDPOINT: control_plane_endpoint,
          ANSIBLE_WAITFOR_TIMEOUT: vagrant_config['ansible']['waitfor_seconds'],
        }
      end
    end
  end  
end
