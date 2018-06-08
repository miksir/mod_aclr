# mod_aclr

This module purpose is to work with "nginx -- apache" scheme. Only apache 1.3.x supported.

Apache 2.x version https://github.com/defanator/mod_aclr2 by Andrey Belov

For mass-hosting usually you can't use light http server (such as nginx). It's because clients of hosting wants to use .htaccess file. You can use nginx only as reverse proxy server for proxing all request to apache. But this scheme has big overhead for big static files - after request this file passed to nginx and caching at disk. As result, apache spend time for send file and you have double load for HDD system.

Nginx has nice solution for backend-controled file transfer. If backend return special http header X-Accel-Redirect with file name as argument (no body required), nginx can send this file to client directly.

mod_aclr stands as default handler in apache after all others handlers (init directive must be first in load modules list). At this stage apache usually ready for send static file (all other actions, as php, allow/deny, etc - already runs). Module check own setup and send X-Accel-Redirect with zero content length. Nginx receive this header and start to send file.

mod_aclr also check X-Accel-Internal header and running only if this header exists. You must set this header for use mod_aclr. If header missed, mod_aclr state is off - it's helps use apache also directly without nginx. Value of this header added to value of returned X-Accel-Redirect - look expample for understand this.

Directives
----------
```
AccelRedirectSet {On|Off}
Default: Off
Context: server config, virtual host, directory

Turn on or off module action

AccelRedirectSize size[k|M]
Default: -1
Context: server config, virtual host, directory

Minimal file size for send special header. 
For small files may be better send file directly. 
Our advise - 10-20k or not set.

AccelRedirectDebug level
Default: 0
Context: server config

Debug level. 0 - turn off debug. Debug levels can be from 1 to 4.
```
Config of nginx
---------------

It's example of nginx config for use with module.
```
server {
        listen x.x.x.x;
        server_name  server.name.ru;
        location / {
            proxy_pass          http://127.0.0.1:80;
            proxy_set_header    X-Real-IP  $remote_addr;
            # we are set X-Accel-Internal to /myinternal1. 
            # As result, if we asking /mydir/myfile.mp3, 
            # mod_aclr return /myinternal1/mydir/myfile.mp3
            # its help us catch this redirect to right root
            proxy_set_header    X-Accel-Internal /myinternal1;
            proxy_set_header    Host $http_host;
        }
        location /myinternal1/ {
            # yep, here we catch redirect
            root        /home/server.name.ru/htdocs;
            rewrite   ^/myinternal1/(.*)$ /$1 break;
            # result file is /home/server.name.ru/htdocs/mydir/myfile.mp3
            internal;
        }
}
```
Second variant - more compact config for mass hosting. Here only one server block for all client servers and server host mapped to root dir by nginx's map directive.
```
map $int_host $root {
    default /path/to/server/no_server_found/;
    server1.ru /path/to/server/1/;
    server2.ru /path/to/server/2/;
}

server {
     listen x.x.x.x;
     location / {
            proxy_pass          http://127.0.0.1:80;
            proxy_set_header    X-Real-IP  $remote_addr;
            proxy_set_header    X-Accel-Internal /internal;
            proxy_set_header    Host $http_host;
     }
      location /internal/ {
            internal;
            set $int_host $http_host;
            root /$root;
            rewrite ^/internal/(.*)$ /$1 break;
      }
}
```
Install
-------

You can install module via apxs:
/usr/local/apache/bin/apxs -c mod_aclr.c - compile
/usr/local/apache/bin/apxs -i mod_aclr.so - installing

After installing add to httpd.conf the following:   
LoadModule aclr_module libexec/mod_aclr.so
AddModule mod_aclr.c
at the top of all others

Attention!  Is this module will not be loaded first, all other default-handler modues will be blocked.
