#number of VMs of each type to bring up
node_count = {
  "master" => 1,
  "slave" => 2
}
chef_json = {
  "master" => { "mesos" => { :master_ips => ["192.168.50.11"], "slave_ips" => ["192.168.50.12", "192.168.50.13"] } },
  "slave" => { "mesos" => { :master_ips => ["192.168.50.11"], "slave_ips" => ["192.168.50.12", "192.168.50.13"] } }
}
forwarded_ports = {
  "master" => [8080],
  "slave" => [9200]
}
run_lists = {
  "master" => %w| mesos::master |,
  "slave" => %w| mesos::slave |
}
private_network = "192.168.50.10"
hostname_base_number = 3000 #emulate openstack
host_port_forward_start_port = 8000

# do not edit
private_octets = private_network.split(".")
private_subnet = private_octets[0..2].join(".")
ip_last_octet = private_octets[3].to_i
host_port = host_port_forward_start_port

servers = Hash.new
node_count.each_key do |node_type|
  count = node_count[node_type]
  if count > 0
    servers[node_type] = Hash.new
    (1..count).each do |i|
      hostnum = hostname_base_number + i
      hostname = node_type + hostnum.to_s
      servers[node_type][hostname] = Hash.new
      servers[node_type][hostname]['fqdn'] = "local-mesos-vagrant-"+hostname+".dev.paypal.com"
      servers[node_type][hostname]['ip'] = private_subnet + "." + ip_last_octet.to_s
      ip_last_octet += 1
      if forwarded_ports.has_key?(node_type)
        servers[node_type][hostname]['forwarded_ports'] = Hash.new
        forwarded_ports[node_type].each do |guest_port|
          servers[node_type][hostname]['forwarded_ports'][host_port] = guest_port
          host_port += 1
        end
      end
      if run_lists.has_key?(node_type)
        servers[node_type][hostname]['run_list'] = run_lists[node_type]
      end
      if chef_json.has_key?(node_type)
        servers[node_type][hostname]['chef_json'] = chef_json[node_type]
      end
    end
  end
end

#Port mapping output
port_forwards = Hash.new
servers.each_key do |node_type|
  servers[node_type].each_key do |hostname|
    if servers[node_type][hostname].has_key?("forwarded_ports")
      servers[node_type][hostname]["forwarded_ports"].each_key do |host_port|
        port_forwards[host_port] = "#{hostname}:#{servers[node_type][hostname]['forwarded_ports'][host_port]}"
      end
    end
  end
end

if port_forwards.length > 0
  puts "The following port forwards from the host are enabled:"
  port_forwards.sort.each do |pair|
    puts "#{pair[0]} => #{pair[1]}"
  end
end

Vagrant.configure("2") do |config|
  config.vm.box     = "precise64"
  config.vm.box_url = "http://files.vagrantup.com/precise64.box"

  # Install Vagrant bershelf, Vagrant Omnibus and Vagrant cachier before running
  # vagrant plugin install vagrant-berkshelf
  # vagrant plugin install vagrant-cachier
  # vagrant plugin install vagrant-omnibus

  config.berkshelf.enabled = true
  config.omnibus.chef_version = :latest
  config.cache.auto_detect = true

  config.vm.provider :virtualbox do |vb|
    #vb.customize ["modifyvm", :id, "--memory", 2048]
    vb.customize ["modifyvm", :id, "--memory", 1024]
  end
  servers.each_key do |node_type|
    servers[node_type].each_key do |hostname|
      config.vm.define hostname do | server |
        server.vm.hostname = servers[node_type][hostname]['fqdn']
        server.vm.network "private_network", ip: servers[node_type][hostname]['ip']

        if servers[node_type][hostname].has_key?("forwarded_ports")
          servers[node_type][hostname]['forwarded_ports'].each_key do |host_port|
            server.vm.network "forwarded_port", guest: servers[node_type][hostname]['forwarded_ports'][host_port], host: host_port
          end
        end
        server.vm.provision "shell",
    		inline: "apt-get update"
        server.vm.provision :chef_solo do |chef|
          # chef.cookbooks_path = ["~/.berkshelf/cookbooks"]
          if servers[node_type][hostname].has_key?("run_list")
            chef.run_list = servers[node_type][hostname]['run_list']
          end
          if servers[node_type][hostname].has_key?("chef_json")
            chef.json = servers[node_type][hostname]['chef_json']
          else
            chef.json = {}
          end
          # Turn on verbose Chef logging if necessary
          chef.log_level      = :debug
        end
      end
    end
  end
end

