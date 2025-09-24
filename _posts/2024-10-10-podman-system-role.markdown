---
layout: post
title:  "RHEL System Role for Podman"
date:   2024-10-10 14:04:00 -0700
categories: stuff
---

```
- name: Configure Podman to run a web server as a rootless container.
  hosts: localhost
  tasks:
    - name: Create a web application and a database
      ansible.builtin.include_role:
        name: rhel-system-roles.podman
      vars:
        podman_create_host_directories: true
        podman_firewall:
          - port: 8080-8081/tcp
            state: enabled
          - port: 12340/tcp
            state: enabled
        podman_selinux_ports:
          - ports: 8080-8081
            setype: http_port_t
        podman_kube_specs:
          - state: started
            run_as_user: dbuser
            run_as_group: dbgroup
            kube_file_content:
              apiVersion: v1
              kind: Pod
              metadata:
                name: db
              spec:
                containers:
                  - name: db
                    image:  quay.io/linux-system-roles/mysql:5.6
                    ports:
                      - containerPort: 1234
                        hostPort: 12340
                    volumeMounts:
                      - mountPath: /var/lib/db:Z
                        name: db
                volumes:
                  - name: db
                    hostPath:
                      path: /var/lib/db
          - state: started
            run_as_user: webapp
            run_as_group: webapp
            kube_file_src: /path/to/webapp.yml
```
