# Zabbix template: Borgmatic

[![License][license-image]][license-url]

Monitor [Borgmatic](https://torsion.org/borgmatic/) backups with [Zabbix](https://zabbix.com).

## Features

- 2 `pie` diagrams for repo size in `chunks` and `bytes`.
- 2 `normal` diagrams for repo size in `chunks` and `bytes`.
- 1 dashboard with 9 panels: `sizes`, `location`, `repo id`, `last modified`.
- 2 `triggers`: "missing config" and "borg is busy".
- Checking information 2 times per hour: every 25 and 55 min (if your `borg` makes archives slowly).

## Requirements

- Borgmatic (installed in `/usr/bin/borgmatic`)
- Zabbix server 6.0 (other version not tested, but it should work))
- Zabbix agent v2 (v1 not tested, but it should work)
- Ansible (optional)

## HowTo

### How to install template manually

In zabbix server:

- Get <https://github.com/don-rumata/zabbix-template-borgmatic/zabbix-template-borgmatic.yml>.
- In web gui zabbix server: `Configuration` -> `Templates` -> `Import`.
- Select file.
- `Import`.

In zabbix client:

```bash
sudo wget https://raw.githubusercontent.com/don-rumata/zabbix-template-borgmatic/master/borgmatic-info.sh -O /usr/local/sbin/borgmatic-info.sh

sudo chmod 755 /usr/local/sbin/borgmatic-info.sh

sudo wget https://raw.githubusercontent.com/don-rumata/zabbix-template-borgmatic/master/10-execute-borgmatic-4-zabbix -O /etc/sudoers.d/10-execute-borgmatic-4-zabbix

sudo chmod 400 /etc/sudoers.d/10-execute-borgmatic-4-zabbix

sudo wget https://raw.githubusercontent.com/don-rumata/zabbix-template-borgmatic/master/borgmatic-info.conf -O /etc/zabbix/zabbix_agent2.d/borgmatic-info.conf

sudo systemctl restart zabbix-agent2.service
```

### How to install template over `Ansible`

Set your vars:

- `zabbix_server_url`
- `zabbix_web_frontend_admin_user`
- `zabbix_admin_password`

`import-zabbix-template.yml`:

```yaml
---
  - name: Import zabbix template in zabbix server
    hosts: all
    strategy: free
    serial:
      - "100%"
    tasks:

    - ansible.builtin.uri:
        url: https://raw.githubusercontent.com/don-rumata/zabbix-template-borgmatic/master/zabbix-template-borgmatic.yml
        method: GET
        return_content: yes
        status_code:
          - 200
      register: zabbix_template_borgmatic_status

    - ansible.builtin.set_fact:
       zabbix_template_borgmatic_fact:
         "{{ zabbix_template_borgmatic_status.content | from_yaml }}"

    - community.zabbix.zabbix_template:
        server_url: "{{ zabbix_server_url }}"
        login_user: "{{ zabbix_web_frontend_admin_user }}"
        login_password: "{{ zabbix_admin_password }}"
        template_json:
          "{{ zabbix_template_borgmatic_fact }}"
```

`config-zabbix-agent.yml`:

```yaml
---
  - name: Config zabbix agent
    hosts: all
    strategy: free
    serial:
      - "100%"
    tasks:

    - name: Config Zabbix Agent 4 Borgmatic 4 Linux
      when:
        - ansible_system == 'Linux'
      become: yes
      block:
        - get_url:
            url: https://raw.githubusercontent.com/don-rumata/zabbix-template-borgmatic/master/borgmatic-info.conf
            dest: /etc/zabbix/zabbix_agent2.d/borgmatic-info.conf
            mode: '644'
            backup: yes
          notify: restart-zabbix-agent

        - get_url:
            url: https://raw.githubusercontent.com/don-rumata/zabbix-template-borgmatic/master/borgmatic-info.sh 
            dest: /usr/local/sbin/borgmatic-info.sh
            mode: '755'
            backup: yes
          notify: restart-zabbix-agent

        - get_url:
            url: https://raw.githubusercontent.com/don-rumata/zabbix-template-borgmatic/master/10-execute-borgmatic-4-zabbix
            dest: /etc/sudoers.d/10-execute-borgmatic-4-zabbix
            mode: '400'
            backup: yes
            validate: 'visudo -cf %s'
          notify: restart-zabbix-agent

    handlers:
      - name: restart-zabbix-agent
        when:
          - ansible_system == 'Linux'
        become: yes
        systemd:
          name: zabbix-agent2.service
          state: restarted
          daemon-reload: yes
```

## License

Apache License, Version 2.0

## Author Information

[don Rumata](https://github.com/don-rumata)

## TODO

- Add `borgmatic list`.
- Add screenshots.
- ru_RU lang for descriptions.

[license-image]: https://img.shields.io/github/license/don-rumata/zabbix-template-borgmatic.svg
[license-url]: https://opensource.org/licenses/Apache-2.0
