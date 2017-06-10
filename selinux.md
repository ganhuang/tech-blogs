# SElinux

## SElinux modes
- `Enforcing` SELinux both logs and protects
- `Premissive` SElinux allows all interactions, 
   logs the interactions only, no reboot is required to go from
   enforcing to premissive or back again
- `Disable` Completely disables SElinux, reboot is needed to make effect

## CLI

- SElinux modes

```
# To display the current SELinux mode in effect
getenforce
# Add option `-Z` to display SElinux contexts
ps axZ
ls -Z /home
# Modify the current SElinux mode (/etc/selinux/config)
setenforce
```

- Changing the SELinux context of a file

```
# option 1: chcon modify the contexts directly by `-t`
chcon -t httpd_sys_content_t /test
# option 2(prefered): `semanage fcontext` modifys the rules that `restorecon` uses to set default file contexts
semanage fcontext -a -t httpd_sys_content_t '/test(/.*)?'
restorecon -RFvv /test
```

- Changing SELinux Booleans

```
semanage boolean -l
getsebool
setsebool -P
```
