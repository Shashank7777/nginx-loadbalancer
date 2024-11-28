Let me walk you through the complete process of deployment of a nginx-loadbalancer using Ansible and Jinja2 template. I have 4 servers as following:

- Two web servers [ansible2, ansible3]
- One loadbalancer [loadbalancer]
- One Ansible control node


**Prerequisites**:

- SSH setup between the control node and managed nodes.
- Proper configuration of /etc/hosts file.

**Control Node File Structure:**

```bash
.
├── ansible.cfg
├── install_nginx.yml
├── install_start_httpd.yml
├── inventory
├── loadbalancer_nginx.conf.j2
└── loadbalancer_setup.yml
```

**Inventory Details:**

```bash
[Webserver]
ansible2
ansible3

[LoadBalancer]
loadbalancer
```

**Ansible.cfg file:**

```bash
[defaults]
remote_user = ansible
host_key_checking = false
inventory = inventory

[privilege_escalation]
become = True
become_method = sudo 
become_user = root
become_ask_pass = False
```

**HTTPD Installation and Web Page Configuration on Web Servers.**

```yaml
---
- name: Install and start httpd package
  hosts: Webserver
  vars:
    web_content: |
      Welcome to {{ ansible_hostname }}
  tasks:
    - name: Install the httpd package.
      yum:
        name: httpd
        state: present
    - name: Start the httpd package.
      service:
       name: httpd
       state: started
       enabled: yes
    - name: Create a custom index.html file.
      copy:
        dest: /var/www/html/index.html
        content: "{{ web_content }}"
        owner: apache
        group: apache
        mode: '0644'
      notify:
        - Restart httpd
    - name: Allow HTTP service through the firewall
      firewalld:
        service: http
        permanent: yes
        state: enabled
        zone: public
    - name: Reload firewalld to apply changes
      command: firewall-cmd --reload
  handlers:
    - name: Restart httpd
      service:
        name: httpd
        state: restarted
```

This Ansible playbook installs and configures the Apache HTTP server (`httpd`) on hosts in the "Webserver" group. It installs the `httpd` package, starts and enables it to run on boot, creates a custom `index.html` file with dynamic content (`ansible_hostname`), and ensures proper ownership and permissions. It also configures the firewall to allow HTTP traffic and reloads the firewall settings. A handler is defined to restart `httpd` whenever the `index.html` file is modified.

**Running the Playbook**

```bash
ansible-playbook install_start_httpd.yml
```

**NGINX Installation and setup** 

```yaml
---
- name: Install nginx on the loadbalancer
  hosts: loadbalancer
  vars:
    nginx_msg: "Welcome to Nginx - {{ ansible_hostname }}"
  tasks:
    - name: Install nginx package
      package:
        name: nginx
        state: present
    - name: Ensure Nginx service is started
      service:
        name: nginx
        state: started
        enabled: yes
    - name: Allow HTTP service through the firewall
      firewalld:
        service: http
        permanent: yes
        state: enabled
        zone: public
    - name: Reload the firewall
      command: firewall-cmd --reload
    - name: Create a custom index.html file
      copy:
        dest: /usr/share/nginx/html/index.html
        content: "{{ nginx_msg }}"
        owner: nginx
        group: nginx
        mode: '0644'
      notify:
        - Restart nginx
  handlers:
    - name: Restart nginx
      command: systemctl restart nginx
```

This Ansible playbook installs NGINX on the **loadbalancer** host, starts and enables the service, and configures the firewall to allow HTTP traffic. It creates a custom `index.html` file with a welcome message (`nginx_msg`) that dynamically includes the server's hostname. If the `index.html` file is updated, the `Restart nginx` handler ensures the NGINX service is restarted to apply the changes.

**Running the playbook:**

```yaml
ansible-playbook install_nginx.yml
```

**Loadbalaner setup.**

```yaml
---
- name: Configure Nginx as a Loadbalancer
  hosts: loadbalancer
  vars:
    backend_servers:
      - ansible2
      - ansible3
    upstream_name: "webservers"

  tasks:
    - name: Install nginx if not already installed
      yum:
        name: nginx
        state: present

    - name: configure nginx as a loadbalancer
      template:
        src: loadbalancer_nginx.conf.j2
        dest: /etc/nginx/nginx.conf
        mode: '0644'

    - name: Ensure nginx service is started and enabled. 
      service:
        name: nginx
        state: started
        enabled: yes
```

This Ansible playbook configures NGINX as a load balancer on the **loadbalancer** host. It first installs NGINX if it’s not already present, then uses a Jinja2 template (`loadbalancer_nginx.conf.j2`) to dynamically generate the NGINX configuration file based on the specified backend servers (`ansible2` and `ansible3`) and upstream group name (`webservers`). Finally, it ensures the NGINX service is started and enabled to persist across reboots.

```python
events {
    worker_connections 1024;
}
http {
	upstream {{ upstream_name }} {
	{% for server in backend_servers %}
	server {{ server  }};
	{% endfor %}
	}

	server {
	    listen 80;

	    location / {
	    	proxy_pass http://{{ upstream_name }};
	    	proxy_set_header Host $host;
	    	proxy_set_header X-Real-IP $remote_addr;
            	proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
             	proxy_set_header X-Forwarded-Proto $scheme;
	    }
	}
}

```

This is a Jinja2 template for configuring NGINX as a load balancer. The `upstream` block defines a backend group (`upstream_name`) with a list of servers (`backend_servers`) dynamically generated by looping through the provided server names. The `server` block listens on port 80 and proxies incoming requests to the upstream backend while forwarding essential headers like `Host`, `X-Real-IP`, and `X-Forwarded-For` for proper request handling.

**Running the Playbook:**

```yaml
ansible-playbook loadbalancer_setup.yml
```
