Keepalived使用VRRP（Virtual Router Redundancy Protocol）協定來實現故障轉移，該協定**允許多個伺服器共享相同的虛擬IP地址**，當主要伺服器失效時，其他伺服器可以接管該IP地址並提供服務。

當Nginx和Keepalived結合在一起時，常見的架構是將多個Nginx伺服器配置為一個群集，**共享相同的虛擬IP地址**。Keepalived在這些Nginx伺服器之間進行健康檢查，監測主要伺服器的可用性。如果主要伺服器失效，Keepalived會將虛擬IP地址轉移到另一個正常運行的Nginx伺服器，以確保持續的服務可用性。這樣的架構提供了高可用性和負載平衡，以防止單點故障和分散流量。



## 實作

### 1.先確認要實作的機器，設定固定ip & 確認網卡編號

master ip: `192.168.22.91`

```bash
$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:0c:29:d8:cd:7c brd ff:ff:ff:ff:ff:ff
    altname enp2s1
    inet 192.168.22.91/24 brd 192.168.22.255 scope global ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fed8:cd7c/64 scope link 
       valid_lft forever preferred_lft forever
```

Backuo ip: `192.168.22.92`

```bash
$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:0c:29:8d:56:01 brd ff:ff:ff:ff:ff:ff
    altname enp2s1
    inet 192.168.22.92/24 brd 192.168.22.255 scope global ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fe8d:5601/64 scope link 
       valid_lft forever preferred_lft forever
```

### 2.在所有架設nginx伺服器上，安裝Keepalived

```yaml
sudo apt install -y install keepalived
```

### 3.配置Keepalived文件( **`/etc/keepalived/keepalived.conf`  )**

下方為`Master`

```yaml
global_defs {
    router_id Nginx_Master # 配置keepalived的名稱(唯一性)，Backup也要取唯一性的名稱
}

vrrp_script check_nginx {
    script "/etc/keepalived/nginx_check.sh" # 偵測 nginx 狀態的腳本路徑
    interval 2 # 檢查執行間隔為 2 秒
    weight 2 # 如果檢查失敗，將增加優先權 2 單位
}
vrrp_instance VI_1 {  # 定義 VI_1 為名稱，Backup也要是一樣的名字
    state MASTER      # 設定這台主機為master
    interface ens33   # 使用配置的網卡,預設會是eth0
    virtual_router_id 51  # on the a VRRP interface, set the same ID on all nodes
    priority 100   # 此為優先級，高的會成為master
    advert_int 1   # 間隔檢測時間

    authentication {  # 身份驗證參數(由於一個網域中不會只有一組keepalived,不同組的keepalived是透過auth找尋共同的機器
        auth_type PASS
        auth_pass 1234
    }

    track_script {     # 執行上方定義的vrrp_script
        check_nginx
    }

    virtual_ipaddress { # 透過此ip，進入後端nginx，也可以設置很多組ip
        192.168.22.100
    }
}
```

- 解釋(vrrp_instance)
    - **`state MASTER`**: 此設置指定了該VRRP實例的狀態為主（MASTER）。在VRRP中，一個實例可以是主（MASTER）或備份（BACKUP）狀態，主機處於主（MASTER）狀態時，它將成為虛擬IP地址的主要提供者。
    - **`interface eth0`**: 此設置指定了VRRP實例使用的網絡接口。在這個例子中，它是**`eth0`**。您需要根據您的系統配置和網絡接口名稱來設置這個值。
    - **`virtual_router_id 51`**: 此設置指定了虛擬路由器的唯一ID。在一個VRRP群組中，每個虛擬路由器ID都應該是唯一的，以區分不同的VRRP實例。
    - **`priority 100`**: 此設置指定了該VRRP實例的優先級。主機的優先級決定了它是否成為虛擬IP地址的主要提供者。具有較高優先級的主機將成為主（MASTER），如果存在多個具有相同優先級的主機，則使用具有較高IP地址的主機。
- 解釋(authentication)
    - **`auth_type PASS`**: 此設置指定了身份驗證類型為密碼（PASS）。這表示在VRRP通信中，使用密碼進行主機之間的身份驗證。
    - **`auth_pass 1234`**: 此設置指定了用於身份驗證的密碼。

下方為`backup`

```bash
global_defs {
    router_id Nginx_Backup  # 取唯一性的名稱
}

vrrp_script check_nginx {
    script "/etc/keepalived/nginx_check.sh"
    interval 2
    weight 2
} 
vrrp_instance VI_1 {   # 要跟Master名字一樣
    state BACKUP         
    interface ens33       
    virtual_router_id 51
    priority 50        # 低於master
    advert_int 1         

    authentication {    
        auth_type PASS
        auth_pass 1234
    }

    track_script { 
        check_nginx
    }

    virtual_ipaddress {  
        192.168.22.100
    }
}
```

### 4.編寫 Nginx 狀態偵測腳本

兩台伺服器上都需要編寫下面文件，可以根據路徑修改上方`vrrp_script`的script路徑。

我這裡是放在  `/etc/keepalived/nginx_check.sh` 

以下這個shell script主要是用於檢測nginx 進程是否在運行，如果nginx未運行，則會啟動nginx，在啟動nginx 後，腳本會暫停2 秒，等待nginx 進程完全啟動，如果nginx 進程啟動失敗或意外停止， 腳本會使用killall 指令終止所有名為keepalived 的進程。

```bash
#!/bin/bash
nginx_process_count=$(ps -C nginx --no-header | wc -l)
if [ $nginx_process_count -eq 0 ]; then
    /usr/nginx/sbin/nginx
    sleep 2
    if [ $(ps -C nginx --no-header | wc -l) -eq 0 ]; then
        killall keepalived
    fi
fi
```

給予shell script 權限

```bash
sudo chmod +x nginx_check.sh
```

### 5.上方都設定成功後，啟動Keepalived & 刷新nginx ，就可以成功了

```bash
sudo systemctl start keepalived
sudo systemctl enable keepalived
sudo nginx -s reload
```

### 6.若要測試，可以使用下方shell

```bash
#!/bin/bash
if [ $(ps -C nginx --no-header | wc -l) -eq 0 ]; then
    killall keepalived
fi
```

把nginx stop，curl `192.168.22.100` 會轉到BACKUP