## Syslog server
The Monasca Log Agent is supposed to run on a remote (virtual) machine and acts as a syslog server. The logs must be sent from FIWARE Lab Nodes to the Monasca Log Agent via syslog. Every FIWARE Lab Node should run its own instance of the Monasca Log Agent.

## Installing the Monasca Log Agent
Copy the [log-client/monasca-log-agent](https://github.com/martel-innovate/deep-log-inspection/tree/master/log-client/monasca-log-agent) directory to the syslog server's home folder. Then, using [docker](https://www.docker.com/), pull the [image](https://hub.docker.com/r/martel/monasca-log-agent/) and run it in a container, as follows:

    docker pull martel/monasca-log-agent
    docker run -d --name=monasca-log-agent --restart on-failure \
     -v ./monasca-log-agent/logstash.conf:/etc/logstash/conf.d/logstash.conf \
     -p 1025:1025/udp martel/monasca-log-agent

Logstash's syslog input plugin will be listening for logs on port 1025 and the monasca_log_api output plugin will send them to the Monasca Log API. The pipeline can be found in [logstash.conf](https://github.com/martel-innovate/deep-log-inspection/blob/master/log-client/monasca-log-agent/logstash.conf).

The monasca_log_api output plugin must authenticate to Keystone, so make sure it authenticates to the same instance as the Monasca Log API. Please refer to Monasca Log API's [configuration guide](monasca-log-api.md) for more details.

## Region
The Monasca Log Agent is responsible for submitting information about the region where the logs are generated. The region has to be added to the _dimensions_ in the output section of `logstash.conf`:

    output {
        monasca_log_api {
            monasca_log_api_url => "https://example.com/v3.0"
            monasca_log_api_insecure => false
            keystone_api_url => "http://example.com:35357/v3"
            keystone_api_insecure => true
            project_name => "deeplog"
            username => "monasca-log-agent"
            password => "PASSWORD"
            user_domain_name => "default"
            project_domain_name => "default"
            dimensions => [ "hostname:monasca-log-agent", "region:zurich" ]
            num_of_logs => 125
            delay => 10
            elapsed_time_sec => 30
            max_data_size_kb => 5120
        }
    }

For more configuration details refer to the [documentation](http://www.rubydoc.info/gems/logstash-output-monasca_log_api/0.5.1#Start_logstash_output_plugin) of the monasca_log_api output plugin.

## Openstack syslog-rsyslog configuration
In order to forward the logs from a FIWARE Lab Node to its own syslog server, Openstack Services must send logging information to syslog. This is done by editing the configuration files of the involved services, e.g.:

+ `/etc/nova/nova.conf`
+ `/etc/glance/glance-api.conf`
+ `/etc/glance/glance-registry.conf`
+ `/etc/cinder/cinder.conf`

In each file, add these lines:

    debug = False
    use_syslog = True
    syslog_log_facility = LOG_LOCAL0

You might want to configure a separate local facility (up to eight: `LOCAL0, LOCAL1, ..., LOCAL7`) for each service, as this provides better isolation and more flexibility. For more information, see the [syslog documentation](https://en.wikipedia.org/wiki/Syslog).

Then, rsyslog must be configured on every service node, e.g. for Nova, create on every compute node a file named `/etc/rsyslog.d/60-nova.conf` with the following content:

    # prevent debug from dnsmasq with the daemon.none parameter
    *.*;auth,authpriv.none,daemon.none,local0.none -/var/log/syslog
    # include all log levels
    local0.*    @syslog-server:1025

Replace `syslog-server` with the address of the machine where the Monasca Log Agent is running. For more details on the configuration illustrated in this section, refer to [Openstack's official guide](https://docs.openstack.org/nova/pike/admin/manage-logs.html).
