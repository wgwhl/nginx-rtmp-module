# NGINX-based Media Streaming Server
## nginx-rtmp-module


### Project blog

  http://nginx-rtmp.blogspot.com

### Wiki manual

  https://github.com/arut/nginx-rtmp-module/wiki/Directives

### Google group

  https://groups.google.com/group/nginx-rtmp

  https://groups.google.com/group/nginx-rtmp-ru (Russian)

### Donation page (Paypal etc)

  http://arut.github.com/nginx-rtmp-module/

### Features

* RTMP/HLS/MPEG-DASH live streaming

* RTMP Video on demand FLV/MP4,
  playing from local filesystem or HTTP

* Stream relay support for distributed
  streaming: push & pull models

* Recording streams in multiple FLVs

* H264/AAC support

* Online transcoding with FFmpeg

* HTTP callbacks (publish/play/record/update etc)

* Running external programs on certain events (exec)

* HTTP control module for recording audio/video and dropping clients

* Advanced buffering techniques
  to keep memory allocations at a minimum
  level for faster streaming and low
  memory footprint

* Proved to work with Wirecast, FMS, Wowza,
  JWPlayer, FlowPlayer, StrobeMediaPlayback,
  ffmpeg, avconv, rtmpdump, flvstreamer
  and many more

* Statistics in XML/XSL in machine- & human-
  readable form

* Linux/FreeBSD/MacOS/Windows

### Build

cd to NGINX source directory & run this:

    ./configure --add-module=/path/to/nginx-rtmp-module
    make
    make install

Several versions of nginx (1.3.14 - 1.5.0) require http_ssl_module to be
added as well:

    ./configure --add-module=/path/to/nginx-rtmp-module --with-http_ssl_module

For building debug version of nginx add `--with-debug`

    ./configure --add-module=/path/to-nginx/rtmp-module --with-debug

[Read more about debug log](https://github.com/arut/nginx-rtmp-module/wiki/Debug-log)

### Windows limitations

Windows support is limited. These features are not supported

* execs
* static pulls
* auto_push

### RTMP URL format

    rtmp://rtmp.example.com/app[/name]

app -  should match one of application {}
         blocks in config

name - interpreted by each application
         can be empty


### Multi-worker live streaming

Module supports multi-worker live
streaming through automatic stream pushing
to nginx workers. This option is toggled with
rtmp_auto_push directive.


### Example nginx.conf

    rtmp {

        server {

            listen 1935;

            chunk_size 4000;

            # TV mode: one publisher, many subscribers
            application mytv {

                # enable live streaming
                live on;

                # record first 1K of stream
                record all;
                record_path /tmp/av;
                record_max_size 1K;

                # append current timestamp to each flv
                record_unique on;

                # publish only from localhost
                allow publish 127.0.0.1;
                deny publish all;

                #allow play all;
            }

            # Transcoding (ffmpeg needed)
            application big {
                live on;

                # On every pusblished stream run this command (ffmpeg)
                # with substitutions: $app/${app}, $name/${name} for application & stream name.
                #
                # This ffmpeg call receives stream from this application &
                # reduces the resolution down to 32x32. The stream is the published to
                # 'small' application (see below) under the same name.
                #
                # ffmpeg can do anything with the stream like video/audio
                # transcoding, resizing, altering container/codec params etc
                #
                # Multiple exec lines can be specified.

                exec ffmpeg -re -i rtmp://localhost:1935/$app/$name -vcodec flv -acodec copy -s 32x32
                            -f flv rtmp://localhost:1935/small/${name};
            }

            application small {
                live on;
                # Video with reduced resolution comes here from ffmpeg
            }

            application webcam {
                live on;

                # Stream from local webcam
                exec_static ffmpeg -f video4linux2 -i /dev/video0 -c:v libx264 -an
                                   -f flv rtmp://localhost:1935/webcam/mystream;
            }

            application mypush {
                live on;

                # Every stream published here
                # is automatically pushed to
                # these two machines
                push rtmp1.example.com;
                push rtmp2.example.com:1934;
            }

            application mypull {
                live on;

                # Pull all streams from remote machine
                # and play locally
                pull rtmp://rtmp3.example.com pageUrl=www.example.com/index.html;
            }

            application mystaticpull {
                live on;

                # Static pull is started at nginx start
                pull rtmp://rtmp4.example.com pageUrl=www.example.com/index.html name=mystream static;
            }

            # video on demand
            application vod {
                play /var/flvs;
            }

            application vod2 {
                play /var/mp4s;
            }

            # Many publishers, many subscribers
            # no checks, no recording
            application videochat {

                live on;

                # The following notifications receive all
                # the session variables as well as
                # particular call arguments in HTTP POST
                # request

                # Make HTTP request & use HTTP retcode
                # to decide whether to allow publishing
                # from this connection or not
                on_publish http://localhost:8080/publish;

                # Same with playing
                on_play http://localhost:8080/play;

                # Publish/play end (repeats on disconnect)
                on_done http://localhost:8080/done;

                # All above mentioned notifications receive
                # standard connect() arguments as well as
                # play/publish ones. If any arguments are sent
                # with GET-style syntax to play & publish
                # these are also included.
                # Example URL:
                #   rtmp://localhost/myapp/mystream?a=b&c=d

                # record 10 video keyframes (no audio) every 2 minutes
                record keyframes;
                record_path /tmp/vc;
                record_max_frames 10;
                record_interval 2m;

                # Async notify about an flv recorded
                on_record_done http://localhost:8080/record_done;

            }


            # HLS

            # For HLS to work please create a directory in tmpfs (/tmp/hls here)
            # for the fragments. The directory contents is served via HTTP (see
            # http{} section in config)
            #
            # Incoming stream must be in H264/AAC. For iPhones use baseline H264
            # profile (see ffmpeg example).
            # This example creates RTMP stream from movie ready for HLS:
            #
            # ffmpeg -loglevel verbose -re -i movie.avi  -vcodec libx264
            #    -vprofile baseline -acodec libmp3lame -ar 44100 -ac 1
            #    -f flv rtmp://localhost:1935/hls/movie
            #
            # If you need to transcode live stream use 'exec' feature.
            #
            application hls {
                live on;
                hls on;
                hls_path /tmp/hls;
            }

            # MPEG-DASH is similar to HLS

            application dash {
                live on;
                dash on;
                dash_path /tmp/dash;
            }
        }
    }

    # HTTP can be used for accessing RTMP stats
    http {

        server {

            listen      8080;

            # This URL provides RTMP statistics in XML
            location /stat {
                rtmp_stat all;

                # Use this stylesheet to view XML as web page
                # in browser
                rtmp_stat_stylesheet stat.xsl;
            }

            location /stat.xsl {
                # XML stylesheet to view RTMP stats.
                # Copy stat.xsl wherever you want
                # and put the full directory path here
                root /path/to/stat.xsl/;
            }

            location /hls {
                # Serve HLS fragments
                types {
                    application/vnd.apple.mpegurl m3u8;
                    video/mp2t ts;
                }
                root /tmp;
                add_header Cache-Control no-cache;
            }

            location /dash {
                # Serve DASH fragments
                root /tmp;
                add_header Cache-Control no-cache;
            }
        }
    }


### Multi-worker streaming example

    rtmp_auto_push on;

    rtmp {
        server {
            listen 1935;

            application mytv {
                live on;
            }
        }
    }
    
    
### nginx.conf 文件的列子
  
    
#user  nobody;
#worker_processes  1; #运行在Windows上时，设置为1，因为Windows不支持Unix domain socket
worker_processes  auto; #1.3.8和1.2.5以及之后的版本

#worker_cpu_affinity  0001 0010 0100 1000; #只能用于FreeBSD和Linux
#worker_cpu_affinity  auto; #1.9.10以及之后的版本

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;

#如果此模块被编译为动态模块并且要使用与RTMP相关的功能时，必须指定下面的配置项并且它必须位于events配置项之前，否则NGINX启动时不会加载此模块或者加载失败
#load_module modules/ngx_http_flv_live_module.so;

events {
    worker_connections  4096;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_iso8601] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;

    upstream AppServers {    
	#ip_hash;
        server 192.168.252.144:9090 down;   
        server 192.168.252.145:8080 weight=2;   
        server 192.168.252.146:6060;   
        server 192.168.252.147:7070 backup;   
    }

    server {
        listen       80;
        server_name  192.168.252.143;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        location / {
	    #proxy_pass http://www.baidu.com;
	    #proxy_pass http://AppServers;
	    #重定向
    	    #rewrite ^ http://localhost:8080;
            root   html;
            index  index.html index.htm;
        }

	location /remote_redirect {
            # no domain name here, only ip
            rewrite ^.*$ rtmp://192.168.252.143/rooms/abcd? permanent;
        }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
	location = /favicon.ico {
            log_not_found off;
            access_log off;
        }

        location /hls {  #添加视频流存放地址。
    	    types {
        	application/vnd.apple.mpegurl m3u8;
	        video/mp2t ts;
            }
       	    #访问权限开启，否则访问这个地址会报403
            autoindex on;
       	    alias /usr/local/nginx/html/hls; #视频流存放地址，与上面的hls_path相对应，这里root和alias的区别可自行百度
            expires -1;
            add_header 'Cache-Control' 'no-cache';
            #防止跨域问题
            add_header 'Access-Control-Allow-Origin' '*';
            add_header 'Access-Control-Allow-Credentials' 'true';
            add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS';
            add_header 'Access-Control-Allow-Headers' 'DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type';
        }

	# http://example.com[:port]/dir?[port=xxx&]app=myapp&stream=mystream
	# 参数dir用于匹配http配置块中的location块（更多详情见下文）。
	# HTTP默认端口为80, 如果使用了其他端口，必须指定:port。
	# RTMP默认端口为1935，如果使用了其他端口，必须指定port=xxx。
	# 参数app用来匹配application块，但是如果请求的app出现在多个server块中，并且这些server块有相同的地址和端口配置，
	# 那么还需要用匹配主机名的server_name配置项来区分请求的是哪个application块，否则，将匹配第一个application块。
	# 参数stream用来匹配发布流的streamname
	# ffplay "http://192.168.252.143/live?app=hls&stream=abcd"  # live表示http块中的loaction的名称，app表示rtmp块中的application, stream表示推送的流名称
	location /live {
            flv_live on; #打开HTTP播放FLV直播流功能
            chunked_transfer_encoding on; #支持'Transfer-Encoding: chunked'方式回复

            add_header 'Access-Control-Allow-Origin' '*'; #添加额外的HTTP头
            add_header 'Access-Control-Allow-Credentials' 'true'; #添加额外的HTTP头
        }

        location /stat {
            #push和pull状态的配置
            rtmp_stat all;
            rtmp_stat_stylesheet stat.xsl;
        }

        location /stat.xsl {
            root /usr/local/nginx/html/rtmp; #指定stat.xsl的位置
        }

        #如果需要JSON风格的stat, 不用指定stat.xsl
        #但是需要指定一个新的配置项rtmp_stat_format
        #location /stat {
        #    rtmp_stat all;
        #    rtmp_stat_format json;
        #}

	# ffplay -i http://192.168.252.143/dash/abcd.mpd
	location /dash {
	    # Serve DASH fragments
            root /tmp;
            add_header 'Cache-Control' 'no-cache';
        }

        location /control {
            rtmp_control all; #rtmp控制模块的配置
        }
    }

    # another virtual host using mix of IP-, name-, and port-based configuration
    #
    server {
        listen       8080;
    #    listen       somename:8080;
        server_name  192.168.252.143;

        location / {
	    include uwsgi_params;
            uwsgi_pass 192.168.252.143:5000;
        }
	location /testvue {
		alias /home/whl;
	}
	location = /favicon.ico {
            log_not_found off;
            access_log off;
        }

    }


    # HTTPS server
    #
    #server {
    #    listen       443 ssl;
    #    server_name  localhost;

    #    ssl_certificate      cert.pem;
    #    ssl_certificate_key  cert.key;

    #    ssl_session_cache    shared:SSL:1m;
    #    ssl_session_timeout  5m;

    #    ssl_ciphers  HIGH:!aNULL:!MD5;
    #    ssl_prefer_server_ciphers  on;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}

}


rtmp_auto_push on; #因为Nginx可能开启多个子进程，这个选项表示推流时，媒体流会发布到多个子进程
rtmp_auto_push_reconnect 1s;
rtmp_socket_dir /tmp; #多个子进程情况下，推流时，最开始只有一个子进程在竞争中接收到数据，然后它再relay给其他子进程，他们之间通过unix domain socket传输数据，这个选项表示unix domain socket的路径

rtmp {
    out_queue           4096;
    out_cork            8;
    max_streams         128; #Nginx能接受的最大的推流数
    timeout             15s;
    drop_idle_publisher 15s;

    log_interval 5s; #log模块在access.log中记录日志的间隔时间，对调试非常有用
    log_size     1m; #log模块用来记录日志的缓冲区大小

    chunk_size 4096; 

    server {
        listen 1935; #Nginx监听的RTMP推流/拉流端口，可以省略，默认监听1935
        #server_name www.test.*; #用于虚拟主机名后缀通配
        chunk_size 4096; 
	#notify_method get;

	access_log logs/rtmp_access.log;

        on_connect http://192.168.252.143:8080/api/rtmp/on_connect;
	publish_notify on;
        on_play http://192.168.252.143:8080/api/rtmp/on_play;
        on_publish http://192.168.252.143:8080/api/rtmp/on_publish;
        on_done http://192.168.252.143:8080/api/rtmp/on_done;
        on_play_done http://192.168.252.143:8080/api/rtmp/on_play_done;
        on_publish_done http://192.168.252.143:8080/api/rtmp/on_publish_done;
        on_record_done http://192.168.252.143:8080/api/rtmp/on_record_done;
        on_update http://192.168.252.143:8080/api/rtmp/on_update;
        #notify_update_timeout 10s;	    
        notify_update_strict on;

	# track client info
	exec_play /usr/bin/bash -c "echo $addr $pageurl >> /tmp/clients";
	exec_publish /usr/bin/bash -c "echo $addr $flashver >> /tmp/publishers";
	# convert recorded file to mp4 format
	exec_record_done ffmpeg -y -i $path -acodec libmp3lame -ar 44100 -ac 1 -vcodec libx264 $path.mp4;

        application hls {
            live on; #当推流时，RTMP路径中的APP（RTMP中一个概念），匹配hls时，开启直播
            hls on; #这个参数把直播服务器改造成实时回放服务器
	    wait_key on; #对视频切片进行保护，这样就不会产生马赛克了
            hls_path /usr/local/nginx/html/hls; #视频流存放地址，切片视频存放位置
            hls_fragment 5s; #每个视频切片时长
            hls_playlist_length 15s; #总共可回看的时间，这里设置15秒
            hls_continuous on; #连续模式。
            hls_cleanup on;    #对多余的切片进行删除。
            hls_nested on;     #嵌套模式。
	    gop_cache on; ##开启GOP（Group of Picture）缓存，这个是减少播放延迟的选项
	}

	#rtmp://localhost/app/movie?a=100&b=face&foo=bar then a, b & foo are also sent with callback
	# TV mode: one publisher, many subscribers
        application rooms {
            # enable live streaming
            live on;
	    gop_cache on; ##开启GOP（Group of Picture）缓存，这个是减少播放延迟的选项

            record manual;   # all;
            record_path /usr/local/nginx/html/rec;
            #record_max_size 1024K;
            # append current timestamp to each flv
            record_unique on;
	    #record_suffix -%Y-%m-%d-%H:%M:%S.flv;
	    record_notify on;

            #allow publish 192.168.252.0/24;
            #deny publish all;
            #allow play all;
	    
	    #exec ffmpeg -re -i rtmp://192.168.252.143/$app/$name?$args -vcodec libx264 -vprofile baseline -g 10 -s 300x200 -acodec alac -ar 44100 -ac 1 -f flv rtmp://192.168.252.143/hls/$name?$args 2>>/var/log/ffmpeg-$name.log;
	    exec_push ffmpeg -re -i rtmp://192.168.252.143/$app/$name?$args -vcodec flv -acodec copy  -f flv rtmp://192.168.252.143/hls/$name?$args;
        }

	application vod {
	    play /usr/local/nginx/vod; #视频文件存放位置。
	    #exec ffmpeg -re -i rtmp://192.168.252.143/$app/$name?$args -vcodec flv -acodec copy -f flv rtmp://192.168.252.143/hls/$name?$args;
	}
	application rec {
	    play /usr/local/nginx/html/rec; #录像文件存放位置。
	}

	# ffmpeg是通过libxml2去解析mpd文件的, 所以在configure之前需要先安装libxml2
	# sudo apt-get install libxml2
	# sudo apt-get install libxml2-dev
	# 安装之后, 在configure的时候, 加上--enable-libxml2,在configure完成之后,查看config.h文件, 检查CONFIG_DASH_DEMUXER宏是否为1.
	# 然后make & make install.   最后用编译完成的ffplay测试下上面搭建的dash直播
	# ffplay -i http://192.168.252.143/dash/abcd.mpd 
	application dash {
            live on;
            dash on;
            dash_path /tmp/dash;
	    gop_cache on; ##开启GOP（Group of Picture）缓存，这个是减少播放延迟的选项
	    #MPEG-DASH（HTTP动态自适应流媒体）
	    #dash
	    #dash_path
	    #dash_fragment
	    #dash_playlist_length
	    #dash_nested
	    #dash_cleanup
       }
    }

    #server {
    #    listen 1935;
    #    server_name *.test.com; #用于虚拟主机名前缀通配

    #    application myapp {
    #        live on;
    #        gop_cache on; #打开GOP缓存，减少首屏等待时间
    #    }
    #}

    #server {
    #    listen 1935;
    #    server_name www.test.com; #用于虚拟主机名完全匹配

    #    application myapp {
    #        live on;
    #        gop_cache on; #打开GOP缓存，减少首屏等待时间
    #    }
    #}
}




    



