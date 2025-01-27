# karma8

## Question 1
```bash
http {
    # new zone for rate limit
    limit_rate_after 0; # apply right after the start 
    limit_rate 1m; # rate limit 1 MB/s == 8 Mb/s

    server {
        listen 80;
        server_name your_domain.com;

        location /videos/ {
            # check the URL is still valid
            if ($arg_expires < $time_iso8601) {
                return 410;
            }

            # apply rate limit
            limit_rate 1m; 
            # allow byte-range requests
            
            add_header Accept-Ranges bytes;
            default_type video/mp4;
            
            alias /path/to/your/videos/;
        }
    }
}
```

## Question 2

```bash
SERVERS=$(cat servers.txt)

for server in $SERVERS; do
    echo "Updating Nginx on $server..."

    scp "$LOCAL_NGINX_CONF" "root@$server:$REMOTE_NGINX_CONF"

    if ssh "root@$server" "nginx -t"; then
        echo "Nginx configuration test passed on $server."

        ssh "root@$server" "systemctl restart nginx"

        echo "Nginx restarted on $server."
    else
        echo "Nginx configuration test failed on $server. Skipping restart."
    fi

    echo "Done with $server."
    echo "-------------------------"
done
```
## Question 3
fail2ban conf file:  
>maxretry = 60: max error count before block   
findtime = 180: time window for make a count (3 min)  
bantime = 2700: time to block (45 min)    

`/etc/fail2ban/jail.d/nginx.conf`:
```ini  
[nginx-errors]
enabled = true
filter = nginx-error
action = iptables-multiport[name=HTTP, port="http,https", protocol=tcp]
logpath = /var/log/nginx/error.log
maxretry = 60
findtime = 180
bantime = 2700
```

 
parse errors 403 (access forbidden) and 410 (gone) in `/etc/fail2ban/filter.d/nginx-error.conf `
```ini
[Definition]
failregex = ^\[error\] \d+#\d+: \*\d+ (?:access forbidden|gone), client: <HOST>, server: \S+, request: "\S+ \S+ HTTP/\d+\.\d+", host: "\S+"(?:, referrer: "\S+")?$
```
```fail2ban-client status nginx-errors```

OR  
Nginx + GeoIP   
```bash
http {
    limit_req_zone $binary_remote_addr zone=error_zone:10m rate=60r/m;

    server {
        listen 80;
        server_name your_domain.com;

        location / {
            limit_req zone=error_zone burst=60 nodelay;

            error_page 403 410 @block;
        }

        location @block {
            return 403 "You are blocked for 45 minutes.";
        }
    }
}
```
## Question 4
inventory.ini:  
```ini
[servers]
server1.example.com
server2.example.com
server3.example.com
...
server120.example.com
```

remount.yml:  
```bash
---
- name: Remount failed mount points
  hosts: servers
  become: yes
  vars:
    max_retries: 3
    mount_points:
      - /mnt/data
      - /mnt/backup

  tasks:
    - name: Remount mount points
      command: mount -o remount "{{ item.item }}"
      when: item.rc != 0  # only if mount point is not present
      register: remount_result
      retries: "{{ max_retries }}"
      delay: 5  # delay between retries
      until: remount_result is success
      loop: "{{ mount_check.results }}"

    - name: Verify mount points after remount
      command: findmnt --mountpoint "{{ item }}"
      loop: "{{ mount_points }}"
      register: verify_mount
      ignore_errors: yes

    - name: Fail if mount points are still not mounted
      fail:
        msg: "Mount point {{ item.item }} is still not mounted after {{ max_retries }} attempts."
      when: item.rc != 0
      loop: "{{ verify_mount.results }}"
```
apply: `ansible-playbook -i inventory.ini remount.yml`      
## Question 5

Ошибка Last_SQL_Errno: 1205 с сообщением Lock wait timeout exceeded; try restarting transaction указывает на проблему с блокировками (lock contention) в MySQL. Это происходит, когда транзакция не может получить блокировку в течение определённого времени (значение innodb_lock_wait_timeout), и MySQL прерывает её.


>сетевая проблема   
на мастер большая нагрузка   
слэйв перегружен и не успевает проиграть лог  
потюнить innodb_lock_wait_timeout и slave_parallel_workers   
## Question 7
```
rsync -avz -e "ssh -i /path/to/your/private_key" user@serverA:/path/to/source/directory/ user@serverB:/path/to/destination/directory/
```
## Question 8
1. htop
1. iostat
1. vmstat
1. netstat
1. /var/log/syslog
1. /var/log/nginx*
1. lsof
## Question 10
1. htop
1. iostat
1. vmstat
1. lsof
1. /var/syslog
1. /var/log/mysql.log
1. show processlist

## Question 11
low free space during backup process  
concurent backup processes: for example system backups and mysql backups
