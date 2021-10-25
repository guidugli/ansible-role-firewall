Ansible Role: firewall
=========

An Ansible Role that install and configure firewalld or ufw on RHEL/CentOS, Fedora and Debian/Ubuntu.

Requirements
------------

No requirements.

Role Variables
--------------

> Please remember that this role:
>    - Assumes/Set default INPUT and FORWARD actions as deny.
>    - Default OUTPUT action is configured by firewall_block_all_output var
>    - is target for internal servers needing basic rules, so no forwarding rules can be implemented here
>    - will configure loopback on UFW as:
>         - ufw allow in on lo
>         - ufw allow out from lo
>         - ufw deny in from 127.0.0.0/8
>         - ufw deny in from ::1
>    - UFW reads service names from /etc/services but firewalld have XML files defining the services.

**Available variables are listed below, along with default values (see defaults/main.yml):**

    firewall_selected: "{{ _suggested_os_firewall }}"

Allow user to choose which firewall software to use

    firewalld_default_zone: public

This is a firewalld option to specify the default firewall zone.

    firewall_default_protocol: tcp

If you don't specify a protocol in `firewall_services`, fall back to this.

    firewall_default_action: allow

If you don't specify a target in `firewall_services`, fall back to this. Valid options: allow, deny.

    #firewall_interface_zone:
    #  - interface: enp1s0
    #    zone: "{{ firewalld_default_zone }}"
    #  - interface: virbr0
    #    zone: libvirt

Firewalld option to assign interfaces to firewall zones.

    #firewall_service_mapping:
    #  - name: ssh
    #    ports: ['14090/tcp']
    #  - name: cockpit
    #    ports: ['9090/tcp']
    #  - name: kodi
    #    # Description will only be used for new services
    #    description: 'This option allows kodi remote controls to communicate with kodi application'
    #    short: 'Allows kodi remotes to access kodi'
    #    ports: ['18998/tcp']
    #  - name: dhcpv6-client
    #    ports: ['546/udp']

Maps a service name with port numbers.
For firewalld, if service exists, updates the ports, otherwise create the service.
For UFW, do nothing, just use the port numbers when running the command later on the ports.
For example, cockpit port 9090/tcp would prevent an error of having a rule with service name as cockpit as it is not on /etc/services, so ufw cannot recognize it.

    firewall_output_default_action: allow   # options: allow or deny
    #firewall_output_allow_ports:
    #  - { port: 80, protocol: tcp }
    #  - { port: 443, protocol: tcp }
    #  - { port: 53, protocol: tcp }
    #  - { port: 53, protocol: udp }
    #  - { port: 67, protocol: tcp }
    #  - { port: 67, protocol: udp }

Should output traffic be allowed?
If set to yes, it will still open ports 80 (http), 443 (https), 67 (dhcp) and 53 (DNS), so system updates continue to work

    #firewall_services:
    #  - name: ssh
    #  - name: kodi
    #  - name: cockpit
    #    action: deny
    #  - name: dhcpv6-client
    #    action: deny

A list of service to allow traffic to.

**The variables listed below do not need to be changed for targeted systems (see vars/main.yml):**

    firewall_packages_conflicting:

Packages that conflict with selected firewall software. Conflicting packages will be removed from the system.

    firewall_packages_required:

Packages to be installed to implement selected firewall software.

    firewall_conflicting_services:

Services that conflicts with selected firewall software. These services will be disabled and stopped.

    firewall_ipv4_service_name:

Name of the service that implements firewall services for IPV4.

    firewall_ipv6_service_name:

Name of the service that implements firewall services for IPV6.


Dependencies
------------

No dependencies.

Example Playbook
----------------

    - hosts: servers
      vars:
        firewall_selected: "{{ _suggested_os_firewall }}"
        firewalld_default_zone: public
        firewall_default_protocol: tcp
        firewall_default_action: allow
        firewall_service_mapping:
          - name: ssh
            ports: ['19999/tcp']
          - name: cockpit
            ports: ['9090/tcp']
          - name: kodi
            # Description will only be used for new services
            description: 'This option allows kodi remote controls to communicate with kodi application'
            short: 'Allows kodi remotes to access kodi'
            ports: ['18998/tcp']
          - name: dhcpv6-client
            ports: ['546/udp']
        firewall_output_default_action: allow   # options: allow or deny
        firewall_output_allow_ports:
          - { port: 80, protocol: tcp }
          - { port: 443, protocol: tcp }
          - { port: 53, protocol: tcp }
          - { port: 53, protocol: udp }
          - { port: 67, protocol: tcp }
          - { port: 67, protocol: udp }
        firewall_services:
          - name: ssh
          - name: kodi
          - name: cockpit
            action: deny
          - name: dhcpv6-client
            action: deny
      roles:
         - { role: guidugli.firewall }

License
-------

MIT / BSD

Author Information
------------------

This role was created in 2020 by Carlos Guidugli.
