# -*- mode: ruby -*-
# vi: set ft=ruby :

MACHINES = {
  :myhost => {
    :box_name => "centos/7",
    :ip_addr => '192.168.56.150'
  }
}

Vagrant.configure("2") do |config|

#  config.vm.provision "ansible" do |ansible|
#    ansible.verbose = "vvv"
#    ansible.playbook = "./playbook.yml"
#    ansible.become = "true"
#  end

  MACHINES.each do |boxname, boxconfig|

    config.vm.define boxname do |box|

      box.vm.box = boxconfig[:box_name]
      box.vm.host_name = boxname.to_s

      box.vm.network "private_network", ip: boxconfig[:ip_addr]

      box.vm.provider :virtualbox do |vb|
        vb.customize ["modifyvm", :id, "--memory", "256"]
      end
          
      box.vm.provision "shell", inline: <<-SHELL
        mkdir -p ~root/.ssh; cp ~vagrant/.ssh/auth* ~root/.ssh
        sed -i '65s/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config
        systemctl restart sshd

        # Create group admin
        groupadd admin
        # Create admin-user adminik and user userik
        useradd adminik && echo "Otus1234" | passwd --stdin adminik && usermod -aG admin adminik; \
        useradd userik && echo "Otus1234" | passwd --stdin userik

        # Add pam rules
        echo '*;*;userik;!Wd0000-2400' >> /etc/security/time.conf
        sed -i '/account    required     pam_nologin.so/a account    required     pam_time.so' /etc/pam.d/sshd

        #sed -i '/account    required     pam_nologin.so/a account    required     pam_exec.so /usr/local/bin/test_login.sh' /etc/pam.d/sshd
        #cat <<'EOF' >> /usr/local/bin/test_login.sh
        #!/bin/bash
        #if [ $PAM_USER = "userik" ]; then
        #  if [ $(date +%a) = "Sat" ] || [ $(date +%a) = "Sun" ]; then
        #    exit 1
        #  else
        #    exit 0
        #  fi
        #fi
        #EOF
        #chmod +x /usr/local/bin/test_login.sh

        # Install and start docker
        yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
        yum install docker-ce docker-ce-cli containerd.io -y
        systemctl start docker
        systemctl enable docker
        # Create dockeruser and add to docker group
        useradd dockeruser && echo "Otus1234" | passwd --stdin dockeruser && usermod -aG docker dockeruser
        # Add rule for dockeruser for restart docker service
        cat <<'EOF'>> /etc/polkit-1/rules.d/01-docker.rules
        polkit.addRule(function(action, subject) {
          if (action.id == "org.freedesktop.systemd1.manage-units" && action.lookup("unit") == "docker.service" && action.lookup("verb") == "restart" && subject.user == "dockeruser") {
            return polkit.Result.YES;
          }
        });
        EOF
      SHELL
    end
  end
end
