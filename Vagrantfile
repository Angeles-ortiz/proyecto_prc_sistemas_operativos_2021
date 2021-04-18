BOX_NAME = "bento/ubuntu-20.04"
RAM = "512"
CPUS = 2
MANAGERS = 2
MANAGER_IP = "172.1.1.1"
WORKERS = 3
WORKER_IP = "172.1.1.10"

$install_docker_script = <<SCRIPT
echo "------ VERIFICANDO ACTUALIZACIONES ------"
sudo apt-get update
echo "------ INSTALANDO DOCKER  ------"
curl -sSL https://get.docker.com/ | sh
sudo usermod -aG docker vagrant
SCRIPT

$manager_script = <<SCRIPT
if [ "$#" -eq 1 ]; then
  echo "------ INICIANDO DOCKER SWARM ------"
  echo Arguments: $*
  docker swarm init --listen-addr $1:2377 --advertise-addr $1:2377
  docker swarm join-token -q manager > /vagrant/swarm-token-manager
  docker swarm join-token -q worker > /vagrant/swarm-token-worker
  echo "------ INSTALANDO PORTAINER ------"
  curl -L https://downloads.portainer.io/portainer-agent-stack.yml -o portainer-agent-stack.yml
  docker stack deploy --compose-file=portainer-agent-stack.yml portainer
else
   echo "OCURRIO UN ERROR EN LOS PARAMETROS"
fi
SCRIPT

$node_script = <<SCRIPT
echo "Arguments: $*"
if [ "$#" -eq 2 ]; then
   echo "------ HACIENDO JOIN CON DOCKER SWARM ------"
   docker swarm join --token `cat /vagrant/swarm-token-$1` $2:2377
else
   echo "OCURRIO UN ERROR EN LOS PARAMETROS"
fi
SCRIPT

first_manager_ip = ""



Vagrant.configure("2") do |config|
  config.vm.box = BOX_NAME
    config.vm.synced_folder ".", "/vagrant"
    config.vm.provision "shell",inline: $install_docker_script, privileged: true
    config.vm.provider "virtualbox" do |vb|
      vb.memory = RAM
      vb.cpus = CPUS
    end
    
    (1..MANAGERS).each do |i|
        config.vm.define "manager0#{i}" do |manager|
          manager.vm.network :private_network, ip: "#{MANAGER_IP}#{i}"
          manager.vm.hostname = "manager0#{i}"
          if first_manager_ip.to_s.empty?
            manager.vm.network :forwarded_port, guest: 8080, host: 9501
            manager.vm.network :forwarded_port, guest: 5000, host: 9502
            manager.vm.network :forwarded_port, guest: 9000, host: 9503
            first_manager_ip = "#{MANAGER_IP}#{i}"
            manager.vm.provision "shell", inline: $manager_script, :args => ["#{first_manager_ip}"], privileged: true
          else
            manager.vm.provision "shell", inline: $node_script,:args => ["manager", "#{first_manager_ip}"], privileged: true
          end
        end
    end
    #Setup Woker Nodes
    (1..WORKERS).each do |i|
        config.vm.define "worker0#{i}" do |worker|
            worker.vm.network :private_network, ip: "#{WORKER_IP}#{i}"
            worker.vm.hostname = "worker0#{i}"
            worker.vm.provision "shell", inline: $node_script, :args => ["worker", "#{first_manager_ip}"], privileged: true
        end
    end
end