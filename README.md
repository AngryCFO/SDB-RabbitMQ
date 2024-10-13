# Домашнее задание к занятию "Очереди RabbitMQ" - `Александра Бужор`

---

### Задание 1

Установка RabbitMQ
Используя Vagrant или VirtualBox, создайте виртуальную машину и установите RabbitMQ. Добавьте management plug-in и зайдите в веб-интерфейс.

Итогом выполнения домашнего задания будет приложенный скриншот веб-интерфейса RabbitMQ.

### Ответ

![image](https://github.com/user-attachments/assets/caf74547-92d4-494f-8a3a-32a09ab8f986)

---

### Задание 2

Отправка и получение сообщений
Используя приложенные скрипты, проведите тестовую отправку и получение сообщения. Для отправки сообщений необходимо запустить скрипт producer.py.

Для работы скриптов вам необходимо установить Python версии 3 и библиотеку Pika. Также в скриптах нужно указать IP-адрес машины, на которой запущен RabbitMQ, заменив localhost на нужный IP.

```$ pip install pika```

Зайдите в веб-интерфейс, найдите очередь под названием hello и сделайте скриншот. После чего запустите второй скрипт consumer.py и сделайте скриншот результата выполнения скрипта

В качестве решения домашнего задания приложите оба скриншота, сделанных на этапе выполнения.

Для закрепления материала можете попробовать модифицировать скрипты, чтобы поменять название очереди и отправляемое сообщение.

### Ответ

![image](https://github.com/user-attachments/assets/935dac91-19fb-4cef-b4be-90d14645a87c)
![image](https://github.com/user-attachments/assets/ccbddc44-462e-4fe1-bf75-7abdaaab41ef)
![image](https://github.com/user-attachments/assets/f17d501d-1667-4bad-be70-b2e0b92fad2b)

---

### Задание 3

Подготовка HA кластера
Используя Vagrant или VirtualBox, создайте вторую виртуальную машину и установите RabbitMQ. Добавьте в файл hosts название и IP-адрес каждой машины, чтобы машины могли видеть друг друга по имени.

Пример содержимого hosts файла:

```
$ cat /etc/hosts
192.168.0.10 rmq01
192.168.0.11 rmq02
```

После этого ваши машины могут пинговаться по имени.

Затем объедините две машины в кластер и создайте политику ha-all на все очереди.

В качестве решения домашнего задания приложите скриншоты из веб-интерфейса с информацией о доступных нодах в кластере и включённой политикой.

Также приложите вывод команды с двух нод:

```$ rabbitmqctl cluster_status```

Для закрепления материала снова запустите скрипт producer.py и приложите скриншот выполнения команды на каждой из нод:

```$ rabbitmqadmin get queue='hello'```

После чего попробуйте отключить одну из нод, желательно ту, к которой подключались из скрипта, затем поправьте параметры подключения в скрипте consumer.py на вторую ноду и запустите его.

Приложите скриншот результата работы второго скрипта.

### Ответ

![image](https://github.com/user-attachments/assets/f85b3577-55ac-4f77-8e1b-4e96a6ca2203)
![image](https://github.com/user-attachments/assets/ab76fe52-58da-4f34-b583-ba3e9336fa6e)
![image](https://github.com/user-attachments/assets/17ce24ea-1036-4bdd-b622-8b191fd96096)
![image](https://github.com/user-attachments/assets/8f2eea59-aecd-4815-8df6-100dbfafdda3)

---

### Задание 4

Напишите плейбук, который будет производить установку RabbitMQ на любое количество нод и объединять их в кластер. При этом будет автоматически создавать политику ha-all.

Готовый плейбук разместите в своём репозитории.

### Ответ

[Ansible Playbook](https://github.com/AngryCFO/SDB-RabbitMQ/blob/44aa8101c1baffb06ee6ae04c488ab3637b468b3/rabbitmq_cluster.yml)
```
---
- name: Install and Configure RabbitMQ Cluster
  hosts: rabbitmq_nodes
  become: true
  vars:
    erlang_cookie: "my_super_secret_cookie"

  tasks:
    - name: Update /etc/hosts with RabbitMQ nodes
      lineinfile:
        path: /etc/hosts
        line: "{{ hostvars[item].ansible_host }} {{ item }}"
        state: present
      loop: "{{ groups['rabbitmq_nodes'] }}"
      when: "'ansible_host' in hostvars[item]"

    - name: Install RabbitMQ
      apt:
        name: rabbitmq-server
        state: present
        update_cache: yes

    - name: Copy Erlang Cookie
      copy:
        content: "{{ erlang_cookie }}"
        dest: "/var/lib/rabbitmq/.erlang.cookie"
        owner: rabbitmq
        group: rabbitmq
        mode: '0600'

    - name: Restart RabbitMQ
      become: true
      service:
        name: rabbitmq-server
        state: restarted

    - name: Wait for RabbitMQ to Start
      wait_for:
        port: 5672
        delay: 5
        timeout: 60

    - name: Enable and start RabbitMQ service
      service:
        name: rabbitmq-server
        enabled: yes
        state: started

    - name: Check if RabbitMQ Management Plugin is enabled
      shell: rabbitmq-plugins list -e
      register: plugins
      changed_when: false
      failed_when: false

    - name: Enable RabbitMQ Management Plugin
      shell: rabbitmq-plugins enable rabbitmq_management
      when: "'rabbitmq_management' not in plugins.stdout"

    - name: Restart RabbitMQ
      become: true
      service:
        name: rabbitmq-server
        state: restarted

    - name: Wait for RabbitMQ to Start
      wait_for:
        port: 5672
        delay: 5
        timeout: 60

    - name: Add RabbitMQ user (rmuser)
      command: rabbitmqctl add_user rmuser rmpassword
      when: inventory_hostname == groups['rabbitmq_nodes'][0]
      ignore_errors: yes

    - name: Add RabbitMQ user (rmuser) permissions
      command: rabbitmqctl set_user_tags rmuser administrator
      when: inventory_hostname == groups['rabbitmq_nodes'][0]
      ignore_errors: yes

    - name: Delete default RabbitMQ user (guest)
      command: rabbitmqctl delete_user guest
      when: inventory_hostname == groups['rabbitmq_nodes'][0]
      ignore_errors: yes

    - name: Set permissions for rmuser on / vhost
      command: rabbitmqctl set_permissions -p / rmuser ".*" ".*" ".*"
      when: inventory_hostname == groups['rabbitmq_nodes'][0]
      ignore_errors: yes

    - block:
        - name: Stop RabbitMQ Application
          shell: rabbitmqctl stop_app
          register: stop_app

        - name: Reset RabbitMQ Node
          shell: rabbitmqctl reset
          when: stop_app is success
          register: reset_node

        - name: Join RabbitMQ cluster
          shell: rabbitmqctl join_cluster rabbit@{{ hostvars[groups['rabbitmq_nodes'][0]]['ansible_hostname'] }}
          when: reset_node is success
          register: join_cluster

        - name: Wait for Cluster Join to Complete
          pause:
            seconds: 5
          when: join_cluster is success

        - name: Start RabbitMQ Application
          shell: rabbitmqctl start_app
          when: join_cluster is success

      when: inventory_hostname != groups['rabbitmq_nodes'][0]

    - name: Set HA policy on all queues
      shell: rabbitmqctl set_policy ha-all ".*" '{"ha-mode":"all"}'
      when: inventory_hostname == groups['rabbitmq_nodes'][0]
```
