## Описание файлов в директории
logFileFull.log - полный лог выполнения  
used_commands.txt - команды которые использовал  

Vagrant_folder - все что понадобится для поднятия VM и краткое описание файлов в ней  
Vagrantfile           - вагрант файл  
provisioning          - директория с файлами для настройки ВМ через ansible разбитый на роли
├── lab.yml
└── roles
    ├── log
    │   ├── handlers
    │   │   └── main.yml
    │   ├── tasks
    │   │   └── main.yml
    │   └── templates
    │       └── rsyslog.conf.j2
    └── web
        ├── files
        │   ├── my-inimfile.pp
        │   ├── my-inimfile.te
        │   ├── my-rsyslogd.pp
        │   └── my-rsyslogd.te
        ├── handlers
        │   └── main.yml
        ├── tasks
        │   └── main.yml
        └── templates
            ├── audit.conf.j2
            ├── nginx.conf.j2
            └── nginx.rules.j2
 

## Описание как запустить виртуальную машину (кратко)
Поднять ВМ и подключиться к ВМ log
```
vagrant up
vagrant ssh log
```
Выполнить команды ниже чтобы проверить что логи с сервера web передаются
```
grep web_config_changed /var/log/rsyslog/web/audit.log
  Oct 17 13:54:06 web tag_audit_log: type=SYSCALL msg=audit(1634478844.921:1593): arch=c000003e syscall=2 success=yes exit=3 a0=7fff82dbdd9c a1=941 a2=1b6 a3=7fff82dbbfe0 items=2 ppid=4733 pid=4787 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=2 comm="touch" exe="/usr/bin/touch" subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 key="web_config_changed"
cat /var/log/rsyslog/web/nginx.log
  Oct 17 13:54:04 web nginx: 192.168.11.101 - - [17/Oct/2021:13:54:04 +0000] "GET / HTTP/1.1" 200 4833 "-" "curl/7.29.0"
```

# VagrantFile только важное
провижинг playbook lab.yml
```
  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "provisioning/lab.yml"
    ansible.become = "true"
  end
```
добавляем дополнителный адаптер и настраиваем на нем адресс
```
web.vm.network "private_network", ip: "192.168.11.101", adapter: 2, netmask: "255.255.255.0", virtualbox__intnet: "elk-lab"
```
Выполняем комманды на ВМ. Добавляем имена ВМ в /etc/hosts, чтобы можно было обращаться по имени.  
Добавляем модули selinux, чтобы rsyslog имел доступ к логу /var/log/audit/audit.log.  
Изменяем атрибуты файла nginx.conf чтобы изменения отобразились в логе, т.к. ведется наблюдение на изменение этого конфига.  
Выполняем curl запрос для того чтобы так же передались данные в лог, на удаленный сервер.
```
web.vm.provision "shell", run: "always", inline: <<-SHELL
 echo 192.168.11.101 web >> /etc/hosts
 echo 192.168.11.102 log >> /etc/hosts
 semodule -i /tmp/my-rsyslogd.pp
 semodule -i /tmp/my-inimfile.pp
 service auditd restart
 touch /etc/nginx/nginx.conf
 curl 192.168.11.101 > /dev/null
 SHELL
```

## Подробнее о ansible файлах  
## ВМ log
### provisioning\roles\log\tasks\main.yml
создаем файл на основе шаблона
```
  - name: RSYSLOG | Create RSYSLOG config file from template
    template:
      src: templates/rsyslog.conf.j2
      dest: /etc/rsyslog.conf
    notify:
      - reload rsyslog
```

### provisioning\roles\log\templates\rsyslog.conf.j2 - только важная часть
Логи auditd передаем с facility=local6 и сохраняем их в файле /var/log/rsyslog/web/audit.log  
Остальные логи складываем /var/log/rsyslog/web/%PROGRAMNAME%.log
```
$template HostAudit, "/var/log/rsyslog/%HOSTNAME%/audit.log"
local6.* ?HostAudit

$template RemoteLogs,"/var/log/rsyslog/%HOSTNAME%/%PROGRAMNAME%.log"
*.* ?RemoteLogs
& ~
```
## ВМ web
### provisioning\roles\web\tasks\main.yml
Установить nginx и заменить его конфиг
```
---
  - name: NGINX | Install EPEL Repo package from standart repo
    yum:
      name: epel-release
      state: present

  - name: NGINX | Install NGINX package from EPEL Repo
    yum:
      name: nginx
      state: latest

  - name: NGINX | Create NGINX config file from template
    template:
      src: templates/nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    notify:
      - reload nginx
```
Настроить rsyslog на отправку всех логов на ВМ log. Так же добавить конфигурационные файлы, через шаблоны, для audit и nginx.
```
  - name: RSYSLOG | Edit config rsyslog
    lineinfile: dest=/etc/rsyslog.conf
       regexp='^#*.* @@remote-host:514'
       line='*.* @@log:514'
       state=present
    notify:
      - reload rsyslog

  - name: RSYSLOG | Edit config rsyslog
    template:
      src: templates/audit.conf.j2
      dest: /etc/rsyslog.d/audit.conf
    notify:
      - reload rsyslog
```
Через шаблон добавляем правила nginx для audit, наблюдение изменения конфигрурационных файлов
```
  - name: AUDIT | Create config audit
    template:
      src: templates/nginx.rules.j2
      dest: /etc/audit/rules.d/nginx.rules
```
Копируем модули для selinux
```
  - name: selinux module
    copy:
      src: '{{ item }}'
      dest: /tmp/
      mode: 666
    loop:
     - files/my-inimfile.pp
     - files/my-rsyslogd.pp
```
### provisioning\roles\web\templates\audit.conf.j2
```
$ModLoad imfile								#загружаем модуль imfile для наблюдением за файлом
$InputFileName /var/log/audit/audit.log		#лог файл, за которым наблюдаем
$InputFileTag tag_audit_log:				#тег, который будет использоваться для записей, полученных из этого файла
$InputFileStateFile audit_log				#фиксируем позицию в файле
$InputFileSeverity info						#уровень важности при котором забираем запись лога
$InputFileFacility local6					#категорию кторую ставим на лог
$InputRunFileMonitor						#включаем наблюдение

*.* @@log:514
```
### provisioning\roles\web\templates\nginx.conf.j2 только важное
Локально пишем только критические события, все остальное отправляем на ВМ log
```
error_log /var/log/nginx/error.log crit;
error_log syslog:server=192.168.11.102;
access_log syslog:server=192.168.11.102;
```
### provisioning\roles\web\templates\nginx.rules.j2
audit наблюдает за изменением конфигурационных файлов и их атрибутов
```
-w /etc/nginx/nginx.conf -p wa -k web_config_changed
-w /etc/nginx/conf.d/ -p wa -k web_config_changed
```

📚Домашнее задание/проектная работа разработано(-на) для курса ["Administrator Linux. Professional"](https://otus.ru/lessons/linux-professional/)