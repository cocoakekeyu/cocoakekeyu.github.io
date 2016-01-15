---
category: openwrt
---

openwrt下samba设置起作用的机制是这样的：
openwrt在/etc/config/下面有一个samba的设置，注意：这个设置不符合samba软件本身的设置文件规范。openwr启动时，会用这个设置去替换掉相应的模板里的字段，生成一个符合samba设置文件规范的文件放到/tmp目录下。

## 一、设置/etc/config/samba

```bash
config samba
       option 'name'                    'openwrt'
       option 'workgroup'             'openwrt'
       option 'description'      'openwrt'
       option 'homes'                     '1'
             
config sambashare
       option 'name'                   'home'共享目录名
       option 'path'                   '/home'要共享的目录
       option 'read_only'             'no'
       option 'guest_ok'             'no'
       option 'create_mask'      '0700'
       option 'dir_mask'             '0700'
```
## 二、修改samba模板
/etc/samba/smb.conf.templet

```bash
[global]
       netbios name =|NAME| 
       workgroup = |WORKGROUP|
       server string = |DESCRIPTION|
       syslog = 10
       encrypt passwords = true
       passdb backend = smbpasswd
       obey pam restrictions = yes
       socket options = TCP_NODELAY
       unix charset= utf-8
       preferred master = yes
       os level = 20
       security = user
       guest account = nobody
       smb passwd file =/etc/samba/smbpasswd
```
