# Создаем Vagrantfile 
MACHINES = {
  :"pam" => {
              :box_name => "ubuntu/jammy64",
              :cpus => 2,
              :memory => 1024,
              :ip => "192.168.56.11",
            }
}

Vagrant.configure("2") do |config|
  MACHINES.each do |boxname, boxconfig|
    config.vm.synced_folder ".", "/vagrant", disabled: true
	config.vm.box_url = "file://C:/vagrant_projects/my_vm/pam/ubuntu-jammy64.box"
  config.vm.boot_timeout = 600
    config.vm.network "private_network", ip: boxconfig[:ip]
    config.vm.define boxname do |box|
      box.vm.box = boxconfig[:box_name]
      box.vm.box_version = boxconfig[:box_version]
      box.vm.host_name = boxname.to_s

      box.vm.provider "virtualbox" do |v|
        v.memory = boxconfig[:memory]
        v.cpus = boxconfig[:cpus]
      end
      box.vm.provision "shell", inline: <<-SHELL
          sed -i 's/^PasswordAuthentication.*$/PasswordAuthentication yes/' /etc/ssh/sshd_config
          systemctl restart sshd.service
  	  SHELL
    end
  end
end
# Создаем ВМ командой vagrant up
# Подключаемся к ВМ и переходим в в root-пользователя
vagrant@pam:~$ sudo -i
root@pam:~#
# Создаём пользователя otusadm и otus
root@pam:~# sudo useradd otusadm && sudo useradd otus
# Создаём пользователям пароли
root@pam:~# sudo passwd otusadm
New password:
Retype new password:
passwd: password updated successfully
root@pam:~# sudo passwd otus
New password:
Retype new password:
passwd: password updated successfully
# Создаём группу admin
root@pam:~# sudo groupadd -f admin
# Добавляем пользователей vagrant,root и otusadm в группу admin
root@pam:~# usermod otusadm -a -G admin && usermod root -a -G admin && usermod vagrant -a -G admin
# Проверяем, что пользователи могут подключаться по SSH к  ВМ
PS C:\WINDOWS\System32> ssh otus@192.168.56.11
The authenticity of host '192.168.56.11 (192.168.56.11)' can't be established.
ED25519 key fingerprint is SHA256:CEc37Ak8yp7Ff0u0LOFVVuDQBUkYg0PR3MYmqveZzZA.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.56.11' (ED25519) to the list of known hosts.
otus@192.168.56.11's password:
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-181-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Wed Jul  8 08:09:56 UTC 2026

  System load:             0.0
  Usage of /:              4.1% of 38.70GB
  Memory usage:            21%
  Swap usage:              0%
  Processes:               110
  Users logged in:         1
  IPv4 address for enp0s3: 10.0.2.15
  IPv6 address for enp0s3: fd00::24:d8ff:fe7c:d655


Expanded Security Maintenance for Applications is not enabled.

0 updates can be applied immediately.

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
New release '24.04.4 LTS' available.
Run 'do-release-upgrade' to upgrade to it.



The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.


The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

Could not chdir to home directory /home/otus: No such file or directory
$ whoami
otus
$ exit
Connection to 192.168.56.11 closed.
PS C:\WINDOWS\System32> ssh otusadm@192.168.56.11
PS C:\WINDOWS\System32> ssh otusadm@192.168.56.11
otusadm@192.168.56.11's password:
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-181-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Wed Jul  8 08:12:11 UTC 2026

  System load:             0.0
  Usage of /:              4.2% of 38.70GB
  Memory usage:            21%
  Swap usage:              0%
  Processes:               112
  Users logged in:         1
  IPv4 address for enp0s3: 10.0.2.15
  IPv6 address for enp0s3: fd00::24:d8ff:fe7c:d655


Expanded Security Maintenance for Applications is not enabled.

0 updates can be applied immediately.

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
New release '24.04.4 LTS' available.
Run 'do-release-upgrade' to upgrade to it.



The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.


The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

Could not chdir to home directory /home/otusadm: No such file or directory
$ whoami
otusadm
$ exit
Connection to 192.168.56.11 closed.
# Настроим правило, по которому все пользователи кроме тех, что указаны в группе admin не смогут подключаться в выходные дни
# Проверим, что пользователи root, vagrant и otusadm есть в группе admin
root@pam:~# cat /etc/group | grep admin
admin:x:117:otusadm,root,vagrant
# Используем модуль pam_exec
