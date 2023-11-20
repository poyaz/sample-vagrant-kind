# -*- mode: ruby -*-
# vi: set ft=ruby :

SNAPSHOT_FIRST_INIT = "000-init"

REQUIRED_PLUGINS = %w(vagrant-libvirt)
exit unless REQUIRED_PLUGINS.all? do |plugin|
  Vagrant.has_plugin?(plugin) || (
    puts "The #{plugin} plugin is required. Please install it with:"
    puts "$ vagrant plugin install #{plugin}"
    false
  )
end

if "up".eql? ARGV[0] or ("snapshot".eql? ARGV[0] and "restore".eql? ARGV[1])
  ENV["VAGRANT_EXPERIMENTAL"] = "typed_triggers"
end

snapshot_machine_name_list = []
Vagrant.configure("2") do |config|
  config.trigger.after :"VagrantPlugins::ProviderLibvirt::Action::SnapshotRestore", type: :action do |t|
    t.run_remote = {
      inline:
        <<-SHELL
          sed -i -E "s/#?DNSSEC=.*/DNSSEC=no/g" /etc/systemd/resolved.conf
          systemctl restart systemd-resolved
          hwclock --hctosys
        SHELL
    }
  end

  config.trigger.after :up do |t|
    t.ruby do |env, machine|
      snapshot_machine_name_list << machine.name
    end
  end

  config.trigger.after :environment_unload, type: :hook do |t|
    t.ruby do |env, machine|
      snapshot_machine_name_list.each do |name|
        command = <<-SHELL
          vagrant snapshot list '#{name}' \
            | grep -q '#{SNAPSHOT_FIRST_INIT}' \
            || echo "Create first snapshot from machine '#{name}' with name '#{SNAPSHOT_FIRST_INIT}'" \
            && vagrant snapshot save '#{name}' '#{SNAPSHOT_FIRST_INIT}'
        SHELL
        system(command)
      end
    end
  end
  
  config.ssh.username = 'vagrant'
  config.ssh.password = 'vagrant'
  config.ssh.insert_key = false

  config.vm.define "kind" do |m|
    m.trigger.after :"VagrantPlugins::ProviderLibvirt::Action::SnapshotRestore", type: :action do |t|
      t.run_remote = {
        inline:
          <<-SHELL
            docker start $(docker ps -aq --filter label=io.x-k8s.kind.cluster)
          SHELL
      }
    end
      
    m.vm.box = "generic/ubuntu1804"
    m.vm.hostname = "kind"
    m.vm.network "private_network", ip: "192.168.33.30"
    
    m.vm.provider :libvirt do |vb|
      vb.memory = 5096
      vb.cpus = 3
  
#      vb.storage :file, :size => '10G', :path => 'kind_disk_1.img', :bus => 'scsi', :type => 'qcow2', :discard => 'unmap', :detect_zeroes => 'on'
    end

    m.vm.provision "file", source: "./vagrant-kind-config.yaml", destination: "/tmp/kind-config.yaml"
    m.vm.provision "shell" do |s|
      s.inline =
        <<-SHELL
          sed -i -E "s/#?DNSSEC=.*/DNSSEC=no/g" /etc/systemd/resolved.conf
          systemctl restart systemd-resolved
          hwclock --hctosys
          sed -i -e 's/us.archive.ubuntu.com/ua.archive.ubuntu.com/' /etc/apt/sources.list
    
          install -m 0755 -d /etc/apt/keyrings
          export DEBIAN_FRONTEND=noninteractive
    
          apt-get update
          apt-get install -y apt-transport-https ca-certificates curl gnupg dnsmasq
    
          echo "server=4.2.2.4\nserver=8.8.8.8" > /etc/dnsmasq.d/001-default.conf
          echo "server=/registry-1.docker.io/178.22.122.100\nserver=/registry.docker.io/178.22.122.100\nserver=/auth.docker.io/178.22.122.100" > /etc/dnsmasq.d/002-shecan.conf
          echo "server=/baltocdn.com/178.22.122.100" >> /etc/dnsmasq.d/002-shecan.conf
    
          systemctl enable dnsmasq
          systemctl restart dnsmasq
    
          sed -i -E "s/nameserver 127.0.0.53/nameserver 127.0.0.1/g" /etc/resolv.conf
    
          curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
          chmod a+r /etc/apt/keyrings/docker.gpg
          echo "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" \
            | tee /etc/apt/sources.list.d/docker.list > /dev/null
          apt-get update
          apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
    
          curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
          echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | tee /etc/apt/sources.list.d/kubernetes.list > /dev/null
          apt-get update
          apt-get install -y kubectl
    
          curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | tee /etc/apt/keyrings/helm-apt-keyring.gpg > /dev/null
          echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/helm-apt-keyring.gpg] https://baltocdn.com/helm/stable/debian/ all main" \
            | tee /etc/apt/sources.list.d/helm-stable-debian.list > /dev/null
          apt-get update
          apt-get install -y helm
    
          [ $(uname -m) = x86_64 ] && curl -Lo /usr/local/bin/kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
          chmod +x /usr/local/bin/kind
    
          kind create cluster --config /tmp/kind-config.yaml
    
          curl -sS https://webi.sh/k9s | sh
          source ~/.config/envman/PATH.env
        SHELL
    end
  end
end
