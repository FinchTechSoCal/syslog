# Syslog Server Config Files

Syslog config for cor-log.

[Official Documentation](https://syslog-ng.github.io/admin-guide/README)

## Prerequisites
* Linux Server
* Storage Area (mounted) for log files
  * /syslogs/ in our case

## Install/Setup

Create an easy way to find our syslogs & set the permissions:
```bash
sudo ln -s /mnt/sdb1 /syslogs
ln -s /mnt/sdb1 ~/syslogs
sudo chown -R $USER:$USER /syslogs/
```

Set a cron job to auto compress files older than 1 day & auto remove files older than 1 year+1 day:
```bash
crontab -l | { cat; echo "0 2 * * * /bin/find /syslogs/ -daystart -mtime +366 -type f -exec rm {} \;"; } | crontab -
crontab -l | { cat; echo "0 3 * * * /bin/find /syslogs/ -daystart -mtime +1 -type f -exec xz -v {} \;"; } | crontab -
```

Add syslog-ng APT repo [link](https://github.com/syslog-ng/syslog-ng?tab=readme-ov-file#adding-the-apt-repository): 
```bash
wget -qO - https://ose-repo.syslog-ng.com/apt/syslog-ng-ose-pub.asc | sudo gpg --dearmor -o /etc/apt/keyrings/syslog-ng-ose.gpg
echo "deb [signed-by=/etc/apt/keyrings/syslog-ng-ose.gpg] https://ose-repo.syslog-ng.com/apt/ stable <os>-<codename>[-<architecture>]" | sudo tee /etc/apt/sources.list.d/syslog-ng-ose.list > /dev/null
sudo apt update -y
```

Install syslog-ng:
```bash
sudo apt install syslog-ng -y
```

Fix a couple things I noticed (as of 2026-02-12):
```bash
sed -i 's/#SYSLOGNG_OPTS="--no-caps"/#SYSLOGNG_OPTS="--no-caps"\nSYSLOGNG_OPTS=""/g' /etc/default/syslog-ng
curl -Ls https://raw.githubusercontent.com/syslog-ng/syslog-ng/refs/heads/develop/lib/multi-line/smart-multi-line.fsm -o /usr/share/syslog-ng/smart-multi-line.fsm
sed -i 's/@version: 3.38/#@version: 3.38\n@version: 4.3/g' /etc/syslog-ng/syslog-ng.conf
# https://support.oneidentity.com/syslog-ng-premium-edition/kb/4380261/warning-your-configuration-file-uses-an-obsoleted-keyword-please-update-your-configuration-keyword-stats_freq-change-use-the-stats-block
sed -i 's/stats_freq(0)/stats(freq(0))/g' /etc/syslog-ng/syslog-ng.conf
```

Stage & set permission for our listener/config file (/etc/syslog-ng/conf.d/network.conf):
```bash
sudo touch /etc/syslog-ng/conf.d/network.conf
sudo chown -R $USER:$USER /etc/syslog-ng/conf.d/network.conf
```

Build our listener/config file (/etc/syslog-ng/conf.d/network.conf):
```bash
echo "## syslog-ng configuration file.

# 2026-02-12

options {
        chain_hostnames(no);
        keep_hostname(yes);
        log_fifo_size(2048);
        log_msg_size(8192);
        time_reopen (10);
        use_dns(no);
        dns_cache(no);
        use_fqdn(yes);
        ts_format(iso);

        ## Configure the directory owner, group, and permissions.
        create_dirs (yes);
        dir_owner(\"1000\");
        dir_group(\"1000\");
        dir_perm(0755);

        ## Configure the file owner, group, and permissions
        owner(\"1000\");
        group(\"1000\");
        perm(0644);
        };


source s_network {
    syslog(ip(0.0.0.0) port(514) transport(\"udp\"));
    syslog(ip(0.0.0.0) port(514) transport(\"tcp\"));
};

destination d_logs {
    file(\"/syslogs/\$HOST/\$YEAR-\$MONTH-\$DAY.log\" create_dirs(yes));
};

log { source(s_network); destination(d_logs); };
" >  /etc/syslog-ng/conf.d/network.conf
```

Syslog-ng needs to be restarted to take the new config:
```bash
sudo systemctl restart syslog-ng
```

## Troubleshooting

This section needs some cleanup
```bash
# Handy commands for troubleshooting
# tcpdump -i ens192 'port 514'                                                             ## see the flow over a port as events are received (or not)
# netstat -tulpn | grep LISTEN                                                             ## check open/listening ports
# /usr/sbin/syslog-ng -F -p /var/run/syslogd.pid                                           ## run syslog-ng and see more errors
# 0 5 * * * /bin/find /syslogs/ -type f -name \*.log -mtime +1 -exec rm {} \;              ## add this crontab to delete files off every day at 5am older than 1 day
# multitail -s 2 /syslogs/*/*/*.log  /opt/splunk/var/log/splunk/splunkd.log                ## monitor all the files in the splunk dir
# syslog-ng-ctl stats                                                                      ## See the stats for each filter
# sudo systemctl status syslog-ng                                                          ## is syslog-ng running & what status
# journalctl -xefu syslog-ng                                                               ## syslog-ng log in long form
#
#
# unxz file.xz to extract
```

## Resources
* https://syslog-ng.github.io/admin-guide/README
* https://www.splunk.com/blog/2016/03/11/using-syslog-ng-with-splunk.html
* https://github.com/syslog-ng/syslog-ng
### Docker
* https://www.syslog-ng.com/community/b/blog/posts/central-log-server-docker
* https://hub.docker.com/r/balabit/syslog-ng/



## To Do
