# PMM

We will use PMM to get graphs from the metrics. The installation method will be via Docker container.

## Installing PMM server (on the app-pxc server)

- `yum install -y docker.x86_64`

- `service docker start`

- `docker pull percona/pmm-server:1;`

- ```
  docker create \
     -v /opt/prometheus/data \
     -v /opt/consul-data \
     -v /var/lib/mysql \
     -v /var/lib/grafana \
     --name pmm-data \
     percona/pmm-server:1 /bin/true;
  ```

- ```
  docker run -d \
     -p 80:80 \
     --volumes-from pmm-data \
     --name pmm-server \
     --restart always \
     percona/pmm-server:1;
  ```

## Installing PMM client (on the pxc nodes)

- `yum install -y pmm-client.x86_64;`
- `pmm-admin config --server 192.168.80.200`
- `pmm-admin add mysql`