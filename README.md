# Домашнее задание к занятию "08.03 Использование Yandex Cloud"

## Порядок запуска
1. Указать ip хостов в inventory/elk/hosts.yml
2. Запустить:   
ansible-playbook -i inventory/elk site.yml --diff

На хост el-instance будет установлен и сконфигурирован elasticsearch.  
На хост k-instance будет установлена kibana и сконфигурирована на подключение к elasticsearch.  
На хост app-instance будет установлен filebeat и сконфигурирован на передачу логов в elasticsearch, с размещением дашбордов в kibana.  

## inventory
1. В групповых переменных для всех хостов (inventory/elk/group_vars/all.yml) заведена переменная elk_stack_version. Определяет версию всех компонентов elk (использовал последнюю на текущий момент 7.16.2).
2. inventory/elk/hosts.yml создано три группы хостов elasticsearch, kibana, app.

```yml
---
elasticsearch:
  hosts:
    el-instance:
      ansible_host: <add ip>
kibana:
  hosts:
    k-instance:
      ansible_host: <add ip>
app:
  hosts:
    app-instance:
      ansible_host: <add ip>
```

## templates
Размещены три файла с конфигурациями для elasticsearch, kibana и filebeat.

## site.yml
```yml
---
- name: Install Elasticsearch # установка Elasticsearch на группу хостов "elasticsearch"
  hosts: elasticsearch
  handlers:
    - name: restart Elasticsearch # хендлер для перезапуска сервиса elasticsearch
      become: true
      service:
        name: elasticsearch
        state: restarted
  tasks:
    - name: "Download Elasticsearch's rpm" # Скачиваем rpm с Elasticsearch
      get_url:
        url: "https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-{{ elk_stack_version }}-x86_64.rpm"
        dest: "/tmp/elasticsearch-{{ elk_stack_version }}-x86_64.rpm"
      register: download_elastic
      until: download_elastic is succeeded
    - name: Install Elasticsearch # Устанавливаем Elasticsearch из скачанного rpm
      become: true
      yum:
        name: "/tmp/elasticsearch-{{ elk_stack_version }}-x86_64.rpm"
        state: present
    - name: Configure Elasticsearch # Размещаем файл с конфигурацией Elasticsearch и обращаемся к хендлеру для перезапуска сервиса Elasticsearch
      become: true
      template:
        src: elasticsearch.yml.j2
        dest: /etc/elasticsearch/elasticsearch.yml
      notify: restart Elasticsearch
- name: Install Kibana # установка Kibana на группу хостов "kibana"
  hosts: kibana
  handlers:
    - name: restart Kibana # хендлер для перезапуска сервиса kibana
      become: true
      service:
        name: kibana
        state: restarted
  tasks:
    - name: "Download Kibana's rpm" # Скачиваем rpm с Kibana
      get_url:
        url: "https://artifacts.elastic.co/downloads/kibana/kibana-{{ elk_stack_version }}-x86_64.rpm"
        dest: "/tmp/kibana-{{ elk_stack_version }}-x86_64.rpm"
      register: download_kibana
      until: download_kibana is succeeded
    - name: Install Kibana # Устанавливаем Kibana из скачанного rpm
      become: true
      yum:
        name: "/tmp/kibana-{{ elk_stack_version }}-x86_64.rpm"
        state: present
    - name: Configure Kibana # Размещаем файл с конфигурацией Kibana и обращаемся к хендлеру для перезапуска сервиса Kibana
      become: true
      template:
        src: kibana.yml.j2
        dest: /etc/kibana/kibana.yml
      notify: restart Kibana
- name: Install Filebeat  # установка Filebeat на группу хостов "app"
  hosts: app
  handlers:
    - name: restart filebeat # хендлер для перезапуска сервиса filebeat
      become: true
      service:
        name: filebeat
        state: restarted
  tasks:
    - name: "Download filebeat's rpm" # Скачиваем rpm с filebeat
      get_url:
        url: "https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-{{ elk_stack_version }}-x86_64.rpm"
        dest: "/tmp/filebeat-{{ elk_stack_version }}-x86_64.rpm"
      register: download_filebeat
      until: download_filebeat is succeeded
    - name: Install filebeat # Устанавливаем filebeat из скачанного rpm
      become: true
      yum:
        name: "/tmp/filebeat-{{ elk_stack_version }}-x86_64.rpm"
        state: present
    - name: Configure filebeat # Размещаем файл с конфигурацией filebeat и обращаемся к хендлеру для перезапуска сервиса filebeat
      become: true
      template:
        src: filebeat.yml.j2
        dest: /etc/filebeat/filebeat.yml
      notify: restart filebeat
    - name: Set filebeat systemwork # активируем модуль filebeat на хосте
      become: true
      command:
        cmd: filebeat modules enable system
        chdir: /usr/share/filebeat/bin
      register: filebeat_modules
      changed_when: filebeat_modules.stdout != 'Module system is already enabled'
    - name: Load kibana dashboard # Инициируем установку дашборда filebeat в Kibana 
      become: true
      command:
        cmd: filebeat setup
        chdir: /usr/share/filebeat/bin
      register: filebeat_setup
      changed_when: false
      until: filebeat_setup is succeeded
```