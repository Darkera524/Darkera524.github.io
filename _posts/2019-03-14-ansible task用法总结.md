---
layout: post
title:  "ansible task用法总结"
categories: ansible
tags: ansible note
author: 大可
---

#### task
 - 幂等性
    - 多次执行同一个task与执行一次task的结果一致
    - 对于ansible task用到的module来说，幂等主要用于对于同一个任务，不会因为执行了多次playbook或者该task而导致最终结果不同，有些（并非全部）module天生是支持幂等的，其背后也是执行了对任务开始时状态的检测，来决定本次执行的动作，对于不能天生支持幂等的module，也应该通过某些手段来实现幂等，使得整个playbook是幂等的，可重复执行的。
 - params：
    - name: 人类可读的描述信息
    - module：使用的是模块名
    - notify/listen（https://docs.ansible.com/ansible/latest/user_guide/playbooks_intro.html#handlers-running-operations-on-change http://www.zsythink.net/archives/2624）
        - handlers：当改动发生时，触发handler任务，但是handler的执行时间为所有task结束后，且同一个handler仅会被触发一次
            - 与幂等性息息相关，只有当任务执行结果为changed，才会触发handler任务
            - meta:flush_handlers  : 是一种特殊的任务，使得在该任务之前的handler均在此刻触发，使得handlers更为灵活
          
          ```
            - name: template configuration file
              template:
                  src: template.j2
                  dest: /etc/foo.conf
              notify:
                  - restart memcached
                  - restart apache
          ```
          
        - listen:如果想要一次触发多个handlers，那么需要使用listen来触发一个topic的所有handler
          ```
          handlers:
            - name: restart memcached
            service:
                name: memcached
                state: restarted
            listen: "restart web services"
            - name: restart apache
            service:
                name: apache
                state:restarted
            listen: "restart web services"

            tasks:
                - name: restart everything
                command: echo "this task will restart the web services"
                notify: "restart web services"
          ```
              
    - when（https://docs.ansible.com/ansible/latest/user_guide/playbooks_conditionals.html#the-when-statement）: 触发条件，可以在不添加双花括号的情况下直接调用variable
        - commonly used facts(https://docs.ansible.com/ansible/latest/user_guide/playbooks_conditionals.html#commonly-used-facts)
    - loop
    
      ```
        - name: non optimal yum, not only slower but might cause issues with interdependencies
          yum:
              name: "{{item}}"
              state: present
          loop: "{{list_of_packages}}"
      ```
          
    - when与loop结合使用，对loop中的每一个元素执行判断when条件
    
      ```
        - command: echo {{ item }}
          loop: [ 0, 2, 4, 6, 8, 10 ]
          when: item > 5
      ```
          
    - ignore_errors:忽略本次task执行失败
    - register:result（保存本次执行结果状态，用以跳转不同condition task）
        - when: result is failed
        - when: result is succeeded
        - when: result is skipped
        
        ```tasks:
        - command: /bin/false
          register: result
          ignore_errors: True

        - command: /bin/something
          when: result is failed

        # In older versions of ansible use ``success``, now both are valid but succeeded uses the correct tense.
        - command: /bin/something_else
          when: result is succeeded

        - command: /bin/still/something_else
          when: result is skipped
      ```
          
    - until & retry:重复执行一个任务，直到达到指定条件（https://docs.ansible.com/ansible/latest/user_guide/playbooks_loops.html#do-until-loops）
        - If the until parameter isn’t defined, the value for the retries parameter is forced to 1.
          
          ```
          - shell: /usr/bin/foo
            register: result
            until: result.stdout.find("all systems go") != -1
            retries: 5
            delay: 10
          ```
          
    - register & loop：当同时使用loop和register时，那么变量中将包括一个result字段，包含每一次循环结果的列表（https://docs.ansible.com/ansible/latest/user_guide/playbooks_loops.html#using-register-with-a-loop）
      
      ```
        - shell: "echo {{ item }}"
          loop:
              - "one"
              - "two"
          register: echo
      ```
          
    - block:是由一系列task组成的逻辑组，绝大多数可以应用到task的操作均可应用到block，可以看作是一个大型的task，或者说是多继承了task的task，每个针对block的操作会应用到其所继承的每一个task，添加到每一个task的上下文中(https://docs.ansible.com/ansible/latest/user_guide/playbooks_blocks.html)
    
      ```
      - name: Install Apache
        block:
          - yum:
              name: "{{ item }}"
              state: installed
            with_items:
              - httpd
              - memcached
          - template:
              src: templates/src.j2
              dest: /etc/foo.conf
          - service:
              name: bar
              state: started
              enabled: True
        when: ansible_facts['distribution'] == 'CentOS'
        become: true
        become_user: root
      ```
          
    - rescue:差错处理，block仅处理状态为failed的task，对于task定义错误、未定义变量错误或者不可达主机等错误是不处理的
    
      ```
      - name: Handle the error
        block:
          - debug:
              msg: 'I execute normally'
          - name: i force a failure
              command: /bin/false
          - debug:
              msg: 'I never execute, due to the above task failing, :-('
        rescue:
          - debug:
              msg: 'I caught an error, can do stuff here to fix it, :-)'
      ```
          
    - always:该部分会执行无论block执行结果如何
    
      ```
      - name: Attempt and graceful roll back demo
        block:
          - debug:
              msg: 'I execute normally'
          - name: i force a failure
            command: /bin/false
          - debug:
              msg: 'I never execute, due to the above task failing, :-('
      rescue:
          - debug:
              msg: 'I caught an error'
          - name: i force a failure in middle of recovery! >:-)
            command: /bin/false
          - debug:
            msg: 'I also never execute :-('
      always:
          - debug:
            msg: "This always executes"
      ```
          
    - become:权限升级，如某task需要以root权限执行（https://docs.ansible.com/ansible/latest/user_guide/become.html#id1）
        - become:yes
    - tags:用以只执行指定的一部分任务而不是执行每一件任务（https://docs.ansible.com/ansible/latest/user_guide/playbooks_tags.html）
        - 当应用在非task上时，会把该tag附在被应用对象下的每一个task上，避免了为每一个task单独打tag的麻烦
    - migrating from with_X to loop:From 2.5（https://docs.ansible.com/ansible/latest/user_guide/playbooks_loops.html#migrating-from-with-x-to-loop）
        - with_list
        
        ```
        - name: with_list
          debug:
              msg: "{{ item }}"
          with_list:
              - one
              - two

        - name: with_list -> loop
          debug:
              msg: "{{ item }}"
          loop:
              - one
              - two
        ```
        
        - with_items
        
        ```
        - name: with_items
          debug:
              msg: "{{ item }}"
          with_items: "{{ items }}"

        - name: with_items -> loop
          debug:
              msg: "{{ item }}"
          loop: "{{ items|flatten(levels=1) }}"
        ```
        
 - templates
    - some operation
    
{:toc}