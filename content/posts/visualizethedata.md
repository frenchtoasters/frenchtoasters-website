+++
categories = ["monitoring","prometheus","vcd","vmware", "grafana"]
date = "2018-05-01T09:32:37-04:00"
tags = ["vmware","prometheus","vcloud","vcd", "grafana"]
title = "prometheusExporter?"

+++

# The Setup

In the previous post we talked about deploying some hosts using the `vcd-cli` so one thing you might be asking is "How can I view the state of this". Well you could always just login to the console and go check things one at a time but who really has time for that? There are also several canned reports that you could use to gather and show you this information, but why wait for the report to generate when you wanna see the health or current utilization of things? Today im going to be show you how you can visualize and track your environment in real time, all of this using a combination of a few different open source projects. First we will be using Promtheus for collecting these metrics then we will use Grafana plus the query lanaguage of Prometheus to visualize these metrics. 

## TL;DR

In short im going to show you how to setup a Prometheus server, Prometheus exporter, and Grafana instance on your local machine. This will give you the ability to visialize any metrics that your Prometheus exporter gathers in Grafana with "fancy" dashboards to replace the need to send generated email reports. In this world of on demand consumption of resources why on earth are we using point in time generated reports when its even easier to just create a dashboard. Where you write down once what you want and can come back to it and see at any point in time the values while also give RBAC to who can view what pieces of information or "fancy" dashboards.

## Configuration

So this post isnt going to explain all the possible ways to run Prometheus, Prometheus exporters, or Grafana , gotta save those for more content later :smile:. For this post im just going to be running both of these as executables on my Ubuntu 18.04 system.

* First download and extract Prometheus

```
# Linux binary
$> wget https://github.com/prometheus/prometheus/releases/download/v2.9.2/prometheus-2.9.2.linux-amd64.tar.gz
$> tar -zxvf prometheus-2.9.2.linux-amd64.tar.gz
```

* Download and extract Grafana

```
# Linux binary 
$> wget https://dl.grafana.com/oss/release/grafana-6.1.6.linux-amd64.tar.gz 
$> tar -zxvf grafana-6.1.6.linux-amd64.tar.gz 
```

* Run Example exporter

```
$> docker pull registry.gitlab.com/frenchtoasters/vcd_exporter:latest
$> docker run -it --rm  -p 9273:9273 -e VCD_USER=${VCD_USERNAME} -e VCD_PASSWORD=${VCD_PASSWORD} -e VCD_IGNORE_SSL=True --name vcd_exporter registry.gitlab.com/frenchtoasters/vcd_exporter:latest -p 9273
```

* For Promtheus you are going to need to create a configuartion file `prometheus.yml` like this:

```
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9090']

  - job_name: 'vcd_exporter'
    metrics_path: '/vcd'
    static_configs:
    - targets: ['localhost:9273']
      labels: 
        group: 'vcd-localhost'
  
  - job_name: 'vcd_exporter_metrics'
    static_configs:
    - targets: ['localhost:9273']
      labels: 
        group: 'vcd-gather-metrics'
```

* Then you can invoke the binary from the extracted directory, with this command to start prometheus in the background:

```
$> ./prometheus --config.file=prometheus.yml &
```

* Then you will need to start Grafana in a similar way by invoking the binary in the extracted directory to run in the background as well:

```
$> ./bin/grafana-server web &
```

* Now if you are following along you can verify everything is up but check these addresses in your browser:

```
localhost:9273/vcd # vCD Exporter default
localhost:9090 # Prometheus Metrics Explorer
localhost:3000 # Grafana Dashboard
```

## How do, please?

There are a few things you will need in place before we can start our work here. First we will need to have some type of environment built so that we know, what type of exporter to use and what metrics we are going to be able to display. In this example im going to be using a `VMware vCD` environment and the [vcd_exporter](https://github.com/frenchtoasters/vcd_exporter), if we check out the link here we can see an example output of the metrics that the `vcd_exporter` gathers for us:

* vcd_org_is_enabled:

```
# HELP vcd_org_is_enabled Enabled status of Organization
# TYPE vcd_org_is_enabled gauge 
vcd_org_is_enabled{org_full_name="Lobster-Shack",org_name="urn:vcloud:org:0be49a83-0e85-460a-b1b4-9ac84a4de480"} 1.0
```

* vcd_vdc_cpu_allocated:

```
# HELP vcd_vdc_cpu_allocated CPU allocated to vdc
# TYPE vcd_vdc_cpu_allocated gauge
vcd_vdc_cpu_allocated{allocation_model="AllocationVApp",org_id="urn:vcloud:org:0be49a83-0e85-460a-b1b4-9ac84a4de480",org_name="Lobster-Shack",vdc_id="urn:vcloud:vdc:12aa5168-bd0b-4958-acf8-2b40f706a81c",vdc_is_enabled="True",vdc_name="DR"} 0.0
```

* vcd_vdc_mhz_to_vcpu:

```
# HELP vcd_vdc_mhz_to_vcpu Mhz to vCPU ratio of vdc
# TYPE vcd_vdc_mhz_to_vcpu gauge
vcd_vdc_mhz_to_vcpu{allocation_model="AllocationVApp",org_id="urn:vcloud:org:0be49a83-0e85-460a-b1b4-9ac84a4de480",org_name="Lobster-Shack",vdc_id="urn:vcloud:vdc:12aa5168-bd0b-4958-acf8-2b40f706a81c",vdc_is_enabled="True",vdc_name="DR"} 1000.0
```

* vcd_vdc_memory_allocated:

```
# HELP vcd_vdc_memory_allocated Memory allocated to vdc
# TYPE vcd_vdc_memory_allocated gauge
vcd_vdc_memory_allocated{allocation_model="AllocationVApp",org_id="urn:vcloud:org:0be49a83-0e85-460a-b1b4-9ac84a4de480",org_name="Lobster-Shack",vdc_id="urn:vcloud:vdc:12aa5168-bd0b-4958-acf8-2b40f706a81c",vdc_is_enabled="True",vdc_name="DR"} 0.0
```

* vcd_vdc_memory_used_bytes:

```
# HELP vcd_vdc_memory_used_bytes Memory used by vdc in bytes
# TYPE vcd_vdc_memory_used_bytes gauge
vcd_vdc_memory_used_bytes{allocation_model="AllocationVApp",org_id="urn:vcloud:org:0be49a83-0e85-460a-b1b4-9ac84a4de480",org_name="Lobster-Shack",vdc_id="urn:vcloud:vdc:12aa5168-bd0b-4958-acf8-2b40f706a81c",vdc_is_enabled="True",vdc_name="DR"} 0.0
```

* vcd_vdc_used_network_count:

```
# HELP vcd_vdc_used_network_count Number of networks used by vdc
# TYPE vcd_vdc_used_network_count gauge
vcd_vdc_used_network_count{allocation_model="AllocationVApp",org_id="urn:vcloud:org:0be49a83-0e85-460a-b1b4-9ac84a4de480",org_name="Lobster-Shack",vdc_id="urn:vcloud:vdc:12aa5168-bd0b-4958-acf8-2b40f706a81c",vdc_is_enabled="True",vdc_name="DR"} 2.0
```

* vcd_vdc_vapp_status:

```
# HELP vcd_vdc_vapp_status Status of vApp
# TYPE vcd_vdc_vapp_status gauge
vcd_vdc_vapp_status{org_id="urn:vcloud:org:0be49a83-0e85-460a-b1b4-9ac84a4de480",org_name="Lobster-Shack",vapp_deployed="false",vapp_id="urn:vcloud:vapp:276e345a-3b35-436f-85cb-42242c0421d6",vapp_name="Web",vapp_status="1",vdc_id="urn:vcloud:vdc:12aa5168-bd0b-4958-acf8-2b40f706a81c",vdc_is_enabled="True",vdc_name="DR"} 1.0
```

* vcd_vdc_vapp_in_maintenance:

```
# HELP vcd_vdc_vapp_in_maintenance Status of maintenance mode of given vApp
# TYPE vcd_vdc_vapp_in_maintenance gauge
vcd_vdc_vapp_in_maintenance{org_id="DR",org_name="urn:vcloud:org:0be49a83-0e85-460a-b1b4-9ac84a4de480",vapp_deployed="false",vapp_id="urn:vcloud:vapp:276e345a-3b35-436f-85cb-42242c0421d6",vapp_name="Web",vdc_id="1",vdc_is_enabled="Lobster-Shack",vdc_name="urn:vcloud:vdc:12aa5168-bd0b-4958-acf8-2b40f706a81c"} 0.0
```

* vcd_vdc_vapp_vm_status:

```
# HELP vcd_vdc_vapp_vm_status Status of VM
# TYPE vcd_vdc_vapp_vm_status gauge
vcd_vdc_vapp_vm_status{org_id="Lobster-Shack",org_name="True",vapp_deployed="urn:vcloud:vdc:12aa5168-bd0b-4958-acf8-2b40f706a81c",vapp_id="App",vapp_name="false",vdc_id="DR",vdc_name="urn:vcloud:org:0be49a83-0e85-460a-b1b4-9ac84a4de480",vm_deployed="false",vm_id="urn:vcloud:vm:393da289-6f35-4d19-82cd-67ba4793ce6e",vm_name="app2.lobstershack.com_replica",vm_os_type="urn:vcloud:vapp:ba2291d4-d730-4ea7-8bfe-7edadfa7bb94",vm_status="8"} 8.0
```

* vcd_vdc_vapp_vm_vcpu:

```
# HELP vcd_vdc_vapp_vm_vcpu vCPU count of vm in given vApp of vdc
# TYPE vcd_vdc_vapp_vm_vcpu gauge
vcd_vdc_vapp_vm_vcpu{org_id="Lobster-Shack",org_name="True",vapp_deployed="urn:vcloud:vdc:12aa5168-bd0b-4958-acf8-2b40f706a81c",vapp_id="App",vapp_name="false",vdc_id="DR",vdc_name="urn:vcloud:org:0be49a83-0e85-460a-b1b4-9ac84a4de480",vm_deployed="false",vm_id="urn:vcloud:vm:393da289-6f35-4d19-82cd-67ba4793ce6e",vm_name="app2.lobstershack.com_replica",vm_os_type="urn:vcloud:vapp:ba2291d4-d730-4ea7-8bfe-7edadfa7bb94",vm_status="8"} 2.0
```

* vcd_vdc_vapp_vm_allocated_memory_mb:

```
# HELP vcd_vdc_vapp_vm_allocated_memory_mb Memory allocated to VM of given vApp of vdc
# TYPE vcd_vdc_vapp_vm_allocated_memory_mb gauge
vcd_vdc_vapp_vm_allocated_memory_mb{org_id="Lobster-Shack",org_name="True",vapp_deployed="urn:vcloud:vdc:12aa5168-bd0b-4958-acf8-2b40f706a81c",vapp_id="App",vapp_name="false",vdc_id="DR",vdc_name="urn:vcloud:org:0be49a83-0e85-460a-b1b4-9ac84a4de480",vm_deployed="false",vm_id="urn:vcloud:vm:393da289-6f35-4d19-82cd-67ba4793ce6e",vm_name="app2.lobstershack.com_replica",vm_os_type="urn:vcloud:vapp:ba2291d4-d730-4ea7-8bfe-7edadfa7bb94",vm_status="8"} 4096.0
```

So if you dont want to take the time to read all that the jist of the metrics that we gather are status and utilization of vOrgs, vDCs, vApps, and VMs aka. "The State Of the Environment". Along with each entery for those metrics that are returned is a set of labels, these labels are what we are going to use when we query Prometheus to narrow down the scope of or search over those metrics. This is really the secret sauce of it all, because we are using these labels the backend query is able to be extremely fast and straight forward to understand. Why is that? Well read the [Prometheus Data Model](https://prometheus.io/docs/concepts/data_model/) to find out why it would be for timeseries data like this. 

### Prometheus Queries

Ok so now that we kinda know what data we might get back from this exporter we can start to think about how we might want to display this information that we have. First though you should read through the basics on [Prometheus Query language](https://prometheus.io/docs/prometheus/latest/querying/basics/), the TL;DR read the labels copy and paste the exporter output. 

So now that you've got that page up here is a puzzle for those of you following along:

```
vcd_org_is_enabled{job='vcd_exporter', group='vcd-localhost'}
sum(vcd_org_is_enabled{})
count(vcd_vdc_cpu_allocated{allocation_model='AllocationPool'})
sum(vcd_vdc_memory_used_bytes{})
sum(vcd_vdc_memory_allocated{})
sum(vcd_vdc_used_network_count{})
sum(vcd_vdc_cpu_allocated{})
sum(vcd_vdc_memory_used_bytes{})
sum(vcd_vdc_vapp_status{vdc_name='DR'})
vcd_vdc_vapp_status{vapp_status='4'}
vcd_vdc_vapp_vm_status{vapp_status='0'}
sum(vcd_vdc_vapp_in_maintenance{})
sum(vcd_vdc_vapp_vm_status{vapp_status='-1'})
sum(vcd_vdc_vapp_vm_vcpu{})
sum(vcd_vdc_vapp_vm_allocated_memory_mb{})
sum(vcd_vdc_memory_used_bytes{})
process_max_fds{job='vcd_exporter_metrics'}
process_max_fds{job='vcd_exporter_metrics'}
process_virtual_memory_bytes{job='vcd_exporter_metrics'}
process_cpu_seconds_total{job='vcd_exporter_metrics'}
process_open_fds{job='vcd_exporter_metrics'}
process_resident_memory_bytes{job='vcd_exporter_metrics'}
```


### Grafana dashboard

After you solve that I think you are going to probably wanna take a look at the [Grafana Docs on its Features](https://grafana.com/docs/features/panels/graph/). 

Now that you are a SME on that you have all the information you need to create an awesome dashboard of your own in Grafana to display metrics! If you followed along though you can just import the [vcd_exporter dashboard here](https://github.com/frenchtoasters/vcd_exporter/blob/master/grafana_dashboard.json) and "see how the world was made", as I think someone said at some point somewhere. 

# Conclusion

So there it is, all the tools and hopefully links you would need to build some dashboards to show some data. There is no need to be writing scripts to just gather data once and send it somewhere, infrastructure is now code you so why not make your montioring all just code too? With Prometheus Exporters you write the steps to gather the information you need and then using a webserver library you can easily expose those metrics to the network. You then have Prometheus Server running jobs to gather those exported metrics, storing those metrics in a persistent location, and providing a single endpoint with an exposed query language to search through the metrics that servers jobs are gathering. Then instead of generating some html, pdf, markdown, etc... type of report you just create a Dashboard in Grafana. That dashboard would contain all the same data in the report that you would query to build your report. However with Grafana you are able to display back to the user that report in real-time, or if you are just gathering data for yourself id suggest also checking out [Alerting in Grafana](https://grafana.com/docs/alerting/rules/). Then Grafana will be reading the report for you and letting you know when something goes wrong. 
