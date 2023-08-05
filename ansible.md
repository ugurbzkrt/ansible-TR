# Ansible Nedir?

IT görevlerini otomatikleştirmek için kullanılan konfigürasyon tooludur.
Tek bir sunucu üzerinden işlem yapmak istediğiniz sunucuları yönetmek için kullanılır.
Örnek verecek olursam herhangi bir Cloud Provider altyapısında çalışan 100 sunucunun "root password" değiştirmek isterseniz tek bir sunucu üzerinden yapabilirsiniz.
Bir diğer örnek ise 200 sunucunuzun "Docker" versiyonunu yükseltmek isterseniz tek bir makine ve dosya üzerinden yapabilirsiniz.
Controller olarak seçilen sucunuya sadece Ansible kurulur ve kontrol edilmek istenen makineler için herhangi bir Agent kurmaya ihtiyaç yoktur.
Agent'a ihtiyaç olmadığı için hem konfigürasyon derdi yoktur hemde hafızandan tasarruf ederiz.
Controller'da Python yüklü olması yeterlidir.
Emirleri SSH Protokolü üzerinden ve 22 Portunu kullarak iletir.
Terraform ile oluşturulan servisleri Ansible ile yönetilir.

Genelde Ansible'yi birbirini tekrar eden işleri otomatize hale getirmek için kullanırız.

 - Yeni kullancı oluşturmak.
 - Kullancıyı grupa dahil etmek.
 - Yetkilendirme. ( chmod + chown )
 - Yedekleme.
 - Sistem OS Restart.

Ansible'nin tek çalışma mantığı otomatizedir.

## Ansible Mimari Yapı:

 - Bir Controller makine ile diğer makineleri yönetiriz, Merkez sunucu.

 - Tek bir "Yaml" dosyasında yapılandırma/kurulum/dağıtım yapılır.

 - Aynı dosya birden çok kez ve farklı ortamlar için yeniden kullanılabilir.

 - Daha güvelinir ve hata olasılığı daha düşük.

 - Bağlantı sağlanarak ilgili Node'leri yönetmiş oluyoruz, SSH.

 - Bağlantıyı IP'ler üzerinden yürütürüz, komutları ve ilgili dosyaları çalıştırırız. ( Playbook )

 - Bu IP ve Pem bilgilerinin olduğu dosyanın kavramsal adı "Inventory" dosyasıdır.

 - Default dosyası "hosts" dosyasıdır. IP'ler ve bilgiler burada yer alır. Linux sistemlerdeki uygulamaların konfigurasyon dizininde yer alır > /etc/ansible/hosts.

## Ansible Commands:

- Yönetmekte olduğunuz tüm Node'leri listeler.
```
$ ansible all --list-hosts
```

- Contoller makineden yönetmekte olduğunuz makinelere erişimi kontrol etmek için:

```
$ ansible all -m ping
$ ansible webservers -m ping
$ ansible node1 -m ping
```

- Ping adımından önce makineler için bir authentication istedi ve bir çok makine olduğu durumlarda o adımı aşmak için Ansible tarafından düzeneleme yapılır.

```
sudo nano /etc/ansible/ansible.cfg
# Buradan host_key_checking = False'i comment out yapabiliriz.
host_key_checking = False
# Ve bu satırıda eklememiz gerekiyor.
[defaults]
interpreter_python=auto_silent
```

Artık komutları verebiliriz.

- Bir dosya oluşturdunuz ve makineye atmak istiyorsunuz, Bu bir scriptte olabilir.
```
$ ansible webservers -m copy -a "src=/etc/ansible/testfile dest=/home/ubuntu/testfile"
```
```
$ ansible webservers -m shell -a "echo Hello World > /home/ubuntu/testfile2 ; cat testfile2"
```

## Ansible Hosts:

- Örnek bir Ansible-Hosts dosyası:

```
[webservers]
node1 ansible_host=<node1_ip> ansible_user=centos

[dbservers]
node2 ansible_host=<node2_ip> ansible_user=ubuntu

[all:vars]
ansible_ssh_private_key_file=/home/ubuntu/<pem file>
ansible_ssh_private_key_file=/home/dbservers/<pem file>
```

---

Controller makinede yeni bir Node ekleyelim:
```
$ vi /etc/ansible/hosts

[ubuntuservers]
node3 ansible_host=<node3_ip> ansible_user=ubuntu

[ubuntuservers:vars] # Genellikle düzen açısından [all:vars] olarak her makineninkini tek şekilde belirtiriz.
ansible_ssh_private_key_file=/home/ubuntu/<YOUR-PEM-FILE-NAME>.pem
```

- Ve erişim testi yapalım.
```
$ ansible all --list-hosts
$ ansible all -m ping -o
```
### Shell modülü kullanarak Nginx kuralım:

```
$ ansible ubuntuservers -b -m shell -a "apt-get update -y; apt-get install nginx -y ; systemctl start nginx ; systemctl enable nginx" 
```


## Playbook:

Inventory dosyasına IP'ler ile tanımlanmış makinelere emir vermek, yönetmek için kullanılan komutların dosyasına denir. Yaml dosyalarıdır.

Playbook kullanım adımları:

 1. Contoller ile Node'lere bağlanılır.
 2. Controller'in çalışacağı makinenin IP'sini "Inventory" dosyasına gireriz.
 3. Nginx Install > Playbook.yml --> Emir.
 4. Belirtilen IP'lere Nginx kurmuş olur.

- Ansible ".pem" dosyaları ile iletişim kurar, bağlanılcak makinen pem'leri doğru şekilde ayarlamanız gerekir.

playbook-testconnection.yml
```yaml
- name: Test Connectivity
  hosts: all
  tasks:
   - name: Ping test
     ping:
```

- Playbook dosyasını çalıştırırız ve "hosts" dosyasında bulunan tüm makineleri erişimi kontrol eder.

```
$ ansible-playbook playbook1.yml
```

echo "Copy Test Ansible" > testfile1
nano playbook-copytest.yml

```yaml
---
- name: Copy for linux
  hosts: webservers
  tasks:
   - name: Copy your file to the webservers
     copy:
       src: /etc/ansible
       dest: /home/centos/testfile1

- name: Copy for ubuntu
  hosts: ubuntuservers
  tasks:
   - name: Copy your file to the ubuntuservers
     copy:
       src: /etc/ansible
       dest: /home/ubuntu/testfile1
```

- Dosyayı çalıştırmak için komutu giriniz.

```
$ ansible-playbook playbookcopytest.yml
```

- Son olarak Playbook ile Apache kurulumu.

vi playbook-httpd.yml

```yml
---
- name: Apache installation for webservers
  hosts: webservers
  tasks:
   - name: install the latest version of Apache
     yum:
       name: httpd
       state: latest

   - name: start Apache
     shell: "service httpd start"

- name: Apache installation for ubuntuservers
  hosts: ubuntuservers
  tasks:
   - name: update
     shell: "apt update -y"
     
   - name: install the latest version of Apache
     apt:
       name: apache2
       state: latest
```

## Ansible ile Docker Kurulum:

- Burada Ansible içindeki döngüleri kullanıyoruz.

vi playbook-docker.yml

```yml
---
- hosts: ubuntuservers # ya da webservers, daha doğrusu /etc/ansible/hosts dosyasında belirttiğiniz Node Grubuna ya da tek bir Node şekilde kullanabilirsiniz. Node1.
  become: true
  tasks:
    - name: installa dipendenze
      apt:
        name: "{{item}}"
        state: present
        update_cache: yes
      loop:
        - apt-transport-https
        - ca-certificates
        - curl
        - gnupg-agent
        - software-properties-common
- name: aggiungi chiave GPG
      apt_key:
        url: https://download.docker.com/linux/ubuntu/gpg
        state: present
- name: aggiungi repository docker
      apt_repository:
        repo: deb https://download.docker.com/linux/ubuntu bionic stable
        state: present
- name: installa docker
      apt:
        name: "{{item}}"
        state: latest
        update_cache: yes
      loop:
        - docker-ce
        - docker-ce-cli
        - containerd.io
- name: assicurati che docker sia attivo
      service:
        name: docker
        state: started
        enabled: yes
handlers:
    - name: restart docker
      service: 
        name: docker 
        state: restarted
```

- Komutu çalıştırdığımızda belirtmiş olduğumuz Node grublarına ya da tek bir Node için Docker kurulumunu yapacaktır.

```
$ ansible-playbook playbook-docker.yml
```