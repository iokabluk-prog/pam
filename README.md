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
# Создадим файл-скрипт /usr/local/bin/login.sh
root@pam:~# nano /usr/local/bin/login.sh
#!/bin/bash
#Первое условие: если день недели суббота или воскресенье
if [ $(date +%a) = "Sat" ] || [ $(date +%a) = "Sun" ]; then
 #Второе условие: входит ли пользователь в группу admin
 if getent group admin | grep -qw "$PAM_USER"; then
        #Если пользователь входит в группу admin, то он может подключиться
        exit 0
      else
        #Иначе ошибка (не сможет подключиться)
        exit 1
    fi
  #Если день не выходной, то подключиться может любой пользователь
  else
    exit 0
fi
# Добавляем права на исполнение root@pam:~# chmod +x /usr/local/bin/login.sh
root@pam:~# chmod +x /usr/local/bin/login.sh
# Укажем в файле /etc/pam.d/sshd модуль pam_exec и наш скрипт

# PAM configuration for the Secure Shell service

# Standard Un*x authentication.
@include common-auth

# === ВЫПОЛНЕНИЕ СКРИПТА ПРИ АУТЕНТИФИКАЦИИ ===
auth       required     pam_exec.so debug /usr/local/bin/login.sh

# Disallow non-root logins when /etc/nologin exists.
account    required     pam_nologin.so

# Uncomment and edit /etc/security/access.conf if you need to set comp>
# access limits that are hard to express in sshd_config.
# account  required     pam_access.so

# Standard Un*x authorization.
@include common-account

# SELinux needs to be the first session rule.  This ensures that any
# lingering context has been cleared.  Without this it is possible tha>
# module could execute code in the wrong domain.
session [success=ok ignore=ignore module_unknown=ignore default=bad]  >

# Set the loginuid process attribute.


# Устанавливаем время 
root@pam:~# sudo date 071112302026.00
Sat Jul 11 12:30:00 UTC 2026
# Проверим подключение по SSh для пользователей otus и otusadm
PS C:\WINDOWS\System32> ssh otus@192.168.56.11
otus@192.168.56.11's password:
Permission denied, please try again.
otus@192.168.56.11's password:
PS C:\WINDOWS\System32> ssh otusadm@192.168.56.11
otusadm@192.168.56.11's password:
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-181-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Wed Jul  8 08:30:45 UTC 2026

  System load:             0.01
  Usage of /:              4.2% of 38.70GB
  Memory usage:            22%
  Swap usage:              0%
  Processes:               111
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

Last login: Wed Jul  8 08:12:12 2026 from 192.168.56.1
Could not chdir to home directory /home/otusadm: No such file or directory
$ whoami
otusadm
$
# При логине пользователя otus появляется ошибка. Пользователь otusadm подключается без проблем
