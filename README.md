# Автоматическая настройка веб-сервера Nginx с помощью Ansible
(Устанавливает и настраивает Nginx, настраивает firewall, создает виртуальный хост, развертывает тестовую HTML страницу)

## Создаем playbook.yml
```yaml
---
- name: Configure Nginx Web Server
  hosts: webservers
  become: yes
  vars:
    domain_name: "example.com"
    admin_email: "admin@example.com"

  tasks:
    - name: Update package cache
      apt:
        update_cache: yes
        cache_valid_time: 3600

    - name: Install Nginx
      apt:
        name: nginx
        state: present

    - name: Start and enable Nginx
      systemd:
        name: nginx
        state: started
        enabled: yes

    - name: Configure firewall
      ufw:
        rule: allow
        port: '80'
        proto: tcp

    - name: Create website directory
      file:
        path: /var/www/{{ domain_name }}
        state: directory
        mode: '0755'

    - name: Deploy HTML page
      copy:
        content: |
          <!DOCTYPE html>
          <html>
          <head>
              <title>Welcome to {{ domain_name }}</title>
          </head>
          <body>
              <h1>Success! Nginx is working!</h1>
              <p>Server configured by Ansible</p>
          </body>
          </html>
        dest: /var/www/{{ domain_name }}/index.html
        mode: '0644'

    - name: Configure Nginx virtual host
      template:
        src: nginx-vhost.j2
        dest: /etc/nginx/sites-available/{{ domain_name }}
      notify: restart nginx

    - name: Enable virtual host
      file:
        src: /etc/nginx/sites-available/{{ domain_name }}
        dest: /etc/nginx/sites-enabled/{{ domain_name }}
        state: link
      notify: restart nginx

  handlers:
    - name: restart nginx
      systemd:
        name: nginx
        state: restarted
```
## Создаем шаблон nginx-vhost.j2
```nginx
server {
    listen 80;
    server_name {{ domain_name }};
    root /var/www/{{ domain_name }};
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    access_log /var/log/nginx/{{ domain_name }}_access.log;
    error_log /var/log/nginx/{{ domain_name }}_error.log;
}
```
## Создаем inventory.ini
```ini
[webservers]
web1 ansible_host=192.168.1.10 ansible_user=ubuntu
web2 ansible_host=192.168.1.11 ansible_user=ubuntu

[webservers:vars]
ansible_ssh_private_key_file=~/.ssh/id_rsa
```
## Использование
```bash
ansible-playbook -i inventory.ini playbook.yml
```

