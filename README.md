## Loki

Most of the loki queries are based on systemd logs using labels for the systemd unit and the syslog identifiers. Also add the systemd priority as a level label, which is useful to get coloring in explore log volume and log views. These are scraped with a Loki job like:
```
- job_name: systemd-journal
  journal:
    max_age: 12h
    labels:
      job: systemd-journal
      instance: <HOSTNAME>
    path: /var/log/journal
  relabel_configs:
    - source_labels: ['__journal__systemd_unit']
      target_label: 'unit'
    - source_labels: ['__journal_syslog_identifier']
      target_label: 'syslog'
    - source_labels: ['__journal_priority_keyword']
      target_label: 'level'
```

### Nginx

The nginx dashboard is based on a custom logfmt-like access log. This is mainly because the default access log does not include the most interesting `$request_time` field, but also because I want to avoid unnecessary personal information like client IP address in my log stashes. Using logfmt makes for easier pattern matching, and integrates nicely with e.g. the grafana's explore log viewer. The custom access log is configured in a nginx http-like section with:
```
log_format anonymized escape=json 'time=$time_iso8601 unix=$msec server=$server_name:$server_port request="$request" status=$status size=$body_bytes_sent connection=$connection request_time=$request_time';
access_log /var/log/nginx/anonymized.log anonymized;
```

And the scraping and labeling from this can be configured with a Loki job like:
```
- job_name: nginx
  static_configs:
  - labels:
      job: nginx
      instance: <HOSTNAME>
      __path__: /var/log/nginx/anonymized.log
  pipeline_stages:
    - regex:
        expression: '.*unix=(?P<timestamp>[^ ]*) (?P<rest>.*)'
    - timestamp:
        format: Unix
        source: timestamp
    - output:
        source: rest
    - regex:
        expression: "server=(?P<server>[^ ]+)"
    - regex:
        expression: "status=(?P<status>[0-9]+)"
    - labels:
        server:
        status:
```
