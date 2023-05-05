# Prometheus &amp; Grafana on a Raspberry Pi
Using docker-compose to run Prometheus &amp; Grafana on a Raspberry Pi

## Install Docker & docker-compose

Instructions from [dev.to](https://dev.to/rohansawant/installing-docker-and-docker-compose-on-the-raspberry-pi-in-5-simple-steps-3mgl)
this is now fairly out of date as Compose is now part of the Docker install

* Install docker `curl -sSL https://get.docker.com | sh`
* Add permissions for user pi to run docker commands `sudo usermod -aG docker pi`

## Run Prometheus & Grafana

* Build and run `sudo docker compose up --build`
* Or run in the background `sudo docker compose up -d`
* Prometheus should now be running at `http://raspberrypi.local:9090`
* Grafana should now be running at `http://raspberrypi.local:3000`

## Using a 32-bit Raspberry Pi

I've had [a problem similar to this one](https://github.com/prometheus/prometheus/issues/7483) with my 32-bit Raspberry Pi failing compacting when setup with a retention time of 1 year, and scraping every 10 seconds.

Pinning Prometheus to `1.7.2` that doesn't use `mmap` seems to have fixed it. To do that your `docker-compose.yml` might look like this:

```yaml
# docker-compose.yml
version: '3'
services:
    prometheus:
        container_name: prometheus
        # For a 32-bit Raspberry Pi
        image: prom/prometheus:v1.7.2
        build: ./prometheus
        volumes:
            - prometheus_data:/prometheus
        command:
            - '--config.file=/etc/prometheus/prometheus.yml'
            - '--web.console.libraries=/etc/prometheus/console_libraries'
            - '--web.console.templates=/etc/prometheus/consoles'
            - '--storage.tsdb.retention.time=1y'
            - '--web.enable-lifecycle'
        ports:
            - "9090:9090"
        restart: on-failure
```

## Prometheus.yml for multiple targets

If you're wanting to scrape from multiple targets, the `prometheus.yml` could look something like this:

```yaml
# prometheus.yml
global:
  scrape_interval: 5s
  external_labels:
    monitor: 'my-monitor'
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']
  - job_name: 'environment'
    static_configs:
      - targets: ['10.1.1.2:8000']
        labels:
          group: 'environment'
          location: 'Melbourne'
      - targets: ['11.1.1.2:8000']
        labels:
          group: 'environment'
          location: 'Adelaide'
alerting:
  alertmanagers:
    - scheme: http
      static_configs:
      - targets: ['alertmanager:9093']
#rule_files:
#  - 'alert.rules'
```

## To change the data retention time

The Prometheus part of your `docker-compose.yml` should add `--storage.tsdb.retention.time=1y` where 1y is 1 year, and should look something like this:

```yaml
# docker-compose.yml
version: '3'
services:
    prometheus:
        container_name: prometheus
        build: ./prometheus
        volumes:
            - prometheus_data:/prometheus
        command:
            - '--config.file=/etc/prometheus/prometheus.yml'
            - '--web.console.libraries=/etc/prometheus/console_libraries'
            - '--web.console.templates=/etc/prometheus/consoles'
            - '--storage.tsdb.retention.time=1y'
            - '--web.enable-lifecycle'
        ports:
            - "9090:9090"
        restart: on-failure
```

## To export your data

Use the [Snapshot API](https://prometheus.io/docs/prometheus/2.1/querying/api/#snapshot).

* Enable it by passing the flag when running Prometheus `--web.enable-admin-api`
* Curl the Snapshot API: `curl -XPOST http://raspberrypi.local:9090/api/v1/admin/tsdb/snapshot`

### Credit
Thanks to [finestructure](https://github.com/finestructure/blogpost-prometheus) for the base.

