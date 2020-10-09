# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'yaml'
require 'ipaddr'
require 'pp'
include Vagrant::Plugin::V2

Vagrant.require_version ">= 1.9.2"


unless File.exist?("vagrant.config.yaml")
  puts 'vagrant.config.yaml file not found. please copy and edit vagrant.config.yaml.dist'
  abort
end

cluster_config = YAML.load_file("vagrant.config.yaml")

subnet = IPAddr.new cluster_config["subnet"]

cluster = []

def generate_nodes_config(node_role, node_config, node_domain)
  nodes = []

  if node_config.key?("num")
    num = node_config["num"]
  else
    num = 1
  end

  (1..num).each do |i|
    node = node_config.dup
    node["role"] = node_role
    node["name"] = "%s%02d" % [node_role, i]
    node["fqdn"] = node["name"] + "." + node_domain
    nodes.push(node)
  end
  return nodes
end

def convert_win_paths(paths)
  paths.map! { |path| Vagrant::Util::Platform.format_windows_path(path, :disable_unc) }
end


if cluster_config["nodes_mode"] == "multi"
  if cluster_config["etcd_mode"] == "external"
    cluster += generate_nodes_config("etcd", cluster_config["etcd"], cluster_config["domain"])
  end
  cluster += generate_nodes_config("service", cluster_config["service"], cluster_config["domain"])
  cluster += generate_nodes_config("master", cluster_config["master"], cluster_config["domain"])
  cluster += generate_nodes_config("node", cluster_config["node"], cluster_config["domain"])
else
  cluster += generate_nodes_config("singlenode", cluster_config["singlenode"], cluster_config["domain"])
end

num_nodes = cluster.length()
cpus = 0
mem = 0

(0..num_nodes - 1).each do |i|
  cpus += cluster[i]["cpu"]
  mem += cluster[i]["mem"]
end

puts("Resources: Cpus: %-5d Memory: %-5d Nodeds: %-5d" % [cpus, mem, num_nodes])

ansible_hosts = {
    "service" => {
        "hosts" => {},
        "vars" => {},
    },

    "etcd" => {
        "children" => {
            "master" => {}
        }
    },

    "master" => {
        "hosts" => {},
        "vars" => {}
    },

    "kube" => {
        "children" => {
            "master" => {},
            "node" => {},
            "service" => {},
        }
    },

    "node" => {
        "hosts" => {},
        "vars" => {},
    },

    "singlenode" => {
        "hosts" => {},
        "vars" => {},
    },

    "local" => {
        "hosts" => {
            "localhost" => {}
        },
        "vars" => {
            "ansible_connection" => "local"
        }
    }
}
ansible_vars = {}
ssh_config = ""
Vagrant.configure(2) do |config|

  required_plugins = %w(vagrant-vbguest)

  # Install plugins if missing
  plugins_to_install = required_plugins.select { |plugin| not Vagrant.has_plugin? plugin }
  if plugins_to_install.any?
    puts "Installing plugins: #{plugins_to_install.join(' ')}"
    if system "vagrant plugin install #{plugins_to_install.join(' ')}"
      exec "vagrant #{ARGV.join(' ')}"
    else
      abort "Installation of one or more plugins has failed. Aborting."
    end
  end

  if Vagrant.has_plugin?("vagrant-vbguest")
    config.vbguest.auto_update = false
  end

  config.ssh.insert_key = false


  (0..num_nodes - 1).each do |i|
    config.vm.define vm_name = cluster[i]["name"] do |node|

      role = cluster[i]["role"]
      fqdn = cluster[i]["fqdn"]
      ip = IPAddr.new(subnet.to_i() + cluster_config["ip_offset"] + i, Socket::AF_INET).to_s()

      node.vm.hostname = fqdn
      node.vm.box = cluster_config["distro"]
      node.vm.network "private_network", ip: ip
      node.vm.provider "virtualbox" do |vb|
        vb.cpus = cluster[i]["cpu"]
        vb.memory = cluster[i]["mem"]
      end
      node.vm.synced_folder '.', '/vagrant', disabled: true

      if cluster_config[role].key?("ports") and cluster_config[role]["ports"].length() > 0
        cluster_config[role]["ports"].each do |port|
          node.vm.network "forwarded_port", guest: port, host: port, protocol: "tcp"
        end
      end

      ansible_hosts[role]["hosts"][fqdn] = {
          "ansible_host" => fqdn,
          "private_dns" => fqdn,
          "private_ip" => ip
      }

      ssh_config += "Host %s %s
      HostName %s
      User vagrant
      Port 22
      UserKnownHostsFile /dev/null
      StrictHostKeyChecking no
      PasswordAuthentication no
      IdentityFile ~/.vagrant.d/insecure_private_key
      IdentitiesOnly yes
      LogLevel FATAL
" % [cluster[i]["fqdn"], cluster[i]["name"], ip]
      if i == num_nodes - 1
        ansible_vars = {
            "cluster" => "vagrant"
        }
        f = File.open("clusters/vagrant/hosts.yml", "w") { |f| f.write ansible_hosts.to_yaml }
        f = File.open("clusters/vagrant/ssh.config", "w") { |f| f.write ssh_config }
        f = File.open("clusters/vagrant/vars.yml", "w") { |f| f.write ansible_vars.to_yaml }
        node.vm.provision "ansible" do |ansible|
          ansible.playbook = "install.yml"
          ansible.inventory_path = "clusters/vagrant/hosts.yml"
          ansible.become = true
          ansible.limit = "all,localhost"
          ansible.host_key_checking = false
          ansible.config_file = "ansible.cfg"
          # Attention: The ansible provisioner does not support whitespace characters in raw_arguments
          ansible.raw_arguments = ["--forks=#{num_nodes}",
                                   "--flush-cache",
                                   "-e ansible_become_pass=vagrant",
                                   "--ssh-extra-args=-Fclusters/vagrant/ssh.config",
                                   "--extra-vars=@clusters/vagrant/vars.yml"
          ]
        end

      end

    end
  end
end

