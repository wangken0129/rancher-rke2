# Rancher RKE2 Cluster



## 參考資料

https://docs.rke2.io/#how-is-this-different-from-rke-or-k3s

## RKE2 vs REK vs K3S

RKE2 combines the best-of-both-worlds from the 1.x version of RKE (hereafter referred to as RKE1) and K3s.

From K3s, it inherits the usability, ease-of-operations, and deployment model.

From RKE1, it inherits close alignment with upstream Kubernetes. In  places K3s has diverged from upstream Kubernetes in order to optimize  for edge deployments, but RKE1 and RKE2 can stay closely aligned with  upstream.

Importantly, RKE2 does not rely on Docker as RKE1 does. RKE1  leveraged Docker for deploying and managing the control plane components as well as the container runtime for Kubernetes. RKE2 launches control  plane components as static pods, managed by the kubelet. The embedded  container runtime is containerd.

## 架構

devop/suse

![image-20221026120148575](assets/image-20221026120148575.png)

DownStream K8S

![image-20221026120458643](assets/image-20221026120458643.png)

## 安裝作業系統 (No Swap)

關閉防火牆

![image-20221025140147837](assets/image-20221025140147837.png)

## 註冊作業系統

```
yast2
email: ken.wang@infotech.com.tw
Registry Code: B8203D65240BDA3A
```

![image-20221025145420736](assets/image-20221025145420736.png)

## 安裝RMS

https://kb.vmware.com/s/article/57122

### 安裝DNS Server

```
devop@rms:~> sudo zypper install -t pattern dhcp_dns_server
[sudo] root 的密碼：
devop@rms:~> sudo yast2
```

![image-20221025153349050](assets/image-20221025153349050.png)

![image-20221025153502151](assets/image-20221025153502151.png)

![image-20221025153543482](assets/image-20221025153543482.png)

![image-20221025153718281](assets/image-20221025153718281.png)

![image-20221025153903862](assets/image-20221025153903862.png)

![image-20221025154021819](assets/image-20221025154021819.png)

### 安裝Ngnix

1. 安裝nginx

   ```
   devop@rms:~> sudo zypper install nginx
   [sudo] root 的密碼：
   正在重新整理服務 'Basesystem_Module_15_SP4_x86_64'。
   正在重新整理服務 'Desktop_Applications_Module_15_SP4_x86_64'。
   正在重新整理服務 'SUSE_Linux_Enterprise_Server_15_SP4_x86_64'。
   正在重新整理服務 'Server_Applications_Module_15_SP4_x86_64'。
   正在載入套件庫資料...
   正在讀取已安裝的套件...
   正在解決套件相依性...
   
   將會安裝下列 1 個新的套件：
     nginx
   
   1 要安裝的新套件.
   全部下載大小：702.9 KiB。已快取：0 B。 完成操作後，將使用額外的 2.3 MiB。
   要繼續嗎？ [y/n/v/...? 顯示所有選項] (y): y
   正在取出 套件 nginx-1.21.5-150400.1.8.x86_64           (1/1), 702.9 KiB (已解開   2.3 MiB)
   正在取回︰ nginx-1.21.5-150400.1.8.x86_64.rpm ......................................[完成]
   
   正在檢查檔案衝突： .................................................................[完成]
   /usr/sbin/useradd -r -c User for nginx -d /var/lib/nginx -U nginx -s /usr/sbin/nologin
   (1/1) 正在安裝：nginx-1.21.5-150400.1.8.x86_64 .....................................[完成]
   
   ```

2. 設定ngnix

   ```
   devop@rms:/etc/nginx> sudo cat /etc/nginx/nginx.conf
   worker_processes 4;
   worker_rlimit_nofile 40000;
   
   
   load_module lib64/nginx/modules/ngx_stream_module.so;
   
   events {
       worker_connections 8192;
   }
   
   #http {
   #    server {
   #        listen         80;
   #        return 301 https://$host$request_uri;
   #    }
   #}
   
   stream {
       upstream rancher_servers_http {
           least_conn;
           server 192.168.33.41:80 max_fails=3 fail_timeout=5s;
           server 192.168.33.42:80 max_fails=3 fail_timeout=5s;
           server 192.168.33.43:80 max_fails=3 fail_timeout=5s;
       }
   
           #後端web server
       server {
   	    listen 80;
   	    proxy_pass rancher_servers_http;
       }
   
       upstream rancher_servers_https {
   	    least_conn;
               server 192.168.33.41:443 max_fails=3 fail_timeout=5s;
               server 192.168.33.42:443 max_fails=3 fail_timeout=5s;
               server 192.168.33.43:443 max_fails=3 fail_timeout=5s;
       }
       server {
   	    listen 443;
   	    proxy_pass rancher_servers_https;
       }
           upstream rancher_servers_6443 {
               least_conn;
               server 192.168.33.41:6443 max_fails=3 fail_timeout=5s;
               server 192.168.33.42:6443 max_fails=3 fail_timeout=5s;
               server 192.168.33.43:6443 max_fails=3 fail_timeout=5s;
       }
       server {
               listen 6443;
               proxy_pass rancher_servers_6443;
       }
       upstream rancher_servers_9345 {
               least_conn;
               server 192.168.33.41:9345 max_fails=3 fail_timeout=5s;
               server 192.168.33.42:9345 max_fails=3 fail_timeout=5s;
               server 192.168.33.43:9345 max_fails=3 fail_timeout=5s;
       }
       server {
               listen 9345;
               proxy_pass rancher_servers_9345;
       }
   	    
       log_format basic '$remote_addr [$time_local] '
                        '$protocol $status $bytes_sent $bytes_received '
                        '$session_time "$upstream_addr" '
                        '"$upstream_bytes_sent" "$upstream_bytes_received" "$upstream_connect_time"';
   
       access_log /var/log/nginx/access.log basic;
       error_log /var/log/nginx/error.log;
   
       
   }
   
   ```

3. 啟動服務

   ```
   devop@rms:/etc/nginx> sudo nginx -t
   nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
   nginx: configuration file /etc/nginx/nginx.conf test is successful
   
   devop@rms:/etc/nginx> sudo systemctl enable nginx --now
   Created symlink /etc/systemd/system/multi-user.target.wants/nginx.service → /usr/lib/systemd/system/nginx.service.
   ```

### 調整防火牆設定

```
devop@rms:/etc/nginx> sudo firewall-cmd --add-port=80/tcp --permanent
success
devop@rms:/etc/nginx> sudo firewall-cmd --add-port=443/tcp --permanent
success
devop@rms:/etc/nginx> sudo firewall-cmd --add-port=6443/tcp --permanent
success
devop@rms:/etc/nginx> sudo firewall-cmd --add-port=9345/tcp --permanent
success
devop@rms:/etc/nginx> sudo firewall-cmd --reload 
success
```

```
Inbound Rules for RKE2 Server Nodes
Protocol	Port	Source	Description
TCP	9345	RKE2 agent nodes	Kubernetes API
TCP	6443	RKE2 agent nodes	Kubernetes API
UDP	8472	RKE2 server and agent nodes	Required only for Flannel VXLAN
TCP	10250	RKE2 server and agent nodes	kubelet
TCP	2379	RKE2 server nodes	etcd client port
TCP	2380	RKE2 server nodes	etcd peer port
TCP	30000-32767	RKE2 server and agent nodes	NodePort port range
UDP	8472	RKE2 server and agent nodes	Cilium CNI VXLAN
TCP	4240	RKE2 server and agent nodes	Cilium CNI health checks
ICMP	8/0	RKE2 server and agent nodes	Cilium CNI health checks
TCP	179	RKE2 server and agent nodes	Calico CNI with BGP
UDP	4789	RKE2 server and agent nodes	Calico CNI with VXLAN
TCP	5473	RKE2 server and agent nodes	Calico CNI with Typha
TCP	9098	RKE2 server and agent nodes	Calico Typha health checks
TCP	9099	RKE2 server and agent nodes	Calico health checks
TCP	5473	RKE2 server and agent nodes	Calico CNI with Typha
UDP	8472	RKE2 server and agent nodes	Canal CNI with VXLAN
TCP	9099	RKE2 server and agent nodes	Canal CNI health checks
UDP	51820	RKE2 server and agent nodes	Canal CNI with WireGuard IPv4
UDP	51821	RKE2 server and agent nodes	Canal CNI with WireGuard IPv6/dual-stack
```

用ansible來執行開啟port (針對各個nodes)

```
wangken@wangken-MAC aps-rancher % cat inventory 
[rke2-cluster]
192.168.2.[71:73] ansible_ssh_pass=P@55w.rd

[downstream]
192.168.2.[74:78] ansible_ssh_pass=P@55w.rd

wangken@wangken-MAC aps-rancher % cat ansible.cfg 
[defaults]
inventory = /Users/wangken/ansible/infoserver/aps-rancher/inventory
remote_user = devop
sudo_user = root
host_key_checking = False

[privilege_escalation]
become = true
become_user = root
become_ask_pass = true


wangken@wangken-MAC aps-rancher % cat firewall.yml 
---
- hosts: rke2-cluster
  tasks:
    - name:
      firewalld:
        port: "{{ item }}"
        permanent: yes
        immediate: yes
        state: enabled
      with_items:
       - 9345/tcp
       - 6443/tcp
       - 8472/udp
       - 10250/tcp
       - 2379-2380/tcp
       - 30000-32767/tcp
       - 8472/udp
       - 4240/tcp
       - 179/tcp
       - 4789/udp
       - 5473/tcp
       - 9098/tcp
       - 9099/tcp
       - 8472/udp
       - 51820/udp
       - 51821/udp
 wangken@wangken-MAC aps-rancher % ansible-playbook firewall.yml
 TASK [firewalld] ***************************************************************
changed: [192.168.2.73] => (item=9345/tcp)
changed: [192.168.2.72] => (item=9345/tcp)
changed: [192.168.2.71] => (item=9345/tcp)
changed: [192.168.2.72] => (item=6443/tcp)
changed: [192.168.2.73] => (item=6443/tcp)
changed: [192.168.2.71] => (item=6443/tcp)
changed: [192.168.2.73] => (item=8472/udp)
changed: [192.168.2.72] => (item=8472/udp)
changed: [192.168.2.71] => (item=8472/udp)
changed: [192.168.2.73] => (item=10250/tcp)
changed: [192.168.2.71] => (item=10250/tcp)
changed: [192.168.2.72] => (item=10250/tcp)
changed: [192.168.2.72] => (item=2379-2380/tcp)
changed: [192.168.2.73] => (item=2379-2380/tcp)
changed: [192.168.2.71] => (item=2379-2380/tcp)
changed: [192.168.2.71] => (item=30000-32767/tcp)
changed: [192.168.2.73] => (item=30000-32767/tcp)
changed: [192.168.2.72] => (item=30000-32767/tcp)

```

### 安裝NFS Server 

https://documentation.suse.com/sles/15-SP1/html/SLES-all/cha-nfs.html

1. 建立NFS Server 目錄

   ```
   devop@rms:/> pwd
   /
   devop@rms:/> sudo mkdir nfs
   devop@rms:/> sudo chmod 777 nfs
   devop@rms:/> ll
   drwxrwxrwx   1 root root    0 11月  8 14:33 nfs
   ```

2. 安裝NFS Server套件 (SUSE OS)

   ```
   devop@rms:/home> sudo zypper install yast2-nfs-server
   下列 1 個推薦的套件已自動被選取：
     nfs-kernel-server
   
   將會安裝下列 2 個新的套件：
     nfs-kernel-server yast2-nfs-server
     
   ```

3. 設定nfs-server path

   ```
   devop@rms:/home> sudo vim /etc/exports 
   devop@rms:/home> cat /etc/exports 
   # See the exports(5) manpage for a description of the syntax of this file.
   # This file contains a list of all directories that are to be exported to 
   # other computers via NFS (Network File System).
   # This file used by rpc.nfsd and rpc.mountd. See their manpages for details
   # on how make changes in this file effective.
   /nfs 192.168.33.0/255.255.255.0(rw,sync,no_root_squash,insecure)
   
   ----
   
   # 參數
       * rw：read-write，可讀寫的權限；
       * ro：read-only，唯讀的權限；
       * sync：資料同步寫入到記憶體與硬碟當中；
       * async：資料會先暫存於記憶體當中，而非直接寫入硬碟。
       * no_root_squash：
         登入 NFS 主機使用分享目錄的使用者如果是 root，對於分享的目錄具有 root 的權限。
         極不安全，不建議使用。
       * root_squash：
         登入 NFS 主機使用分享目錄的使用者如果是 root，權限將被壓縮成匿名使用者，
         通常他的 UID 與 GID 都會變成 nobody(nfsnobody) 那個系統帳號的身份；
       * all_squash：
         不論登入 NFS 的使用者身份為何，都會被壓縮成為匿名使用者，通常是 nobody(nfsnobody) 。
       * anonuid：
         anon 意指 anonymous (匿名者)，自訂匿名使用者的 UID。 
       * anongid：自訂匿名使用者的是變成 GID。
   ```

4. 啟動nfs server、調整防火牆

   ```
   devop@rms:/home> sudo systemctl enable nfs-server --now
   Created symlink /etc/systemd/system/multi-user.target.wants/nfs-server.service → /usr/lib/systemd/system/nfs-server.service.
   
   devop@rms:~/nfs> sudo firewall-cmd --add-service=nfs --permanent
   [sudo] root 的密碼：
   success
   devop@rms:~/nfs> sudo firewall-cmd --reload
   success
   
   devop@rms:~/nfs> sudo showmount -e
   Export list for rms:
   /nfs 192.168.33.0/255.255.255.0
   ```

## 安裝RKE2-1

1. 安裝OS
2. 調整DNS 指向192.168.33.40
3. 關閉防火牆

### 安裝RKE2

1. 下載install.sh 

   ```
   devop@rke2-1:~> curl -sfL https://get.rke2.io --output install.sh
   devop@rke2-1:~> chmod +x install.sh
   ```

2. 新增設定檔

   ```
   devop@rke2-1:~> sudo mkdir -p /etc/rancher/rke2/
   [sudo] root 的密碼：
   devop@rke2-1:~> sudo vim /etc/rancher/rke2/config.yaml
   node-name:
     - "rke2-1"
   token: my-shared-secret
   tls-san:
     - rms.rancher.ken.lab
   ```

3. 安裝rke2 -- version 1.23.9

   ```
   devop@rke2-1:~> sudo INSTALL_RKE2_CHANNEL=v1.23.9+rke2r1 ./install.sh
   [WARN]  /usr/local is read-only or a mount point; installing to /opt/rke2
   [INFO]  finding release for channel v1.23.9+rke2r1
   [INFO]  using v1.23.9+rke2r1 as release
   [INFO]  downloading checksums at https://github.com/rancher/rke2/releases/download/v1.23.9+rke2r1/sha256sum-amd64.txt
   [INFO]  downloading tarball at https://github.com/rancher/rke2/releases/download/v1.23.9+rke2r1/rke2.linux-amd64.tar.gz
   [INFO]  verifying tarball
   [INFO]  unpacking tarball file to /opt/rke2
   [INFO]  updating tarball contents to reflect install path
   [INFO]  moving systemd units to /etc/systemd/system
   [INFO]  install complete; you may want to run:  export PATH=$PATH:/opt/rke2/bin
   --
   devop@rke2-1:~>  export PATH=$PATH:/opt/rke2/bin
   devop@rke2-1:~> sudo systemctl enable rke2-server --now
   Created symlink /etc/systemd/system/multi-user.target.wants/rke2-server.service → /etc/systemd/system/rke2-server.service.
   ```

   1. 調整kubectl 指令及kubeconfig >> 可把 kubectl、rke2.yaml  scp到 rms

   ```
   devop@rke2-1:~> mkdir .kube
   devop@rke2-1:~> sudo cp /etc/rancher/rke2/rke2.yaml .kube/config
   devop@rke2-1:~> sudo chown devop .kube/config 
   devop@rke2-1:~> sudo cp /var/lib/rancher/rke2/bin/kubectl /usr/local/bin/
   ----
   devop@rke2-1:~> sudo scp /etc/rancher/rke2/rke2.yaml devop@192.168.33.40:/home/devop/.kube/config
   The authenticity of host '192.168.33.40 (192.168.33.40)' can't be established.
   ECDSA key fingerprint is SHA256:NS1PCnKGGGiOsCVPSdoRDhX5jq0yhqhtEIr3pMNDSzw.
   Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
   Warning: Permanently added '192.168.33.40' (ECDSA) to the list of known hosts.
   Password: 
   rke2.yaml                                                                100% 2969     2.6MB/s   00:00    
   devop@rke2-1:~> sudo scp /var/lib/rancher/rke2/bin/kubectl  root@192.168.33.40:/usr/local/bin
   Password: 
   kubectl                                                                  100%   47MB 220.3MB/s   00:00    
   ```

4. 確認pod status (切到 rms來操作)

   ```
   devop@rms:~> kubectl get pods -A
   NAMESPACE     NAME                                                    READY   STATUS      RESTARTS   AGE
   kube-system   cloud-controller-manager-rke2-1                         1/1     Running     0          5m46s
   kube-system   etcd-rke2-1                                             1/1     Running     0          5m30s
   kube-system   helm-install-rke2-canal-9pg26                           0/1     Completed   0          5m35s
   kube-system   helm-install-rke2-coredns-d4j4q                         0/1     Completed   0          5m35s
   kube-system   helm-install-rke2-ingress-nginx-gz7db                   0/1     Completed   0          5m35s
   kube-system   helm-install-rke2-metrics-server-fx65q                  0/1     Completed   0          5m35s
   kube-system   kube-apiserver-rke2-1                                   1/1     Running     0          5m21s
   kube-system   kube-controller-manager-rke2-1                          1/1     Running     0          5m47s
   kube-system   kube-proxy-rke2-1                                       1/1     Running     0          5m43s
   kube-system   kube-scheduler-rke2-1                                   1/1     Running     0          5m47s
   kube-system   rke2-canal-qkrdv                                        2/2     Running     0          5m19s
   kube-system   rke2-coredns-rke2-coredns-545d64676-2q6x9               1/1     Running     0          5m20s
   kube-system   rke2-coredns-rke2-coredns-autoscaler-5dd676f5c7-5bwdl   1/1     Running     0          5m20s
   kube-system   rke2-ingress-nginx-controller-gxp6v                     1/1     Running     0          4m42s
   kube-system   rke2-metrics-server-6564db4569-98ghm                    1/1     Running     0          4m53s
   devop@rms:~> 
   
   ```

5. 安裝Helm

   ```
   devop@rms:~> wget https://get.helm.sh/helm-v3.8.2-linux-amd64.tar.gz
   --2022-10-25 16:55:33--  https://get.helm.sh/helm-v3.8.2-linux-amd64.tar.gz
   Resolving get.helm.sh (get.helm.sh)... 152.199.39.108, 2606:2800:247:1cb7:261b:1f9c:2074:3c
   Connecting to get.helm.sh (get.helm.sh)|152.199.39.108|:443... connected.
   HTTP request sent, awaiting response... 200 OK
   Length: 13633605 (13M) [application/x-tar]
   Saving to: ‘helm-v3.8.2-linux-amd64.tar.gz’
   
   helm-v3.8.2-linux-amd64.ta 100%[=======================================>]  13.00M  34.4MB/s    in 0.4s    
   
   2022-10-25 16:55:35 (34.4 MB/s) - ‘helm-v3.8.2-linux-amd64.tar.gz’ saved [13633605/13633605]
   devop@rms:~> tar zxvf helm-v3.8.2-linux-amd64.tar.gz
   linux-amd64/
   linux-amd64/helm
   linux-amd64/LICENSE
   linux-amd64/README.md
   devop@rms:~> sudo cp linux-amd64/helm /usr/local/bin/
   [sudo] root 的密碼：
   devop@rms:~> helm version
   version.BuildInfo{Version:"v3.8.2", GitCommit:"6e3701edea09e5d55a8ca2aae03a68917630e91b", GitTreeState:"clean", GoVersion:"go1.17.5"}
   ```

### install rancher and cert-manager

1. Add helm repo & Install Cert-manager

   ```
   devop@rms:~> helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
   "rancher-stable" has been added to your repositories
   devop@rms:~> kubectl create namespace cattle-system
   namespace/cattle-system created
   
   devop@rms:~> kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.7.1/cert-manager.crds.yaml
   customresourcedefinition.apiextensions.k8s.io/certificaterequests.cert-manager.io created
   customresourcedefinition.apiextensions.k8s.io/certificates.cert-manager.io created
   customresourcedefinition.apiextensions.k8s.io/challenges.acme.cert-manager.io created
   customresourcedefinition.apiextensions.k8s.io/clusterissuers.cert-manager.io created
   customresourcedefinition.apiextensions.k8s.io/issuers.cert-manager.io created
   customresourcedefinition.apiextensions.k8s.io/orders.acme.cert-manager.io created
   
   devop@rms:~> helm repo add jetstack https://charts.jetstack.io
   "jetstack" has been added to your repositories
   
   devop@rms:~> helm repo update
   Hang tight while we grab the latest from your chart repositories...
   ...Successfully got an update from the "jetstack" chart repository
   ...Successfully got an update from the "rancher-stable" chart repository
   Update Complete. ⎈Happy Helming!⎈
   
   devop@rms:~> helm install cert-manager jetstack/cert-manager \
   > --namespace cert-manager \
   > --create-namespace \
   > --version v1.7.1
   NAME: cert-manager
   LAST DEPLOYED: Tue Oct 25 17:01:20 2022
   NAMESPACE: cert-manager
   STATUS: deployed
   REVISION: 1
   TEST SUITE: None
   NOTES:
   cert-manager v1.7.1 has been deployed successfully!
   
   In order to begin issuing certificates, you will need to set up a ClusterIssuer
   or Issuer resource (for example, by creating a 'letsencrypt-staging' issuer).
   
   More information on the different types of issuers and how to configure them
   can be found in our documentation:
   
   https://cert-manager.io/docs/configuration/
   
   For information on how to configure cert-manager to automatically provision
   Certificates for Ingress resources, take a look at the `ingress-shim`
   documentation:
   
   https://cert-manager.io/docs/usage/ingress/
   
   devop@rms:~> kubectl get pods --namespace cert-manager
   NAME                                     READY   STATUS    RESTARTS   AGE
   cert-manager-76d44b459c-xnr7q            1/1     Running   0          75s
   cert-manager-cainjector-9b679cc6-8lk2m   1/1     Running   0          75s
   cert-manager-webhook-57c994b6b9-hssn2    1/1     Running   0          75s
   ```

2. 安裝Rancher

   ````
   devop@rms:~> helm install rancher rancher-stable/rancher --namespace cattle-system --set hostname=rms.rancher.ken.lab --version 2.6.6
   NAME: rancher
   LAST DEPLOYED: Tue Oct 25 17:05:13 2022
   NAMESPACE: cattle-system
   STATUS: deployed
   REVISION: 1
   TEST SUITE: None
   NOTES:
   Rancher Server has been installed.
   
   NOTE: Rancher may take several minutes to fully initialize. Please standby while Certificates are being issued, Containers are started and the Ingress rule comes up.
   
   Check out our docs at https://rancher.com/docs/
   
   If you provided your own bootstrap password during installation, browse to https://rms.rancher.ken.lab to get started.
   
   If this is the first time you installed Rancher, get started by running this command and clicking the URL it generates:
   
   ```
   echo https://rms.rancher.ken.lab/dashboard/?setup=$(kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}')
   ```
   
   To get just the bootstrap password on its own, run:
   
   ```
   kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}{{ "\n" }}'
   ```
   
   
   Happy Containering!
   
   -----
   密碼：
   devop@rms:~> kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}{{ "\n" }}'
   mc85v6z7s6tnfcrwpnd67pg4phlnnch64gwsd5gwkflsmt8bjqckgg
   ````

3. 瀏覽器連線 https://rms.rancher.ken.lab

   ![image-20221025170335200](assets/image-20221025170335200.png)

4. 登入後修改預設密碼
   admin / rancheradmin
   ![image-20221025170619277](assets/image-20221025170619277.png)

5. 登入測試
   ![image-20221025170758099](assets/image-20221025170758099.png)

## 安裝RKE2-2 

1. 安裝OS
2. 調整DNS 指向192.168.33.40
3. 關閉防火牆

### 安裝RKE2

1. 下載install.sh 

   ```
   devop@rke2-2:~> curl -sfL https://get.rke2.io --output install.sh
   devop@rke2-2:~> chmod +x install.sh
   ```

2. 新增設定檔

   ```
   devop@rke2-2:~> sudo mkdir -p /etc/rancher/rke2/
   [sudo] root 的密碼：
   devop@rke2-2:~> sudo vim /etc/rancher/rke2/config.yaml
   server: https://rms.rancher.ken.lab:9345
   node-name:
     - "rke2-2"
   token: my-shared-secret
   tls-san:
     - rms.rancher.ken.lab
   ```

3. 安裝rke2

   ```
   devop@rke2-2:~> sudo INSTALL_RKE2_CHANNEL=v1.23.9+rke2r1 ./install.sh
   ```

4. 啟動服務

   ```
   devop@rke2-2:~> export PATH=$PATH:/opt/rke2/bin
   devop@rke2-2:~> sudo systemctl enable rke2-server --now
   Created symlink /etc/systemd/system/multi-user.target.wants/rke2-server.service → /etc/systemd/system/rke2-server.service.
   
   devop@rke2-2:~> systemctl status rke2-server.service
   ● rke2-server.service - Rancher Kubernetes Engine v2 (server)
        Loaded: loaded (/etc/systemd/system/rke2-server.service; enabled; vendor preset: disabled)
        Active: active (running) since Tue 2022-10-25 17:53:05 CST; 17min ago
          Docs: https://github.com/rancher/rke2#readme
       Process: 5602 ExecStartPre=/bin/sh -xc ! /usr/bin/systemctl is-enabled --quiet nm-cloud-setup.service (code=exited, status=0/SUCCESS)
       Process: 5604 ExecStartPre=/sbin/modprobe br_netfilter (code=exited, status=0/SUCCESS)
       Process: 5605 ExecStartPre=/sbin/modprobe overlay (code=exited, status=0/SUCCESS)
      Main PID: 5606 (rke2)
         Tasks: 148
   
   ```

5. 切換到RMS 確認nodes狀態

   ```
   devop@rms:~> kubectl get nodes
   NAME     STATUS   ROLES                       AGE   VERSION
   rke2-1   Ready    control-plane,etcd,master   81m   v1.23.9+rke2r1
   rke2-2   Ready    control-plane,etcd,master   17m   v1.23.9+rke2r1
   ```

## 安裝RKE 2-3

1. 安裝OS
2. 調整DNS 指向192.168.33.40
3. 關閉防火牆

### 安裝RKE2

1. 下載install.sh 

   ```
   devop@rke2-3:~> curl -sfL https://get.rke2.io --output install.sh
   devop@rke2-3:~> chmod +x install.sh
   ```

2. 新增設定檔

   ```
   devop@rke2-3:~> sudo mkdir -p /etc/rancher/rke2/
   [sudo] root 的密碼：
   devop@rke2-3:~> sudo vim /etc/rancher/rke2/config.yaml
   server: https://rms.rancher.ken.lab:9345
   node-name:
     - "rke2-3"
   token: my-shared-secret
   tls-san:
     - rms.rancher.ken.lab
   ```

3. 安裝rke2

   ```
   devop@rke2-3:~> sudo INSTALL_RKE2_CHANNEL=v1.23.9+rke2r1 ./install.sh
   [WARN]  /usr/local is read-only or a mount point; installing to /opt/rke2
   [INFO]  finding release for channel v1.23.9+rke2r1
   [INFO]  using v1.23.9+rke2r1 as release
   [INFO]  downloading checksums at https://github.com/rancher/rke2/releases/download/v1.23.9+rke2r1/sha256sum-amd64.txt
   [INFO]  downloading tarball at https://github.com/rancher/rke2/releases/download/v1.23.9+rke2r1/rke2.linux-amd64.tar.gz
   [INFO]  verifying tarball
   [INFO]  unpacking tarball file to /opt/rke2
   [INFO]  updating tarball contents to reflect install path
   [INFO]  moving systemd units to /etc/systemd/system
   [INFO]  install complete; you may want to run:  export PATH=$PATH:/opt/rke2/bin
   ```

4. 啟動服務

   ```
   devop@rke2-3:~> export PATH=$PATH:/opt/rke2/bin
   devop@rke2-3:~> sudo systemctl enable rke2-server --now
   ```

5. 切換到RMS 確認Nodes

   ```
   devop@rms:~> kubectl get nodes
   NAME     STATUS   ROLES                       AGE   VERSION
   rke2-1   Ready    control-plane,etcd,master   91m   v1.23.9+rke2r1
   rke2-2   Ready    control-plane,etcd,master   27m   v1.23.9+rke2r1
   rke2-3   Ready    control-plane,etcd,master   58s   v1.23.9+rke2r1
   ```

   ![image-20221025181826167](assets/image-20221025181826167.png)



# 新增Down Stream K8s

## 架構 建議用v1.24以上

![image-20221026120534677](assets/image-20221026120534677.png)

## 作業系統與安裝Dokcer

要關閉防火牆、安裝Docker (用RKE2 不用安裝Docker)

```
需先註冊系統或是設定repo

devop@master01:~> sudo SUSEConnect -p sle-module-containers/15.4/x86_64
Registering system to SUSE Customer Center

Updating system details on https://scc.suse.com ...

Activating sle-module-containers 15.4 x86_64 ...
-> Adding service to system ...
-> Installing release package ...

Successfully registered system

---

devop@master01:~> sudo zypper -n install docker
正在重新整理服務 'Basesystem_Module_15_SP4_x86_64'。
正在重新整理服務 'Containers_Module_15_SP4_x86_64'。
正在重新整理服務 'SUSE_Linux_Enterprise_Server_15_SP4_x86_64'。
正在重新整理服務 'Server_Applications_Module_15_SP4_x86_64'。
正在載入套件庫資料...
正在讀取已安裝的套件...
正在解決套件相依性...

下列 1 個推薦的套件已自動被選取：
  git-core

將會安裝下列 7 個新的套件：
  catatonit containerd docker docker-bash-completion git-core libsha1detectcoll1 runc

7 要安裝的新套件.
全部下載大小：52.2 MiB。已快取：0 B。 完成操作後，將使用額外的 242.1 MiB。
要繼續嗎？ [y/n/v/...? 顯示所有選項] (y): y
正在取出 套件 libsha1detectcoll1-1.0.3-2.18.x86_64                (1/7),  23.2 KiB (已解開  45.8 KiB)
正在取回︰ libsha1detectcoll1-1.0.3-2.18.x86_64.rpm ...........................................[完成]
正在取出 套件 catatonit-0.1.5-3.3.2.x86_64                        (2/7), 257.2 KiB (已解開 696.5 KiB)
正在取回︰ catatonit-0.1.5-3.3.2.x86_64.rpm ...................................................[完成]
正在取出 套件 runc-1.1.4-150000.33.4.x86_64                       (3/7),   2.6 MiB (已解開   9.1 MiB)
正在取回︰ runc-1.1.4-150000.33.4.x86_64.rpm ....................................[完成 (288.0 KiB/s)]
正在取出 套件 containerd-1.6.6-150000.73.2.x86_64                 (4/7),  17.7 MiB (已解開  74.2 MiB)
正在取回︰ containerd-1.6.6-150000.73.2.x86_64.rpm ................................[完成 (3.6 MiB/s)]
正在取出 套件 git-core-2.35.3-150300.10.15.1.x86_64               (5/7),   4.8 MiB (已解開  26.6 MiB)
正在取回︰ git-core-2.35.3-150300.10.15.1.x86_64.rpm ..........................................[完成]
正在取出 套件 docker-20.10.17_ce-150000.166.1.x86_64              (6/7),  26.6 MiB (已解開 131.4 MiB)
正在取回︰ docker-20.10.17_ce-150000.166.1.x86_64.rpm .............................[完成 (7.9 MiB/s)]
正在取出 套件 docker-bash-completion-20.10.17_ce-150000.166.1.noarch
                                                                  (7/7), 121.3 KiB (已解開 113.6 KiB)
正在取回︰ docker-bash-completion-20.10.17_ce-150000.166.1.noarch.rpm .........................[完成]

正在檢查檔案衝突： ............................................................................[完成]
(1/7) 正在安裝：libsha1detectcoll1-1.0.3-2.18.x86_64 ..........................................[完成]
(2/7) 正在安裝：catatonit-0.1.5-3.3.2.x86_64 ..................................................[完成]
(3/7) 正在安裝：runc-1.1.4-150000.33.4.x86_64 .................................................[完成]
(4/7) 正在安裝：containerd-1.6.6-150000.73.2.x86_64 ...........................................[完成]
(5/7) 正在安裝：git-core-2.35.3-150300.10.15.1.x86_64 .........................................[完成]
Updating /etc/sysconfig/docker ...
(6/7) 正在安裝：docker-20.10.17_ce-150000.166.1.x86_64 ........................................[完成]
(7/7) 正在安裝：docker-bash-completion-20.10.17_ce-150000.166.1.noarch ........................[完成]

--- 啟動服務

devop@master01:~> sudo systemctl enable docker --now
Created symlink /etc/systemd/system/multi-user.target.wants/docker.service → /usr/lib/systemd/system/docker.service.

```



## 從Rancher管理介面新增Cluster

1. 點選Create
   ![image-20221026121450385](assets/image-20221026121450385.png)
   
2. 選擇Custom 要把RKE2的選項打勾 才可以選擇RKE2
   ![image-20221026121555947](assets/image-20221026121555947.png)
   ![image-20221027165708557](assets/image-20221027165708557.png)
   
3. 建立名稱: DownStreamK8S ，其他Default即可
   ![image-20221026121636567](assets/image-20221026121636567.png)
   
4. 針對各Node選擇不同的指令-Master
   ![image-20221026121758039](assets/image-20221026121758039.png)
   
5. 直接貼在Master01、Master02、Ｍaster03上
   ![image-20221026123232444](assets/image-20221026123232444.png)
   
6. 針對各Node選擇不同的指令-Master
   ![image-20221026123846849](assets/image-20221026123846849.png)
   
7. 直接貼在Woker01、Worker02
   ![image-20221026123951429](assets/image-20221026123951429.png)
   
8. 查看Log
   ![image-20221026124043002](assets/image-20221026124043002.png)
   ![image-20221026132912006](assets/image-20221026132912006.png)
   
9. 查看Cluster狀態
   ![image-20221027134018550](assets/image-20221027134018550.png)
   
   

# 安裝後的設定

## Authentication LDAP

1. 點到Users & Authentication
   ![image-20221028173919721](assets/image-20221028173919721.png)
2. 設定參數並Enable
   ![image-20221028181954045](assets/image-20221028181954045.png)
   ![image-20221028174539292](assets/image-20221028174539292.png)
   ![image-20221028182009242](assets/image-20221028182009242.png)
3. 指定特定Group可存取Rancher
   ![image-20221028182709975](assets/image-20221028182709975.png)
4. 登入測試
   ![image-20221028182305555](assets/image-20221028182305555.png)
5. 調整群組權限 Assign Global Roles
   ![image-20221028182403843](assets/image-20221028182403843.png)
6. 加入ranchers的群組為Administrator 並Save
   ![image-20221028182450209](assets/image-20221028182450209.png)
7. 確認權限
   ![image-20221028182514532](assets/image-20221028182514532.png)

## Storage設定

1. 在Local Cluster的infra project建立rancher-nfs-storage的namespace
   ![image-20221031171840318](assets/image-20221031171840318.png)

2. 建立namespace
   ![image-20221031172147900](assets/image-20221031172147900.png)

3. Storage Class的yaml

   ```
   devop@rms:~/config> cat storageclass.yaml 
   apiVersion: storage.k8s.io/v1
   kind: StorageClass
   metadata:
     annotations:
       # 設定為default StorageClass，如果建立pvc時不指定StorageClass，則會使用當前的StorageClass
       storageclass.kubernetes.io/is-default-class: "true"
     name: default-nfs-storage
     namespace: rancher-nfs-storage
   provisioner: provisioner-nfs # or choose another name, must match deployment's env PROVISIONER_NAME'
   parameters:
     # 刪除PVC時不會保留資料
     archiveOnDelete: "false"
   # 回收策略：刪除
   reclaimPolicy: Delete
   #允許pvc建立後擴容
   allowVolumeExpansion: true 
   ```

4. Deployment.yaml

   ```
   devop@rms:~/config> cat deployment.yaml 
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: nfs-client-provisioner
     labels:
       app: nfs-client-provisioner
     # replace with namespace where provisioner is deployed
     namespace: rancher-nfs-storage
   spec:
     replicas: 1
     strategy:
       type: Recreate
     selector:
       matchLabels:
         app: nfs-client-provisioner
     template:
       metadata:
         labels:
           app: nfs-client-provisioner
       spec:
         serviceAccountName: nfs-client-provisioner
         containers:
           - name: nfs-client-provisioner
             image: k8s.gcr.io/sig-storage/nfs-subdir-external-provisioner:v4.0.2
             volumeMounts:
               - name: nfs-client-root
                 mountPath: /persistentvolumes
             env:
               - name: PROVISIONER_NAME
                 value: provisioner-nfs
               - name: NFS_SERVER
                 value: 192.168.33.40
               - name: NFS_PATH
                 value: /nfs
         volumes:
           - name: nfs-client-root
             nfs:
               server: 192.168.33.40
               path: /nfs
   ```

5. rbac.yaml

   ```
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: nfs-client-provisioner
     # replace with namespace where provisioner is deployed
     namespace: rancher-nfs-storage
   ---
   kind: ClusterRole
   apiVersion: rbac.authorization.k8s.io/v1
   metadata:
     name: nfs-client-provisioner-runner
   rules:
     - apiGroups: [""]
       resources: ["persistentvolumes"]
       verbs: ["get", "list", "watch", "create", "delete"]
     - apiGroups: [""]
       resources: ["persistentvolumeclaims"]
       verbs: ["get", "list", "watch", "update"]
     - apiGroups: ["storage.k8s.io"]
       resources: ["storageclasses"]
       verbs: ["get", "list", "watch"]
     - apiGroups: [""]
       resources: ["events"]
       verbs: ["create", "update", "patch"]
     - apiGroups: [""]
       resources: ["services", "endpoints"]
       verbs: ["get","create","list", "watch","update"]
     - apiGroups: ["extensions"]
       resources: ["podsecuritypolicies"]
       resourceNames: ["nfs-client-provisioner"]
       verbs: ["use"]
   ---
   kind: ClusterRoleBinding
   apiVersion: rbac.authorization.k8s.io/v1
   metadata:
     name: run-nfs-client-provisioner
   subjects:
     - kind: ServiceAccount
       name: nfs-client-provisioner
       # replace with namespace where provisioner is deployed
       namespace: rancher-nfs-storage
   roleRef:
     kind: ClusterRole
     name: nfs-client-provisioner-runner
     apiGroup: rbac.authorization.k8s.io
   ---
   kind: Role
   apiVersion: rbac.authorization.k8s.io/v1
   metadata:
     name: leader-locking-nfs-client-provisioner
     # replace with namespace where provisioner is deployed
     namespace: rancher-nfs-storage
   rules:
     - apiGroups: [""]
       resources: ["endpoints"]
       verbs: ["get", "list", "watch", "create", "update", "patch"]
   ---
   kind: RoleBinding
   apiVersion: rbac.authorization.k8s.io/v1
   metadata:
     name: leader-locking-nfs-client-provisioner
     # replace with namespace where provisioner is deployed
     namespace: rancher-nfs-storage
   subjects:
     - kind: ServiceAccount
       name: nfs-client-provisioner
       # replace with namespace where provisioner is deployed
       namespace: rancher-nfs-storage
   roleRef:
     kind: Role
     name: leader-locking-nfs-client-provisioner
     apiGroup: rbac.authorization.k8s.io
   
   ```

6. NFS Server先設定好可連線的主機

   ```
   範例：
   [root@bastion auth]# cat /etc/exports
   /data/nfsshare	*(rw,sync,no_wdelay,no_root_squash,insecure)
   ---
   devop@rms:~/config> cat /etc/exports 
   /nfs 192.168.33.0/255.255.255.0(rw,sync,no_root_squash,insecure)
   ```

7. 將rbac.yaml 、storage class.yaml、deployment.yaml部署到rancher-nfs-storage的NS上

   ```
   devop@rms:~/config> kubectl create -f rbac.yaml -f storageclass.yaml -f deployment.yaml 
   serviceaccount/nfs-client-provisioner created
   clusterrole.rbac.authorization.k8s.io/nfs-client-provisioner-runner created
   clusterrolebinding.rbac.authorization.k8s.io/run-nfs-client-provisioner created
   role.rbac.authorization.k8s.io/leader-locking-nfs-client-provisioner created
   rolebinding.rbac.authorization.k8s.io/leader-locking-nfs-client-provisioner created
   storageclass.storage.k8s.io/default-nfs-storage created
   deployment.apps/nfs-client-provisioner created
   
   ---
   devop@rms:~/nfs> kubectl get all -n rancher-nfs-storage
   NAME                                          READY   STATUS    RESTARTS   AGE
   pod/nfs-client-provisioner-6b8787f55b-r4m4t   1/1     Running   0          5m19s
   
   NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
   deployment.apps/nfs-client-provisioner   1/1     1            1           5m19s
   
   NAME                                                DESIRED   CURRENT   READY   AGE
   replicaset.apps/nfs-client-provisioner-6b8787f55b   1         1         1       5m19s
   ```

8. 查看WebUI，確認deployment、StorageClass
   ![image-20221108141107082](assets/image-20221108141107082.png)
   
9. 新增測試Claim 確認狀態為bound

   ```
   devop@rms:~/nfs> cat test-claim.yaml 
   kind: PersistentVolumeClaim
   apiVersion: v1
   metadata:
     name: test-claim
     annotations:
       #storageclass 名稱
       volume.beta.kubernetes.io/storage-class: "default-nfs-storage"
   spec:
     #訪問模式
     accessModes:
       - ReadWriteMany
     resources:
       requests:
         #請求資料大小
         storage: 1024Mi
         
   ---
   devop@rms:~/nfs> kubectl create -f test-claim.yaml
   persistentvolumeclaim/test-claim created
   
   devop@rms:~/nfs> kubectl get pv,pvc
   NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                STORAGECLASS          REASON   AGE
   persistentvolume/pvc-f058cf4a-cc7b-4e44-97c3-aa1bfc654b65   1Gi        RWX            Delete           Bound    default/test-claim   default-nfs-storage            33m
   
   NAME                               STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          AGE
   persistentvolumeclaim/test-claim   Bound    pvc-f058cf4a-cc7b-4e44-97c3-aa1bfc654b65   1Gi        RWX            default-nfs-storage   37m
   ```

10. 新增測試pod 確認可寫入

    ```
    devop@tpeinfoapsrms01:~/nfs> cat test-pod.yaml 
    kind: Pod
    apiVersion: v1
    metadata:
      name: test-pod
    spec:
      containers:
      - name: test-pod
        image: busybox:latest
        command:
          - "/bin/sh"
        args:
          - "-c"
          - "touch /mnt/SUCCESS && exit 0 || exit 1"
        volumeMounts:
          - name: nfs-pvc
            mountPath: "/mnt"
      restartPolicy: "Never"
      volumes:
        - name: nfs-pvc
          persistentVolumeClaim:
            claimName: test-claim
            
    ---
    devop@rms:~/nfs> kubectl create -f test-pod.yaml 
    pod/test-pod created
    
    devop@rms:~/nfs> kubectl get pv,pvc
    NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                STORAGECLASS          REASON   AGE
    persistentvolume/pvc-f058cf4a-cc7b-4e44-97c3-aa1bfc654b65   1Gi        RWX            Delete           Bound    default/test-claim   default-nfs-storage            35m
    
    NAME                               STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          AGE
    persistentvolumeclaim/test-claim   Bound    pvc-f058cf4a-cc7b-4e44-97c3-aa1bfc654b65   1Gi        RWX            default-nfs-storage   38m
    
    ---
    
    ```

11. WebUI
    ![image-20221031174825937](assets/image-20221031174825937.png)

12. 刪除測試用的pv,pvc

    ```
    devop@rms:~/nfs> kubectl delete -f test-pod.yaml -f test-claim.yaml 
    pod "test-pod" deleted
    persistentvolumeclaim "test-claim" deleted
    ```
    
13. 同樣設定也放到DownStream K8s ，建立時也可透過WEB UI來將yaml檔放到Cluster
    ![image-20221031175458133](assets/image-20221031175458133.png)
    ![image-20221031175603727](assets/image-20221031175603727.png)
    ![image-20221031175616696](assets/image-20221031175616696.png)

    

## 安裝Logging

1. 點到local的cluster
2. 建立project (infra)
3. 點選Apps --> Charts --> logging
   ![image-20221028163035932](assets/image-20221028163035932.png)
4. Default
   ![image-20221028163149621](assets/image-20221028163149621.png)
5. 開始安裝
   ![image-20221028163228433](assets/image-20221028163228433.png)
6. 重整網頁後即會出現
   ![image-20221028163610607](assets/image-20221028163610607.png)
7. 後續可再針對後端的logging 機制
8. 

## 安裝Monitoring

https://www.ibm.com/docs/en/cloud-private/3.2.0?topic=console-namespace-is-stuck-in-terminating-state

1. 點到local的cluster

2. 建立project (infra)
   ![image-20221028161139089](assets/image-20221028161139089.png)

3. 點選Apps --> Charts --> monitoring
   ![](assets/image-20221028160838561.png)

4. 點選Monitoring -- > Install  (安裝到Infra Project)
   ![image-20221028163730065](assets/image-20221028163730065.png)

   

5. Admin API Enable、PV
   ![image-20221028163815459](assets/image-20221028163815459.png)
   ![image-20221031180012545](assets/image-20221031180012545.png)

6. Grafana Enable
   ![image-20221031180129747](assets/image-20221031180129747.png)

7. 點選Install後重新整理WEB UI，即會出現Monitoring

   ![image-20221031180517689](assets/image-20221031180517689.png)

8. 點選Grafana確認資料
   ![image-20221031180605633](assets/image-20221031180605633.png)

9. 查詢Grafana預設帳號密碼
   會放在cattle-monitoring-system的 Secret裡 (帳密為admin/prom-operator)
   ![image-20221031180724481](../../../../../../../assets/image-20221031180724481.png)

## 安裝Neuvector

https://www.twblogs.net/a/6225a6dfc1f9c8b6ea0c84d2

1. 建立NameSpace

   ```
   
   ```

   

2. 建立RBAC

3. 123

4. 123

5. 123

6. 123

7. 123

# 升級Rancher Cluster

https://docs.ranchermanager.rancher.io/getting-started/installation-and-upgrade/install-upgrade-on-a-kubernetes-cluster/upgrades

## 安裝Rancher Backup

1. 在Apps --> Charts 搜尋rancher backups
   ![image-20221108153112348](assets/image-20221108153112348.png)

2. 點選install
   ![image-20221108153148832](assets/image-20221108153148832.png)

3. 設定儲存位置 (Storage Class)

   ![image-20221108153231799](assets/image-20221108153231799.png)

4. 點選install
   ![image-20221108153312905](assets/image-20221108153312905.png)
   ![image-20221108153324294](assets/image-20221108153324294.png)

5. 重整Web UI，出現Rancher Backups
   ![image-20221108153426191](assets/image-20221108153426191.png)

6. 點選Rancher Backups --> Create
   ![image-20221108153514800](assets/image-20221108153514800.png)

7. 設定名稱、參數等，點選Create
   ![image-20221108153603733](assets/image-20221108153603733.png)

8. 確認備份完成
   ![image-20221108153642511](assets/image-20221108153642511.png)

9. 123

10. 123

11. 123

12. 123

13. 123



## 升級Rancher Cluster

1. 確認Rancher Cluster

   ```
   devop@rms:~/nfs> helm get values rancher -n cattle-system
   USER-SUPPLIED VALUES:
   hostname: rms.rancher.ken.lab
   ---
   devop@rms:~> helm get values rancher -n cattle-system -o yaml > values.yaml
   devop@rms:~> cat values.yaml 
   hostname: rms.rancher.ken.lab
   ```

2. 確認rancher helm chart

   ```
   devop@rms:~> helm repo list
   NAME          	URL                                              
   rancher-stable	https://releases.rancher.com/server-charts/stable
   jetstack      	https://charts.jetstack.io                       
   neuvector     	https://neuvector.github.io/neuvector-helm/  
   ```

3. update helm chart

   ```
   devop@rms:~> helm repo update
   Hang tight while we grab the latest from your chart repositories...
   ...Successfully got an update from the "rancher-stable" chart repository
   ...Successfully got an update from the "neuvector" chart repository
   ...Successfully got an update from the "jetstack" chart repository
   Update Complete. ⎈Happy Helming!⎈
   ```

4. 確認目前rancher 最新版本

   ```
   devop@rms:~> helm search repo rancher-stable
   NAME                  	CHART VERSION	APP VERSION	DESCRIPTION                                       
   rancher-stable/rancher	2.6.9        	v2.6.9     	Install Rancher Server to manage Kubernetes clu...
   ```

5. 確認更新項目

   ```
   devop@rms:~> helm fetch rancher-stable/rancher --version=2.6.9
   ---
   devop@rms:~> tar xvf rancher-2.6.9.tgz 
   rancher/Chart.yaml
   rancher/values.yaml
   rancher/templates/NOTES.txt
   rancher/templates/_helpers.tpl
   rancher/templates/clusterRoleBinding.yaml
   rancher/templates/deployment.yaml
   rancher/templates/ingress.yaml
   rancher/templates/issuer-letsEncrypt.yaml
   rancher/templates/issuer-rancher.yaml
   rancher/templates/post-delete-hook-cluster-role-binding.yaml
   rancher/templates/post-delete-hook-cluster-role.yaml
   rancher/templates/post-delete-hook-config-map.yaml
   rancher/templates/post-delete-hook-job.yaml
   rancher/templates/post-delete-hook-psp.yaml
   rancher/templates/post-delete-hook-service-account.yaml
   rancher/templates/pvc.yaml
   rancher/templates/secret.yaml
   rancher/templates/service.yaml
   rancher/templates/serviceAccount.yaml
   rancher/.helmignore
   rancher/README.md
   rancher/scripts/post-delete-hook.sh
   ----
   devop@rms:~> tree rancher
   rancher
   ├── Chart.yaml
   ├── README.md
   ├── scripts
   │   └── post-delete-hook.sh
   ├── templates
   │   ├── clusterRoleBinding.yaml
   │   ├── deployment.yaml
   │   ├── _helpers.tpl
   │   ├── ingress.yaml
   │   ├── issuer-letsEncrypt.yaml
   │   ├── issuer-rancher.yaml
   │   ├── NOTES.txt
   │   ├── post-delete-hook-cluster-role-binding.yaml
   │   ├── post-delete-hook-cluster-role.yaml
   │   ├── post-delete-hook-config-map.yaml
   │   ├── post-delete-hook-job.yaml
   │   ├── post-delete-hook-psp.yaml
   │   ├── post-delete-hook-service-account.yaml
   │   ├── pvc.yaml
   │   ├── secret.yaml
   │   ├── serviceAccount.yaml
   │   └── service.yaml
   └── values.yaml
   
   2 directories, 21 files
   
   ```

6. 執行更新

   ````
   devop@rms:~> helm upgrade rancher rancher-stable/rancher --namespace cattle-system -f values.yaml --version=2.6.9
   
   ---
   
   Release "rancher" has been upgraded. Happy Helming!
   NAME: rancher
   LAST DEPLOYED: Tue Nov  8 15:52:00 2022
   NAMESPACE: cattle-system
   STATUS: deployed
   REVISION: 2
   TEST SUITE: None
   NOTES:
   Rancher Server has been installed.
   
   NOTE: Rancher may take several minutes to fully initialize. Please standby while Certificates are being issued, Containers are started and the Ingress rule comes up.
   
   Check out our docs at https://rancher.com/docs/
   
   If you provided your own bootstrap password during installation, browse to https://rms.rancher.ken.lab to get started.
   
   If this is the first time you installed Rancher, get started by running this command and clicking the URL it generates:
   
   ```
   echo https://rms.rancher.ken.lab/dashboard/?setup=$(kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}')
   ```
   
   To get just the bootstrap password on its own, run:
   
   ```
   kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}{{ "\n" }}'
   ```
   
   
   Happy Containering!
   
   ````

7. 確認status

   ```
   devop@rms:~> kubectl get pod -n cattle-system
   NAME                               READY   STATUS              RESTARTS       AGE
   helm-operation-trgz9               0/2     Completed           0              17m
   rancher-6c8f689f57-47hfd           0/1     ContainerCreating   0              43s
   rancher-6c8f689f57-g666k           0/1     ContainerCreating   0              43s
   rancher-7fd65d9cd6-m9gt7           1/1     Running             2 (7d1h ago)   13d
   rancher-7fd65d9cd6-vjcpk           1/1     Running             2 (7d1h ago)   13d
   rancher-webhook-5b65595df9-9xsrc   1/1     Running             2 (7d1h ago)   13d
   
   devop@rms:~> kubectl get pod -n cattle-system
   NAME                             READY   STATUS      RESTARTS   AGE
   helm-operation-ffdn6             0/2     Completed   0          13m
   helm-operation-kjqgc             0/2     Completed   0          14m
   helm-operation-tfbhr             0/2     Completed   0          13m
   helm-operation-trgz9             0/2     Completed   0          33m
   rancher-6c8f689f57-47hfd         1/1     Running     0          16m
   rancher-6c8f689f57-g666k         1/1     Running     0          16m
   rancher-6c8f689f57-nkfkb         1/1     Running     0          15m
   rancher-webhook-d6f8b6cb-xzjxc   1/1     Running     0          13m
   ```

8. 從Web UI確認版本 v2.6.9
   ![image-20221108160654097](assets/image-20221108160654097.png)

9. 123

10. 123

## Rolling Backup Rancher

1. 確認先前backup的檔案
   ![image-20221108162822587](assets/image-20221108162822587.png)

2. 點選Restore --> Create，選擇備份檔案
   Prune選項：現行的resource若不在備份檔案的話會刪除。

   ![image-20221108162904330](assets/image-20221108162904330.png)

3. 點選Create即會開始執行restore
   ![image-20221108163112147](assets/image-20221108163112147.png)

4. 確認cattle-system status，Rancher Cluster服務會中斷 (約3分鐘)

   ```
   devop@rms:~> kubectl get po -n cattle-system
   NAME                             READY   STATUS        RESTARTS   AGE
   helm-operation-trgz9             0/2     Completed     0          59m
   rancher-6c8f689f57-4jsmd         0/1     Terminating   0          1s
   rancher-6c8f689f57-dcvsk         0/1     Terminating   0          1s
   rancher-6c8f689f57-h7blc         0/1     Terminating   0          1s
   rancher-webhook-d6f8b6cb-xzjxc   1/1     Running       0          39m
   ```

   ![image-20221108163320830](assets/image-20221108163320830.png)

5. 還原後Rancher版本不會還原，但resoruce會還原 (cattle-monitoring-system)
   ![image-20221108170614349](assets/image-20221108170614349.png)

6. 還原rancher版本要使用helm來roll back

7. 3123

8. 132













