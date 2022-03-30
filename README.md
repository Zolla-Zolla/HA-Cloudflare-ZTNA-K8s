# # Cloudfare Auto-Scaling Zero-Trust on k8s

So you want to do Cloudflare Zero Trust, but you're worried about having a single point of failure from a single tunnel.  What can you do?!  Let's solve this problem with Kubernetes! 

This process below will give to the ability to create zero trust and also continue to the next level.  This will allow you to access systems defined in Cloudflare Zero Trust Design.  In this example we will configure access through Cloudlare Zero-Trust to allow connections to the Rancher application managing my K8 clusters

## What are we doing?

1.  Simple Install Cloudflared on Alpine Linux
2.  Add Application to the Tunnel
3.  Moving to K8 for Autoscalling and Redundancy

## Simple Install Cloudflared on Alpine Linux

On the Alpine Linux system you would like to install on, running the following will download the package locally, and create the connection to your existing Cloudflare Zero Trust account

```bash
mkdir /opt/cloudflared
cd /opt/cloudflared/
wget https://github.com/cloudflare/cloudflared/releases/download/2022.3.0/cloudflared-linux-amd64
chmod +x cloudflared-linux-amd64
ln -s /opt/cloudflared/cloudflared-linux-amd64 /usr/bin/cloudflared
cloudflared login
```
When get the following output to paste into a browser to continue
```bash
Please open the following URL and log in with your Cloudflare account:

https://dash.cloudflare.com/argotunnel?callback=https%3A%2F%2Flogin.cloudflareaccess.org%2FMAxI9vv-XnavcbLD_7YAVwltNyzbmAzvPZSvjWaWcTM%3D

Leave cloudflared running to download the cert automatically.
```
After completion you can create your secure tunnel as well as a DNS entry for it (change TunnelName and ServerName)
```bash
cloudflared tunnel create TunnelName
cloudflared tunnel route dns TunnelName ServerName
```

## Adding Devices to the Tunnel
During the authentication login process a file was created in /etc/cloudflared/config.yml you should deit this file.  At the top of the file will have tunnel and the credentials file. But adding information under the ingress rules matters.  Below is an example of a service needing https and one with just http.  It does make a difference to let the "-service: http_status:404" be the last line.  When editing make sure to follow spacing as it can lead to the file failing to load.
```bash
tunnel: 12345678-1234-1234-1234-123456789abc
credentials-file: /root/.cloudflared/12345678-1234-1234-1234-123456789abc.json
warp-routing:
  enabled: true

ingress:
  - hostname: rancher.domain.xxx
    service: https://192.168.1.15
    originRequest:
      noTLSVerify: true
  - hostname: service2.domain.xxx
    service: http://192.168.1.20:port
  - service: http_status:404
```
Once complete you need to create the DNS entry in Cloudflare for each service. To do so run the following from the example:
```bash
cloudflared tunnel route dns TunnelName rancher
cloudflared tunnel route dns TunnelName service2
```

## Moving to K8 for Autoscalling and Redundancy

By choosing to move further you should have your K8 environment up and functioning.  In my environment I'm leveraging rancher to manage my cluster.  The first step in this process is to create a secret. that can be passed to your pod when the deployment happens.  To do this you will have to locate your stored connection information.  This is typically located in /root/.cloudflared 
```bash
cd /root/.cloudflared
kubectl create secret generic tunnel-credentials --from-file=credentials.json=<YOUR .JSON FILE NAME>
```
Create a cloudflared.yaml file example below where you insert your config old ingress information from (/etc/cloudflared/config.yml)
```bash
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudflared
spec:
  selector:
    matchLabels:
      app: cloudflared
  replicas: 1
  template:
    metadata:
      labels:
        app: cloudflared
        lbtype: external
    spec:
      containers:
        - name: cloudflared
          image: cloudflare/cloudflared:2022.3.4
          args:
            - tunnel
            - --config
            - /etc/cloudflared/config/config.yaml
            - run
          volumeMounts:
            - name: config
              mountPath: /etc/cloudflared/config
              readOnly: true
            - name: creds
              mountPath: /etc/cloudflared/creds
              readOnly: true
      volumes:
        - name: creds
          secret:
            secretName: tunnel-credentials
        - name: config
          configMap: 
            name: cloudflared-config
            items:
              - key: config.yaml
                path: config.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: cloudflared-config
data:
  config.yaml: |
    tunnel: cloudzolla
    credentials-file: /etc/cloudflared/creds/credentials.json
    no-autoupdate: true
    ingress:
      - hostname: rancher.domain.xxx
        service: https://192.168.1.15
        originRequest:
          noTLSVerify: true
     - hostname: service2.domain.xxx
       service: http://192.168.1.20:port
     - service: http_status:404
```

Within Rancher set your replicas to the desired number of nodes
<kbd>
  <img width="1721" alt="image" src="https://user-images.githubusercontent.com/95427400/160814013-4673b7cf-d3c2-4575-92db-b5908fa369e4.png">
<kbd>

By looking at the workloads you will notice under Pods there are now two workers running, one on each node
<kbd>
  <img width="1742" alt="image" src="https://user-images.githubusercontent.com/95427400/160815003-1554ec2c-a715-4d8c-a449-882c3a95102e.png">
<kbd>
  
Then by looking in the cloudflared Zero Trust under tunnels notice that there are redundant tunnels
<kbd>
  <img width="1160" alt="image" src="https://user-images.githubusercontent.com/95427400/160814558-9312da5b-376c-450d-b25c-8f12ca2c4341.png">
<kbd>
