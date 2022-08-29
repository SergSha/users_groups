<h3>### USERS AND GROUPS ###</h3>

<h4>Описание домашнего задания</h4>

<p>Запретить всем пользователям, кроме группы admin логин в выходные (суббота и воскресенье), без учета праздников</p>
<ul>
<li>дать конкретному пользователю права работать с докером и возможность рестартить докер сервис</li>
</ul>

<h4># Создадим виртуальную машину myhost</h4>

<p>В домашней директории создадим директорию users_groups, в котором будут храниться настройки виртуальной машины:</p>

<pre>[user@localhost otus]$ mkdir ./users_groups
[user@localhost otus]$</pre>

<p>Перейдём в директорию users_groups:</p>

<pre>[user@localhost otus]$ cd ./users_groups/
[user@localhost users_groups]$</pre>

<p>Создадим файл Vagrantfile:</p>

<pre>[user@localhost users_groups]$ vi ./Vagrantfile</pre>

<p>Заполним следующим содержимым:</p>

<pre># -*- mode: ruby -*-
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
      SHELL

    end
  end
end
</pre>

<p>Запустим эту виртуальную машину:</p>

<pre>[user@localhost users_groups]$ vagrant up</pre>

<p>Проверим состояние созданной и запущенной машины:</p>

<pre>[user@localhost users_groups]$ vagrant status
Current machine states:

myhost                    running (virtualbox)

The VM is running. To stop this VM, you can run `vagrant halt` to
shut it down forcefully, or you can run `vagrant suspend` to simply
suspend the virtual machine. In either case, to restart it again,
simply run `vagrant up`.
[user@localhost users_groups]$</pre>

<p>Заходим на сервер myhost:</p>

<pre>[user@localhost users_groups]$ vagrant ssh myhost
[vagrant@myhost ~]$</pre>

<h4>PAM</h4>

<p>Заходим под правами root:</p>

<pre>[vagrant@myhost ~]$ sudo -i
[root@myhost ~]#</pre>

<p>Создадим двух пользователей adminik и userik и назначим им пароли:</p>

<pre>[root@myhost ~]# useradd adminik && echo "Otus1234" | passwd --stdin adminik && useradd userik && echo "Otus1234" | passwd --stdin userik
Changing password for user adminik.
passwd: all authentication tokens updated successfully.
Changing password for user userik.
passwd: all authentication tokens updated successfully.
[root@myhost ~]#</pre>

<p>Создадим группу admin и добавим в эту группу пользователя adminik:</p>

<pre>[root@myhost ~]# groupadd admin && usermod -aG admin adminik
[root@myhost ~]#</pre>

<p>Теперь пользователь adminik входит в группу admin:</p>

<pre>[root@myhost ~]# id adminik
uid=1001(adminik) gid=1001(adminik) groups=1001(adminik),1003(admin)
[root@myhost ~]#</pre>

<h4>Способ первый - PAM. Модуль pam_time</h4>

<p>Модуль pam_time позволяет достаточно гибко настроить доступ
пользователя с учетом времени. Настройки данного модуля хранятся
в файле /etc/security/time.conf. Данный файл содержит в себе
пояснения и примеры использования. Добавим в конец файла строки, где пользователям группы admin разрешаем вход в любое время, а пользователю userik - в любое время, кроме выходных:</p>

<pre>[root@myhost ~]# echo -e '*;*;admin;Al0000-2400\n*;*;userik;!Wd0000-2400' >> /etc/security/time.conf
[root@myhost ~]#</pre>

<p>Вставленные строки в файл time.conf выглядят следующим образом:</p>

<pre>[root@myhost ~]# cat /etc/security/time.conf 
...
*;*;admin;Al0000-2400
*;*;userik;!Wd0000-2400
[root@myhost ~]#</pre>

<p>Теперь настроим PAM, так как по-умолчанию данный модуль не
подключен.<br />
Для этого приведем файл /etc/pam.d/sshd к виду:</p>

<pre>[root@myhost ~]# sed -i '/account    required     pam_nologin.so/a account    required     pam_time.so' /etc/pam.d/sshd
[root@myhost ~]#</pre>

<pre>[root@myhost ~]# cat /etc/pam.d/sshd 
#%PAM-1.0
auth	   required	pam_sepermit.so
auth       substack     password-auth
auth       include      postlogin
# Used with polkit to reauthorize users in remote sessions
-auth      optional     pam_reauthorize.so prepare
account    required     pam_nologin.so
account    required     pam_time.so  # <----------------Добавлена строка--->
account    include      password-auth
password   include      password-auth
# pam_selinux.so close should be the first session rule
session    required     pam_selinux.so close
session    required     pam_loginuid.so
# pam_selinux.so open should only be followed by sessions to be executed in the user context
session    required     pam_selinux.so open env_params
session    required     pam_namespace.so
session    optional     pam_keyinit.so force revoke
session    include      password-auth
session    include      postlogin
# Used with polkit to reauthorize users in remote sessions
-session   optional     pam_reauthorize.so prepare
[root@myhost ~]#</pre>

<p>После чего в отдельном терминале можно проверить доступ к серверу по ssh для созданных пользователей.<br />Выходим из виртуальной машины:</p>

<pre>[root@myhost ~]# logout
[vagrant@myhost ~]$ logout
Connection to 127.0.0.1 closed.
[user@localhost users_groups]$</pre>

<p>Сегодня 28 августа 2022 года, воскресенье:</p>

<pre>[user@localhost users_groups]$ date
Вс авг 28 09:59:58 MSK 2022
[user@localhost users_groups]$</pre>

<p>Пробуем войти в виртуальную машину по ssh под логином adminik:</p>

<pre>[user@localhost users_groups]$ ssh adminik@192.168.56.150
The authenticity of host '192.168.56.150 (192.168.56.150)' can't be established.
ECDSA key fingerprint is SHA256:8kPcQ1DUNqyaE0Ho+Xe9KSnMoBevISf7RFNifC0GkOo.
ECDSA key fingerprint is MD5:01:cd:cc:da:68:b4:5e:65:4d:4f:f4:7e:4f:6a:74:da.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '192.168.56.150' (ECDSA) to the list of known hosts.
adminik@192.168.56.150's password: 
[adminik@myhost ~]$</pre>

<p>Как видим, мы вошли по ssh в эту виртуальную машину под логином adminik.<br />Выходим из неё и попытаемся войти под логином userik:</p>

<pre>[user@localhost users_groups]$ ssh userik@192.168.56.150
userik@192.168.56.150's password: 
Authentication failed.
[user@localhost users_groups]$</pre>

<p>Как видим, под логином userik не удалось войти в эту виртуальную машину.</p>

<p>Для теста установим сегодня как бы понедельник 29 августа 2022 года:</p>

<pre>[root@myhost ~]# date +%Y%m%d -s "20220829"
20220829
[root@myhost ~]# date
Mon Aug 29 00:00:03 UTC 2022
[root@myhost ~]#</pre>

<p>Снова пытаемся зайти под логином userik:</p>

<pre>[user@localhost users_groups]$ ssh userik@192.168.56.150
userik@192.168.56.150's password: 
Last failed login: Sun Aug 28 00:05:07 UTC 2022 from 192.168.56.1 on ssh:notty
There were 2 failed login attempts since the last successful login.
[userik@myhost ~]$</pre>

<p>В "понедельник" доступ разрешён под логином userik.</p>

<h4>Способ второй - PAM. Модуль pam_exec</h4>

<p>Еще один способ реализовать задачу это выполнить при
подключении пользователя скрипт, в котором мы сами обработаем
необходимую информацию.</p>

<p>Снова заходим в виртуальную машину и заходим под правами root:</p>

<pre>[user@localhost users_groups]$ vagrant ssh myhost
Last login: Sun Aug 28 06:03:07 2022 from 10.0.2.2
[vagrant@myhost ~]$ sudo -i
[root@myhost ~]#</pre>

<p>Удалим из /etc/pam.d/sshd изменения из предыдущего этапа и
приведем его к следующему виду:</p>

<pre>[root@myhost ~]# sed -i '/account    required     pam_time.so/d' /etc/pam.d/sshd
[root@myhost ~]#</pre>

<pre>[root@myhost ~]# sed -i '/account    required     pam_nologin.so/a account    required     pam_exec.so /usr/local/bin/test_login.sh' /etc/pam.d/sshd
[root@myhost ~]#</pre>

<pre>[root@myhost ~]# cat /etc/pam.d/sshd 
#%PAM-1.0
auth	   required	pam_sepermit.so
auth       substack     password-auth
auth       include      postlogin
# Used with polkit to reauthorize users in remote sessions
-auth      optional     pam_reauthorize.so prepare
account    required     pam_nologin.so
account    required     pam_exec.so /usr/local/bin/test_login.sh  # <--Добавлена строка-->
account    include      password-auth
password   include      password-auth
# pam_selinux.so close should be the first session rule
session    required     pam_selinux.so close
session    required     pam_loginuid.so
# pam_selinux.so open should only be followed by sessions to be executed in the user context
session    required     pam_selinux.so open env_params
session    required     pam_namespace.so
session    optional     pam_keyinit.so force revoke
session    include      password-auth
session    include      postlogin
# Used with polkit to reauthorize users in remote sessions
-session   optional     pam_reauthorize.so prepare
[root@myhost ~]#</pre>

<p>Мы добавили модуль pam_exec и, в качестве параметра, указали скрипт, который осуществит необходимые проверки.<br />Создадим сам скрипт test_login.sh:</p>

<pre>[root@myhost ~]# cat <<'EOF' >> /usr/local/bin/test_login.sh
#!/bin/bash
if [[ -n $(id $PAM_USER | grep -ow admin) || $(date +%u) -le 5 ]]; then
  exit 0
else
  exit 1
fi
EOF
[root@myhost ~]#</pre>

<p>и сделаем его исполняемым:</p>

<pre>[root@myhost ~]# chmod +x /usr/local/bin/test_login.sh 
[root@myhost ~]#</pre>

<p>Сегодня 28 августа 2022 года, воскресенье:</p>

<pre>[root@myhost ~]# date
Sun Aug 28 12:58:37 UTC 2022
[root@myhost ~]#</pre>

<p>Проверим, сможем ли войти под логином adminik:</p>

<pre>[user@localhost users_groups]$ ssh adminik@192.168.56.150
adminik@192.168.56.150's password: 
Last login: Sun Aug 28 12:16:45 2022 from 192.168.56.1
[adminik@myhost ~]$</pre>

<p>Как видим, под логином adminik доступ разрешен.<br />Теперь попробуем войти под логином userik:</p>

<pre>[user@localhost users_groups]$ ssh userik@192.168.56.150
userik@192.168.56.150's password: 
/usr/local/bin/test_login.sh failed: exit code 1
Authentication failed.
[user@localhost users_groups]$</pre>

<p>Как видим, доступ под логином userik в выходной день запрещен.<br />Установим сегодняшний день, например, "понедельником":</p>

<pre>[root@myhost ~]# date +%Y%m%d -s "20220829"
20220829
[root@myhost ~]# date
Mon Aug 29 00:00:04 UTC 2022
[root@myhost ~]#</pre>

<p>Проверим доступ под логином userik:</p>

<pre>[user@localhost users_groups]$ ssh userik@192.168.56.150
userik@192.168.56.150's password: 
Last failed login: Sun Aug 28 13:07:02 UTC 2022 from 192.168.56.1 on ssh:notty
There was 1 failed login attempt since the last successful login.
Last login: Sun Aug 28 13:02:48 2022
[userik@myhost ~]$</pre>

<p>Наблюдаем, что теперь доступ под логином userik разрешен.</p>

<pre>[user@localhost users_groups]$ ssh adminik@192.168.56.150
adminik@192.168.56.150's password: 
Last login: Sun Aug 28 13:04:39 2022 from 192.168.56.1
[adminik@myhost ~]$</pre>

<p>Вход под пользователем adminik также разрешен.</p>

<h4>Дать конкретному пользователю права работать с докером и возможность рестартить докер сервис</h4>

<p>В виртуальной машине добавим официальный репозиторий Docker в систему:</p>

<pre>[root@myhost ~]# yum install yum-utils -y
...
Updated:
  yum-utils.noarch 0:1.1.31-54.el7_8

Complete!
[root@myhost ~]#</pre>

<pre>[root@myhost ~]# yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
Loaded plugins: fastestmirror
adding repo from: https://download.docker.com/linux/centos/docker-ce.repo
grabbing file https://download.docker.com/linux/centos/docker-ce.repo to /etc/yum.repos.d/docker-ce.repo
repo saved to /etc/yum.repos.d/docker-ce.repo
[root@myhost ~]#</pre>

<p>Установим Docker:</p>

<pre>[root@myhost ~]# yum install docker-ce docker-ce-cli containerd.io -y
...
Installed:
  containerd.io.x86_64 0:1.6.8-3.1.el7         docker-ce.x86_64 3:20.10.17-3.el7
  docker-ce-cli.x86_64 1:20.10.17-3.el7

Dependency Installed:
  audit-libs-python.x86_64 0:2.8.5-4.el7
  checkpolicy.x86_64 0:2.5-8.el7
  container-selinux.noarch 2:2.119.2-1.911c772.el7_8
  docker-ce-rootless-extras.x86_64 0:20.10.17-3.el7
  docker-scan-plugin.x86_64 0:0.17.0-3.el7
  fuse-overlayfs.x86_64 0:0.7.2-6.el7_8
  fuse3-libs.x86_64 0:3.6.1-4.el7
  libcgroup.x86_64 0:0.41-21.el7
  libsemanage-python.x86_64 0:2.5-14.el7
  policycoreutils-python.x86_64 0:2.5-34.el7
  python-IPy.noarch 0:0.75-6.el7
  setools-libs.x86_64 0:3.3.8-4.el7
  slirp4netns.x86_64 0:0.4.3-4.el7_8

Complete!
[root@myhost ~]#</pre>

<p>Запустим docker сервис и включим автозапуск:</p>

<pre>[root@myhost ~]# systemctl enable docker --now
Created symlink from /etc/systemd/system/multi-user.target.wants/docker.service to /usr/lib/systemd/system/docker.service.
[root@myhost ~]#</pre>

<p>Статус запущенного docker сервиса:</p>

<pre>[root@myhost ~]# systemctl status docker
● docker.service - Docker Application Container Engine
   Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; vendor preset: disabled)
   Active: active (running) since Mon 2022-08-29 18:09:05 UTC; 20s ago
     Docs: https://docs.docker.com
 Main PID: 3508 (dockerd)
    Tasks: 7
   Memory: 91.2M
   CGroup: /system.slice/docker.service
           └─3508 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd....

Aug 29 18:09:05 myhost dockerd[3508]: time="2022-08-29T18:09:05.406140574Z" level=...pc
Aug 29 18:09:05 myhost dockerd[3508]: time="2022-08-29T18:09:05.406163276Z" level=...pc
Aug 29 18:09:05 myhost dockerd[3508]: time="2022-08-29T18:09:05.406173904Z" level=...pc
Aug 29 18:09:05 myhost dockerd[3508]: time="2022-08-29T18:09:05.452947410Z" level=...."
Aug 29 18:09:05 myhost dockerd[3508]: time="2022-08-29T18:09:05.653892735Z" level=...s"
Aug 29 18:09:05 myhost dockerd[3508]: time="2022-08-29T18:09:05.766364211Z" level=...."
Aug 29 18:09:05 myhost dockerd[3508]: time="2022-08-29T18:09:05.840364149Z" level=...17
Aug 29 18:09:05 myhost dockerd[3508]: time="2022-08-29T18:09:05.845024259Z" level=...n"
Aug 29 18:09:05 myhost systemd[1]: Started Docker Application Container Engine.
Aug 29 18:09:05 myhost dockerd[3508]: time="2022-08-29T18:09:05.894331463Z" level=...k"
Hint: Some lines were ellipsized, use -l to show in full.
[root@myhost ~]#</pre>

<p>Создадим пользователя dockeruser и добавим его в группу docker:</p>

<pre>[root@myhost ~]# useradd dockeruser -G docker && echo "Otus1234" | passwd --stdin dockeruser
Changing password for user dockeruser.
passwd: all authentication tokens updated successfully.
[root@myhost ~]#</pre>

<p>Смотрим информацию о пользователе dockeruser:</p>

<pre>[root@myhost ~]# id dockeruser
uid=1003(dockeruser) gid=1004(dockeruser) groups=1004(dockeruser),994(docker)
[root@myhost ~]#</pre>

<p>Включим логирование, а для этого создадим файл /etc/polkit-1/rules.d/00-access.rules со следующим содержимым:</p>

<pre>[root@myhost ~]# cat <<'EOF'>> /etc/polkit-1/rules.d/00-access.rules
polkit.addRule(function(action, subject) {
    polkit.log("action=" + action);
    polkit.log("subject=" + subject);
});
EOF
[root@myhost ~]#</pre>

<p>Попробуем перезапустить docker сервис:</p>

<pre>[dockeruser@myhost ~]$ systemctl restart docker
==== AUTHENTICATING FOR org.freedesktop.systemd1.manage-units ===
Authentication is required to manage system services or units.
Authenticating as: root
Password:</pre>

<p>Смотрим, что у нас в логе /var/log/secure:</p>

<pre>[root@myhost ~]# tail /var/log/secure
...
Aug 29 18:20:40 localhost polkit-agent-helper-1[3820]: pam_unix(polkit-1:auth): conversation failed
Aug 29 18:20:40 localhost polkit-agent-helper-1[3820]: pam_unix(polkit-1:auth): auth could not identify password for [root]
Aug 29 18:20:40 localhost polkit-agent-helper-1[3820]: pam_succeed_if(polkit-1:auth): requirement "uid >= 1000" not met by user "root"
Aug 29 18:20:40 localhost polkitd[336]: Unregistered Authentication Agent for unix-process:3810:93589 (system bus name :1.63, object path /org/freedesktop/PolicyKit1/AuthenticationAgent, locale en_US.UTF-8) (disconnected from bus)
Aug 29 18:20:40 localhost polkitd[336]: Operator of unix-process:3810:93589 FAILED to authenticate to gain authorization for action org.freedesktop.systemd1.manage-units for system-bus-name::1.64 [<unknown>] (owned by unix-user:dockeruser)
[root@myhost ~]#</pre>

<p>Из последних строк данного лога видно, что пользователю dockeruser НЕ УДАЛОСЬ пройти аутентификацию, чтобы получить разрешение на действие org.freedesktop.systemd1.manage-units.</p>

<p>Добавим пользователю право на перезапуск docker сервиса, создав для этого правило /etc/polkit-1/rules.d/01-docker.rules:</p>

<pre>cat <<'EOF'>> /etc/polkit-1/rules.d/01-docker.rules
polkit.addRule(function(action, subject) {
  if (action.id == "org.freedesktop.systemd1.manage-units" && action.lookup("unit") == "docker.service" && action.lookup("verb") == "restart" && subject.user == "dockeruser") {
    return polkit.Result.YES;
  }
});
EOF
[root@myhost ~]#</pre>

<p>Так как в CentOS 7 используется systemd v219:</p>

<pre>[root@myhost ~]# systemctl --version
systemd 219
+PAM +AUDIT +SELINUX +IMA -APPARMOR +SMACK +SYSVINIT +UTMP +LIBCRYPTSETUP +GCRYPT +GNUTLS +ACL +XZ +LZ4 -SECCOMP +BLKID +ELFUTILS +KMOD +IDN
[root@myhost ~]#</pre>

<p>что некоторые правила, в частности, action.lookup("unit") и action.lookup("verb") не сработают, нужно обновить systemd до v226. Обновим systemd из репозитория с бэкпортами: </p>

<p>Временно отключим selinux:</p>

<pre>[root@myhost ~]# setenforce 0
[root@myhost ~]#</pre>

<p>Скачиваем репо:</p>

<pre>[root@myhost ~]# curl https://copr.fedorainfracloud.org/coprs/jsynacek/systemd-backports-for-centos-7/repo/epel-7/jsynacek-systemd-backports-for-centos-7-epel-7.repo -o /etc/yum.repos.d/jsynacek-systemd-centos-7.repo
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   460  100   460    0     0    390      0  0:00:01  0:00:01 --:--:--   391
[root@myhost ~]#</pre>

<p>Обновляем systemd:</p>

<pre>[root@myhost ~]# yum update systemd -y
...
Replaced:
  systemd.x86_64 0:219-73.el7_8.5

Complete!
[root@myhost ~]#</pre>

<p>Systemd обновилась до v234:</p>

<pre>[root@myhost ~]# systemctl --version
systemd 234
+PAM +AUDIT +SELINUX +IMA -APPARMOR +SMACK +SYSVINIT +UTMP +LIBCRYPTSETUP +GCRYPT +GNUTLS +ACL +XZ +LZ4 +SECCOMP +BLKID +ELFUTILS +KMOD -IDN2 +IDN default-hierarchy=hybrid
[root@myhost ~]#</pre>

<p>Включаем selinux:</p>

<pre>[root@myhost ~]# setenforce 1
[root@myhost ~]#</pre>

<p>Снова пробуем перезапустить docker сервис под пользователем dockeruser:</p>

<pre>[dockeruser@myhost ~]$ systemctl restart docker
[dockeruser@myhost ~]$</pre>

<p>Как видим, docker сервис успешно перезапустился под пользователем dockeruser.</p>

