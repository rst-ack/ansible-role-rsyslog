# Ansible Role: Rsyslog

A modular approach to configuring the Rsyslog daemon. Now with more complex configuration of remote log ingestion.

## Requirements

If you are deploying this syslog configuration on a CentOS or RHEL machine with SELinux enabled, you will need to configure it to allow certain actions.

Here is an example SELinux policy (in `.te` format -- you will need to compile and install it yourself) that has worked for me:

```perl
module rsyslog_policy 1.0;

require {
	type fsdaemon_t;
	type unconfined_t;
	type device_t;
	type syslogd_t;
	type auditd_t;
	type local_login_t;
	type sshd_t;
	type setroubleshootd_t;
	class sock_file { unlink write };
	class unix_dgram_socket sendto;
}

#============= auditd_t ==============
allow auditd_t device_t:sock_file write;

#============= fsdaemon_t ==============
allow fsdaemon_t device_t:sock_file write;

#============= local_login_t ==============
allow local_login_t device_t:sock_file write;

#============= setroubleshootd_t ==============
allow setroubleshootd_t device_t:sock_file write;

#============= sshd_t ==============
allow sshd_t device_t:sock_file write;
allow sshd_t unconfined_t:unix_dgram_socket sendto;

#============= syslogd_t ==============
allow syslogd_t device_t:sock_file write;
allow syslogd_t device_t:sock_file unlink;
```

## Role Variables

Available variabes are listed below in chunks, along with default values (see `defaults/main.yml`):

```yaml
# To configure the target machine to receive logs, set rsyslog_receiver to yes.
rsyslog_receiver: no

# Hostname or IP address of the host to send syslog data to. By not setting this variable, the target machine will not forward logs.
# rsyslog_remote:

# tcp, udp, and/or relp
rsyslog_remote_protocols:
  tcp: 514
```

I recommend using an IP address when setting `rsyslog_remote`. If the rsyslog client has issues with resolving the hostname of `rsyslog_remote`, then it will stop forwarding logs.

You can specify multiple protocols to forward logs with in `rsyslog_remote_protocols`. The format to use is `protocol_type: port_number`. For example, to send logs over 514/tcp and RELP on port 514:

```yaml
rsyslog_remote_protocols:
  tcp: 514
  relp: 514
```

You can also specify the ownership and file mode of created files and directories:

```yaml
# Set the mode for new directories
rsyslog_dircreatemode: "0700"
rsyslog_dirowner: "root"
rsyslog_dirgroup: "root"

# Set the mode for new files
rsyslog_filecreatemode: "0644"
rsyslog_fileowner: "root"
rsyslog_filegroup: "root"
```

By default, the standard `imuxsock` and `imjournal` modules are loaded:

```yaml
# List of modules to enable
rsyslog_modules:
  - imuxsock
  - imjournal
```

Additional allowed modules are `imklog` and `immark`. The `imfile` module is loaded by default, so there is no need to include it here (in fact, the playbook will throw an error if you do).

By default, file output rules are set in a format that makes sense for a client (the default target state of this role):

```yaml
rsyslog_client_file_output_rules:
  - rule: '*.info,mail.none,authpriv.none,cron.none'
    logpath: '/var/log/messages'
  - rule: 'authpriv.*'
    logpath: '/var/log/secure'
  - rule: 'mail.*'
    logpath: '-/var/log/maillog'
  - rule: 'cron.*'
    logpath: '/var/log/cron'
  - rule: '*.emerg'
    logpath: ':omusrmsg:*'
  - rule: 'uucp,news.crit'
    logpath: '/var/log/spooler'
  - rule: 'local7.*'
    logpath: '/var/log/boot.log'
  - rule: '*.debug'
    logpath: '/var/log/debug.log'
```

If the target machine is going to be receiving logs, I recommend the following output rules, which will retain *similar* log file names to that of the sending system (assuming the client is RHEL/CentOS based).

```yaml
rsyslog_file_output_rules:
  - rule: '*.info,mail.none,authpriv.none,cron.none'
    logfile: 'messages'
  - rule: 'authpriv.*'
    logfile: 'secure'
  - rule: 'mail.*'
    logfile: 'maillog'
  - rule: 'cron.*'
    logfile: 'cron'
  - rule: '*.emerg'
    logfile: 'emergency.log'
  - rule: 'uucp,news.crit'
    logfile: 'spooler'
  - rule: 'local7.*'
    logfile: 'boot.log'
  - rule: '*.debug'
    logfile: 'debug.log'
```

You can choose to preserve FQDNs when receiving log files:

```yaml
rsyslog_preservefqdn: no
```

By default (to remain compatible with legacy sysklogd), `rsyslog_preservefqdn` is turned off. In this case, if logs are sent within the same domain, only the short hostname is kept (the domain is stripped). If set to yes, full names are always kept.

You can also choose the method of checking if the `rsyslog` package is simply installed (`present`), or tell Ansible to keep the package up-to-date (`latest`):

```yaml
rsyslog_package_state: present
```

Rsyslog can be configured to drop logs that contain malicious DNS PTR records (https://www.rsyslog.com/doc/v8-stable/configuration/input_directives/rsconf1_dropmsgswithmaliciousdnsptrrecords.html). By default, this functionality is disabled to prevent any surprises.

```yaml
rsyslog_drop_malicious_ptr_records: no
```

A more advanced configuration is to have Rsyslog encrypt logs in transit with TLS. This is disabled by default due to the complexity it introduces, and the requirement for creating certificates.

```yaml
rsyslog_use_tls: no

# Only relevant if rsyslog_use_tls is set to yes
rsyslog_tls_ca_file: /etc/rsyslog.d/keys/rootCA.crt
rsyslog_tls_cert_file: /etc/rsyslog.d/keys/logmanagement-client.crt
rsyslog_tls_key_file: /etc/rsyslog.d/keys/logmanagement-client.key
```

Rsyslog has an array of plugins available, such as for outputting logs directly to Elasticsearch (https://www.rsyslog.com/doc/v8-stable/configuration/modules/omelasticsearch.html). Simply define the package name(s) in the list to install them.

**Note**: this role does not includ default configs for plugins; you will need to define configuration for plugins in the `rsyslog_custom_configs` variable (details below).

```yaml
# Install additional rsyslog plugins
# For example, to install the elastisearch plugin:
# rsyslog_plugins:
#   - gnutls
#   - elastisearch
rsyslog_plugins: []
```

Custom configurations can be defined directly in the YAML. The dictionary keys in `rsyslog_custom_configs` are used to set the name of each custom config file. Custom config files are included at the end of the base configuration file (set by `rsyslog_dest_conf_file`)

```yaml
# Configure custom configs to deploy in /etc/rsyslog.d/
# Example:
# rsyslog_custom_configs:
#   forward_splunk_tcp: # this will create /etc/rsyslog.d/forward_splunk_tcp.conf
#     content: |
#       action(type="omfwd"
#         target="splunk-indexer.example.com"
#         port="514"
#         protocol="tcp")
rsyslog_custom_configs: {}
```

The remaining -- more generic -- variables are below:

```yaml
# Set maximum message length
rsyslog_max_message_size: "4k"

# https://www.rsyslog.com/doc/v8-stable/configuration/modules/imuxsock.html#syssock-use
rsyslog_imuxsock_syssock: yes

# Default destination of rsyslog config file
rsyslog_dest_conf_file: "/etc/rsyslog.conf"

# Enable / Disable option OmitLocalLogging
rsyslog_omit_local_logging: yes

# rsyslog queue settings
rsyslog_queue_dir: "{{ rsyslog_work_directory }}/queue"
rsyslog_queue_filename: "input.q"
# Queue type
# Allowed types are: FixedArray, LinkedList, Direct, or Disk
rsyslog_queue_type: "LinkedList"
# queue sizing:
# - size for 1.000.000 messages in-memory queue
# - assume avg. message size is 1024 bytes
# - overhead due to pointers: 4 bytes on 32bit systems, 8 bytes on 64bit systems
# - memory usage: 1000000 * 1024 byyes + (1000000 * 8 bytes) = 984 MB
rsyslog_queue_size: 1000000
rsyslog_queue_threads: 1
# Defines the minimum number of messages in the queue to start a new thread
rsyslog_queue_min_messages: 1000
rsyslog_queue_dequeue_batch_size: 16
# Save the queue to disk on shutdown
rsyslog_queue_save_on_shutdown: on

# dynaFile template options
rsyslog_remote_log_save_path_template_name: "RemoteLogSavePath"
rsyslog_remote_log_save_path_template: "/var/log/remote_syslog/%SOURCE%/%$YEAR%-%$MONTH%-%$DAY%/%$.logpath%"
```

## Dependencies

None.

## Example playbook

```yaml
---
- name: Deploy Rsyslog configuration
  hosts: all
  roles:
    - role: rst-ack.rsyslog
```

## License

GNU GPLv3

## Contributing

If you would like to contribute to this role, please create a pull request including as much detail as you can.

## Acknowledgements

This role was created in 2023 with inspiration from:

* [Robert DeBock's rsyslog role](https://github.com/robertdebock/ansible-role-rsyslog)
* [jmaas's rsyslog-configs](https://github.com/jmaas/rsyslog-configs)
