# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  # Configurações globais para todas as VMs
  config.vm.box = "debian/bookworm64"
  config.ssh.insert_key = false
  
  if Vagrant.has_plugin?("vagrant-vbguest")
    config.vbguest.auto_update = false
  end

  config.vm.provider "virtualbox" do |vb|
    vb.linked_clone = true
    vb.check_guest_additions = false
  end

  # Gatilho para desabilitar o DHCP do VirtualBox
  config.trigger.before :up do |trigger|
    trigger.info = "Desabilitando DHCP do VirtualBox..."
    trigger.run = {inline: "vboxmanage dhcpserver stop --interface vboxnet0"}
  end

  # Servidor de Arquivos (arq)
  config.vm.define "arq" do |arq|
    arq.vm.hostname = "arq.samuel.vinicius.devops"
    arq.vm.network "private_network", ip: "192.168.56.102" # Substitua XX
    arq.vm.provider "virtualbox" do |vb|
      vb.memory = "512"
    end
    # Criando 3 discos adicionais de 10GB
    (1..3).each do |i|
      arq.vm.disk :disk, size: "10GB", name: "disk-#{i}"
    end
  end

  # Servidor de Banco de Dados (db)
  config.vm.define "db" do |db|
    db.vm.hostname = "db.samuel.vinicius.devops"
    # MAC configurado para o servidor DHCP atribuir IP estático
    db.vm.network "private_network", type: "dhcp", mac: "080027000001" 
    db.vm.provider "virtualbox" do |vb|
      vb.memory = "512"
    end
  end

  # Servidor de Aplicação (app)
  config.vm.define "app" do |app|
    app.vm.hostname = "app.samuel.vinicius.devops"
    app.vm.network "private_network", type: "dhcp", mac: "080027000002"
    app.vm.provider "virtualbox" do |vb|
      vb.memory = "512"
    end
  end

  # Host Cliente (cli)
  config.vm.define "cli" do |cli|
    cli.vm.hostname = "cli.samuel.vinicius.devops"
    cli.vm.network "private_network", type: "dhcp"
    cli.vm.provider "virtualbox" do |vb|
      vb.memory = "1024" # Memória de 1024 MB
    end
  end
end
