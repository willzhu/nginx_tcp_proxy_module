Name
    nginx_tcp_proxy_module - support TCP proxy with Nginx

Installation
    Download the latest stable version of the release tarball of this module
    from github (<http://github.com/yaoweibin/nginx_tcp_proxy_module>)

    The development version of this module is here
    (<https://github.com/yaoweibin/nginx_tcp_proxy_module/tree/develop>). I
    have added the features of tcp_ssl_proxy, tcp_upstream_busyness,
    access_log.

    Grab the nginx source code from nginx.org (<http://nginx.org/>), for
    example, the version 1.2.1 (see nginx compatibility), and then build the
    source with this module:

        $ wget 'http://nginx.org/download/nginx-1.2.1.tar.gz'
        $ tar -xzvf nginx-1.2.1.tar.gz
        $ cd nginx-1.2.1/
        $ patch -p1 < /path/to/nginx_tcp_proxy_module/tcp.patch

        $ ./configure --add-module=/path/to/nginx_tcp_proxy_module

        $ make
        $ make install

Synopsis
    http {

        server {
            listen 80;

            location /status {
                check_status;
            }
        }
    }

    tcp {

        upstream cluster {
            # simple round-robin
            server 127.0.0.1:3306;
            server 127.0.0.1:1234;

            check interval=3000 rise=2 fall=5 timeout=1000;

            #check interval=3000 rise=2 fall=5 timeout=1000 type=ssl_hello;

            #check interval=3000 rise=2 fall=5 timeout=1000 type=http;
            #check_http_send "GET / HTTP/1.0\r\n\r\n";
            #check_http_expect_alive http_2xx http_3xx;
        }

        server {
            listen 8888;

            proxy_pass cluster;
        }
    }

Description
    This module actually include many modules: ngx_tcp_module,
    ngx_tcp_core_module, ngx_tcp_upstream_module, ngx_tcp_proxy_module,
    ngx_tcp_upstream_ip_hash_module. All these modules work togther to add
    the support of TCP proxy with Nginx. I also add other features: ip_hash,
    upstream server health check, status monitor.

    The motivation of writing these modules is Nginx's high performance and
    robustness. At first, I developed this module just for general TCP
    proxy. And now, this module is frequently used in websocket reverse
    proxying.

    You can't use the same listening port with HTTP modules.

Directives
  ngx_tcp_moodule
   tcp
    syntax: *tcp {...}*

    default: *none*

    context: *main*

    description: All the tcp related directives are contained in the tcp
    block.

    ngx_tcp_core_moodule

   server
    syntax: *server {...}*

    default: *none*

    context: *tcp*

    description: All the specific server directives are contained in the
    server block.

   listen
    syntax: *listen address:port [ bind ]*

    default: *none*

    context: *server*

    description: The same as listen
    (<http://wiki.nginx.org/NginxMailCoreModule#listen>).

   allow
    syntax: *allow [ address | CIDR | all ]*

    default: *none*

    context: *server*

    description: Directive grants access for the network or addresses
    indicated.

   deny
    syntax: *deny [ address | CIDR | all ]*

    default: *none*

    context: *server*

    description: Directive grants access for the network or addresses
    indicated.

   so_keepalive
    syntax: *so_keepalive on|off*

    default: *off*

    context: *main, server*

    description: The same as so_keepalive
    (<http://wiki.nginx.org/NginxMailCoreModule#so_keepalive>).

   tcp_nodelay
    syntax: *tcp_nodelay on|off*

    default: *on*

    context: *main, server*

    description: The same as tcp_nodelay
    (<http://wiki.nginx.org/NginxHttpCoreModule#tcp_nodelay>).

   timeout
    syntax: *timeout milliseconds*

    default: *60000*

    context: *main, server*

    description: set the timeout value with clients.

   server_name
    syntax: *server_name name fqdn_server_host*

    default: *The name of the host, obtained through gethostname()*

    context: *tcp, server*

    description: The same as server_name
    (<http://wiki.nginx.org/NginxMailCoreModule#server_name>).

   resolver
    syntax: *resolver address*

    default: *none*

    context: *tcp, server*

    description: DNS server

   resolver_timeout
    syntax: *resolver_timeout time*

    default: *30s*

    context: *tcp, server*

    description: Resolver timeout in seconds.

  ngx_tcp_upstream_module
   upstream
    syntax: *upstream {...}*

    default: *none*

    context: *tcp*

    description: All the upstream directives are contained in this block.
    The upstream server will be dispatched with round robin by default.

   server
    syntax: *server name [parameters]*

    default: *none*

    context: *upstream*

    description: Most of the parameters are the same as server
    (<http://wiki.nginx.org/NginxHttpUpstreamModule#server>). Default port
    is 80.

   check
    syntax: *check interval=milliseconds [fall=count] [rise=count]
    [timeout=milliseconds] [type=tcp|ssl_hello|smtp|mysql|pop3|imap]*

    default: *none, if parameters omitted, default parameters are
    interval=30000 fall=5 rise=2 timeout=1000*

    context: *upstream*

    description: Add the health check for the upstream servers. At present,
    the check method is a simple tcp connect.

    The parameters' meanings are:

    *   *interval*: the check request's interval time.

    *   *fall*(fall_count): After fall_count check failures, the server is
        marked down.

    *   *rise*(rise_count): After rise_count check success, the server is
        marked up.

    *   *timeout*: the check request's timeout.

    *   *type*: the check protocol type:

        1.  *tcp* is a simple tcp socket connect and peek one byte.

        2.  *ssl_hello* sends a client ssl hello packet and receives the
            server ssl hello packet.

        3.  *http* sends a http requst packet, recvives and parses the http
            response to diagnose if the upstream server is alive.

        4.  *smtp* sends a smtp requst packet, recvives and parses the smtp
            response to diagnose if the upstream server is alive. The
            response begins with '2' should be an OK response.

        5.  *mysql* connects to the mysql server, recvives the greeting
            response to diagnose if the upstream server is alive.

        6.  *pop3* recvives and parses the pop3 response to diagnose if the
            upstream server is alive. The response begins with '+' should be
            an OK response.

        7.  *imap* connects to the imap server, recvives the greeting
            response to diagnose if the upstream server is alive.

   check_http_send
    syntax: *check_http_send http_packet*

    default: *"GET / HTTP/1.0\r\n\r\n"*

    context: *upstream*

    description: If you set the check type is http, then the check function
    will sends this http packet to check the upstream server.

   check_http_expect_alive
    syntax: *check_http_expect_alive [ http_2xx | http_3xx | http_4xx |
    http_5xx ]*

    default: *http_2xx | http_3xx*

    context: *upstream*

    description: These status codes indicate the upstream server's http
    response is ok, the backend is alive.

   check_smtp_send
    syntax: *check_smtp_send smtp_packet*

    default: *"HELO smtp.localdomain\r\n"*

    context: *upstream*

    description: If you set the check type is smtp, then the check function
    will sends this smtp packet to check the upstream server.

   check_smtp_expect_alive
    syntax: *check_smtp_expect_alive [smtp_2xx | smtp_3xx | smtp_4xx |
    smtp_5xx]*

    default: *smtp_2xx*

    context: *upstream*

    description: These status codes indicate the upstream server's smtp
    response is ok, the backend is alive.

   check_shm_size
    syntax: *check_shm_size size*

    default: *(number_of_checked_upstream_blocks + 1) * pagesize*

    context: *tcp*

    description: If you store hundreds of serveres in one upstream block,
    the shared memory for health check may be not enough, you can enlarged
    it by this directive.

   check_status
    syntax: *check_status*

    default: *none*

    context: *location*

    description: Display the health checking servers' status by HTTP. This
    directive is set in the http block.

    The table field meanings are:

    *   *Index*: The server index in the check table

    *   *Name* : The upstream server name

    *   *Status*: The marked status of the server.

    *   *Busyness*: The number of connections which are connecting to the
        server.

    *   *Rise counts*: Count the successful checking

    *   *Fall counts*: Count the unsuccessful checking

    *   *Access counts*: Count the times accessing to this server

    *   *Check type*: The type of the check packet

    ngx_tcp_upstream_ip_hash_module

   ip_hash
    syntax: *ip_hash*

    default: *none*

    context: *upstream*

    description: the upstream server will be dispatched by ip_hash.

  ngx_tcp_proxy_module
   proxy_pass
    syntax: *proxy_pass host:port*

    default: *none*

    context: *server*

    description: proxy the request to the backend server. Default port is
    80.

   proxy_buffer
    syntax: *proxy_buffer size*

    default: *4k*

    context: *tcp, server*

    description: set the size of proxy buffer.

   proxy_connect_timeout
    syntax: *proxy_connect_timeout miliseconds*

    default: *60000*

    context: *tcp, server*

    description: set the timeout value of connection to backends.

   proxy_read_timeout
    syntax: *proxy_read_timeout miliseconds*

    default: *60000*

    context: *tcp, server*

    description: set the timeout value of reading from backends.

   proxy_send_timeout
    syntax: *proxy_send_timeout miliseconds*

    default: *60000*

    context: *tcp, server*

    description: set the timeout value of sending to backends.

Compatibility
    *   Nginx 0.7.65+

Notes
    The http_response_parse.rl and smtp_response_parse.rl are ragel
    (<http://www.complang.org/ragel/>) scripts , you can edit the script and
    compile it like this:

        $ ragel -G2 http_response_parse.rl
        $ ragel -G2 smtp_response_parse.rl

TODO
Known Issues
    *   This module can't use the same listening port with the HTTP module.

Changelogs
  v0.19
    *   add many check methods

  v0.1
    *   first release

Authors
    Weibin Yao(姚伟斌) *yaoweibin at gmail dot com*

Copyright & License
    This README template copy from agentzh (<http://github.com/agentzh>).

    I borrowed a lot of code from upstream and mail module from the nginx
    0.7.* core. This part of code is copyrighted by Igor Sysoev. And the
    health check part is borrowed the design of Jack Lindamood's healthcheck
    module healthcheck_nginx_upstreams
    (<http://github.com/cep21/healthcheck_nginx_upstreams>);

    This module is licensed under the BSD license.

    Copyright (C) 2011 by Weibin Yao <yaoweibin@gmail.com>.

    All rights reserved.

    Redistribution and use in source and binary forms, with or without
    modification, are permitted provided that the following conditions are
    met:

    *   Redistributions of source code must retain the above copyright
        notice, this list of conditions and the following disclaimer.

    *   Redistributions in binary form must reproduce the above copyright
        notice, this list of conditions and the following disclaimer in the
        documentation and/or other materials provided with the distribution.

    THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS
    IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED
    TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A
    PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT
    HOLDER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
    SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED
    TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR
    PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF
    LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING
    NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS
    SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

