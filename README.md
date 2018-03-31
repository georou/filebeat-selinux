# filebeat-selinux

Filebeat SELinux policy module for CentOS 7 & RHEL 7 systems with systemd

This policy module is created as a baseline. It aims to provide filebeat with the necessary allow rules to function.

!! You will need to add the specific permissions to allow filebeat to read logs that you want !!
Example, if you want to read appache/nginx logs, you will need to add `apache_read_log(filebeat_t)`.

I have added a Boolean (filebeat_can_read_all_system_logs) you can use that might make it simpler. But keep in mind, that this is a blanket allow to read all logs.

### "I'm getting dac_override and/or dac_read_search AVC denials"
If you're reading nginx/apache logs or any other log file that does not allow root (or if using separate a filebeat UID) to read the log, this will trigger a deny because in SELinux root doesn't have access anywhere.
You have a few options;
* Add root/filebeat as group with appropriate file mode bits
* Use ACL's to allow root/filebeat
* Allow everyone to read permission bit 644 <- least favourable option!
*Don't forgot to check directories for appropriate permissions too!*


## Installation
```sh
# Clone the repo
git clone https://github.com/georou/filebeat-selinux.git

# Optional - Copy relevant .if interface file to /usr/share/selinux/devel/include to expose them when building and for future modules
install -Dp -m 0644 -o root -g root filebeat.if /usr/share/selinux/devel/include/myapplications/filebeat.if

# Compile the selinux module (see below)

# Install the SELinux policy module. Compile it before hand to ensure proper compatibility (see below)
semodule -i filebeat.pp

# Restore all the correct context labels
restorecon -RvF /etc/filebeat \
                /etc/init.d/filebeat \
                /usr/lib/systemd/system/filebeat.service \
                /usr/bin/filebeat.sh \
                /usr/share/filebeat \
                /var/lib/filebeat \
                /var/log/filebeat

# Start filebeat
systemctl enable --now filebeat.service

# Ensure it's working
ps -eZ | grep filebeat
```

## How To Compile The Module Locally(Recommended before installing)
Ensure you have the `selinux-policy-devel` package installed.
```sh
# Ensure you have the devel packages
yum install selinux-policy-devel
# Change to the directory containing the .if, .fc & .te files
cd filebeat-selinux
make -f /usr/share/selinux/devel/Makefile filebeat.pp
semodule -i filebeat.pp
```

## Debugging and Troubleshooting

* If you're getting permission errors, uncomment permissive in the .te file and try again. Re-check logs for any issues.

* Easy way to add in allow rules is the below command, then copy or redirect into the .te module. Rebuild and re-install:
* Don't forget to actually look at what is suggested. audit2allow will most likely go for a coarse grained permission!

```sh
ausearch -m avc -ts recent | audit2allow -R
# If you get a could not open interface info [/var/lib/sepolgen/interface_info] error, install:
yum install policycoreutils-devel
```


## Compatibility Notes
Built on CentOS 7.4 at the time with:
```
selinux-policy-3.13.1-166.el7_4.4.noarch
policycoreutils-devel-2.5-17.1.el7.x86_64
selinux-policy-devel-3.13.1-166.el7_4.4.noarch
policycoreutils-2.5-17.1.el7.x86_64
selinux-policy-targeted-3.13.1-166.el7_4.4.noarch
policycoreutils-python-2.5-17.1.el7.x86_64
```

