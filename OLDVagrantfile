Vagrant.configure("2") do |config|
  config.vm.box_check_update = false

  # === FIREWALL / ROUTER ===
  config.vm.define "fw" do |fw|
    fw.vm.box = "bento/ubuntu-18.04"
    fw.vm.hostname = "fw"

    # Three internal networks: internal, dmz, attacker
    fw.vm.network "private_network", ip: "10.0.0.1", virtualbox__intnet: "net-int"
    fw.vm.network "private_network", ip: "10.0.1.1", virtualbox__intnet: "net-dmz"
    fw.vm.network "private_network", ip: "10.0.2.1", virtualbox__intnet: "net-attacker"

    fw.vm.provider :virtualbox do |vb|
      vb.customize ["modifyvm", :id, "--memory", "512"]
      vb.customize ["modifyvm", :id, "--cpus", "1"]
    end

    fw.vm.provision "shell", inline: <<-SHELL
      apt-get update -y
      apt-get install -y iptables-persistent net-tools

      echo 1 > /proc/sys/net/ipv4/ip_forward

      # Flush old rules
      iptables -F
      iptables -t nat -F
      iptables -X

      # Default policy DROP
      iptables -P INPUT DROP
      iptables -P FORWARD DROP
      iptables -P OUTPUT ACCEPT

      # Allow loopback and established
      iptables -A INPUT -i lo -j ACCEPT
      iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

      # Allow SSH
      iptables -A INPUT -p tcp --dport 22 -j ACCEPT

      # === Forwarding rules ===
      # DMZ web (10.0.1.20) -> DB (10.0.0.10:3306)
      iptables -A FORWARD -s 10.0.1.20 -d 10.0.0.10 -p tcp --dport 3306 -m state --state NEW,ESTABLISHED -j ACCEPT
      iptables -A FORWARD -s 10.0.0.10 -d 10.0.1.20 -p tcp --sport 3306 -m state --state ESTABLISHED -j ACCEPT

      # Attacker (10.0.2.0/24) -> DMZ only
      iptables -A FORWARD -s 10.0.2.0/24 -d 10.0.1.0/24 -j ACCEPT

      # DNAT: Host port 8080 -> DMZ web (10.0.1.20:80)
      iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 10.0.1.20:80
      iptables -t nat -A POSTROUTING -s 10.0.1.0/24 -o eth0 -j MASQUERADE

      # Save rules
      netfilter-persistent save
    SHELL
  end

  # === DATABASE (INTERNAL) ===
  config.vm.define "db" do |db|
    db.vm.box = "bento/ubuntu-18.04"
    db.vm.hostname = "db"
    db.vm.network "private_network", ip: "10.0.0.10", virtualbox__intnet: "net-int"

    db.vm.provider :virtualbox do |vb|
      vb.customize ["modifyvm", :id, "--memory", "512"]
      vb.customize ["modifyvm", :id, "--cpus", "1"]
    end

    db.vm.provision "shell", inline: <<-SHELL
      apt-get update -y
      DEBIAN_FRONTEND=noninteractive apt-get install -y mysql-server
      systemctl enable mysql
      sed -i "s/^bind-address.*/bind-address = 0.0.0.0/" /etc/mysql/mysql.conf.d/mysqld.cnf
      systemctl restart mysql
    SHELL
  end

  # === WEB SERVER (DMZ) ===
  config.vm.define "web" do |web|
    web.vm.box = "bento/ubuntu-18.04"
    web.vm.hostname = "web"
    web.vm.network "private_network", ip: "10.0.1.20", virtualbox__intnet: "net-dmz"

    web.vm.provider :virtualbox do |vb|
      vb.customize ["modifyvm", :id, "--memory", "512"]
      vb.customize ["modifyvm", :id, "--cpus", "1"]
    end

    web.vm.provision "shell", inline: <<-SHELL
      apt-get update -y
      apt-get install -y apache2 php php-mysql snort

      systemctl enable apache2
      systemctl restart apache2
    SHELL
  end
end
