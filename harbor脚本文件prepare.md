#### harbor脚本文件prepare

* 背景
  上文“harbor后悔药之修改用户名密码”提到修改两个用户名密码的关键在于执不执行prepare脚本，那么prepare干了啥

* prepare执行日志记录
  
  ```
  root@VM-16-8-ubuntu:/data/harbor# ./prepare 
  + [[ -n '' ]]
  +++ dirname ./prepare
  ++ cd .
  ++ pwd -P
  + harbor_prepare_path=/data/harbor
  + echo 'prepare base dir is set to /data/harbor'
  prepare base dir is set to /data/harbor
  + rm -rf /data/harbor/input
  + mkdir -p /data/harbor/input
  + input_dir=/data/harbor/input
  + [[ ! '' =~ ^-- ]]
  + '[' -f '' ']'
  + '[' -f /data/harbor/harbor.yml ']'
  + cp /data/harbor/harbor.yml /data/harbor/input/harbor.yml
  ++ grep '^[^#]*data_volume:' /data/harbor/input/harbor.yml
  ++ awk '{print $NF}'
  + data_path=/data
  + previous_secretkey_path=/data/secretkey
  + previous_defaultalias_path=/data/defaultalias
  + '[' -f /data/secretkey ']'
  + '[' -f /data/defaultalias ']'
  + secret_dir=/data/secret
  + config_dir=/data/harbor/common/config
  + docker run --rm -v /data/harbor/input:/input -v /data:/data -v /data/harbor:/compose_location -v /data/harbor/common/config:/config -v /:/hostfs --privileged goharbor/prepare:v2.8.4 prepare
  Clearing the configuration file: /config/registry/passwd
  Clearing the configuration file: /config/registry/config.yml
  Clearing the configuration file: /config/portal/nginx.conf
  Clearing the configuration file: /config/jobservice/env
  Clearing the configuration file: /config/jobservice/config.yml
  Clearing the configuration file: /config/registryctl/env
  Clearing the configuration file: /config/registryctl/config.yml
  Clearing the configuration file: /config/core/env
  Clearing the configuration file: /config/core/app.conf
  Clearing the configuration file: /config/db/env
  Clearing the configuration file: /config/nginx/nginx.conf
  Clearing the configuration file: /config/log/logrotate.conf
  Clearing the configuration file: /config/log/rsyslog_docker.conf
  Generated configuration file: /config/portal/nginx.conf
  Generated configuration file: /config/log/logrotate.conf
  Generated configuration file: /config/log/rsyslog_docker.conf
  Generated configuration file: /config/nginx/nginx.conf
  Generated configuration file: /config/core/env
  Generated configuration file: /config/core/app.conf
  Generated configuration file: /config/registry/config.yml
  Generated configuration file: /config/registryctl/env
  Generated configuration file: /config/registryctl/config.yml
  Generated configuration file: /config/db/env
  Generated configuration file: /config/jobservice/env
  Generated configuration file: /config/jobservice/config.yml
  loaded secret from file: /data/secret/keys/secretkey
  Generated configuration file: /compose_location/docker-compose.yml
  + echo 'Clean up the input dir'
  Clean up the input dir
  + rm -rf /data/harbor/input
  root@VM-16-8-ubuntu:/data/harbor# 
  
  root@VM-16-8-ubuntu:/data/harbor# docker images 
  REPOSITORY                      TAG       IMAGE ID       CREATED       SIZE
  goharbor/harbor-exporter        v2.8.4    b8d33e28ec68   2 weeks ago   97.7MB
  goharbor/redis-photon           v2.8.4    7b7324d651ca   2 weeks ago   120MB
  goharbor/trivy-adapter-photon   v2.8.4    91d8e9f0b21a   2 weeks ago   464MB
  goharbor/notary-server-photon   v2.8.4    a46f91560454   2 weeks ago   113MB
  goharbor/notary-signer-photon   v2.8.4    da66bd8d944b   2 weeks ago   110MB
  goharbor/harbor-registryctl     v2.8.4    805b38ca6bee   2 weeks ago   141MB
  goharbor/registry-photon        v2.8.4    756769e94123   2 weeks ago   79MB
  goharbor/nginx-photon           v2.8.4    375018db778b   2 weeks ago   116MB
  goharbor/harbor-log             v2.8.4    8a2045fb24d2   2 weeks ago   124MB
  goharbor/harbor-jobservice      v2.8.4    97808fc10f64   2 weeks ago   141MB
  goharbor/harbor-core            v2.8.4    c26fcd0714d8   2 weeks ago   164MB
  goharbor/harbor-portal          v2.8.4    4a8b0205c0f9   2 weeks ago   124MB
  goharbor/harbor-db              v2.8.4    5b8af16d7420   2 weeks ago   174MB
  goharbor/prepare                v2.8.4    bdbf974d86ce   2 weeks ago   166MB
  ```

* prepare脚本核心
  
  ```
  # Run prepare script
  docker run --rm -v $input_dir:/input \
                      -v $data_path:/data \
                      -v $harbor_prepare_path:/compose_location \
                      -v $config_dir:/config \
                      -v /:/hostfs \
                      --privileged \
                      goharbor/prepare:v2.8.4 prepare $@
  ```
  
  脚本前部分准备了一堆文件目录，其实就是为了将他们挂载进容器然后执行prepare脚本，详情如下
  
  ```
  + docker run --rm \
  -v /data/harbor/input:/input \
  -v /data:/data \
  -v /data/harbor:/compose_location \
  -v /data/harbor/common/config:/config \
  -v /:/hostfs \
  --privileged goharbor/prepare:v2.8.4 prepare
  ```
  
  核心挂载就是将common下的config挂载进容器然后执行prepare，prepare看日志就是清理原来的配置然后生成新的配置，这些配置当然包括数据库连接密码等核心配置，原来就在common下的config里，仔细观察里面都是明文保存
  
  ```
  root@VM-16-8-ubuntu:/data/harbor/common/config# tree -L 2
  .
  ├── core
  │   ├── app.conf
  │   ├── certificates
  │   └── env
  ├── db
  │   └── env
  ├── jobservice
  │   ├── config.yml
  │   └── env
  ├── log
  │   ├── logrotate.conf
  │   └── rsyslog_docker.conf
  ├── nginx
  │   ├── conf.d
  │   └── nginx.conf
  ├── portal
  │   └── nginx.conf
  ├── registry
  │   ├── config.yml
  │   └── passwd
  ├── registryctl
  │   ├── config.yml
  │   └── env
  └── shared
      └── trust-certificates
  
  12 directories, 13 files
  ```

* 总结
  harbor服务有多个组件一块组成，如下：
  
  ```
  root@VM-16-8-ubuntu:/data/harbor# docker-compose -f /data/harbor/docker-compose.yml ps 
  NAME                IMAGE                                COMMAND                  SERVICE             CREATED             STATUS                    PORTS
  harbor-core         goharbor/harbor-core:v2.8.4          "/harbor/entrypoint.…"   core                49 minutes ago      Up 49 minutes (healthy)   
  harbor-db           goharbor/harbor-db:v2.8.4            "/docker-entrypoint.…"   postgresql          49 minutes ago      Up 49 minutes (healthy)   
  harbor-jobservice   goharbor/harbor-jobservice:v2.8.4    "/harbor/entrypoint.…"   jobservice          49 minutes ago      Up 49 minutes (healthy)   
  harbor-log          goharbor/harbor-log:v2.8.4           "/bin/sh -c /usr/loc…"   log                 49 minutes ago      Up 49 minutes (healthy)   127.0.0.1:1514->10514/tcp
  harbor-portal       goharbor/harbor-portal:v2.8.4        "nginx -g 'daemon of…"   portal              49 minutes ago      Up 49 minutes (healthy)   
  nginx               goharbor/nginx-photon:v2.8.4         "nginx -g 'daemon of…"   proxy               49 minutes ago      Up 49 minutes (healthy)   0.0.0.0:80->8080/tcp, :::80->8080/tcp, 0.0.0.0:443->8443/tcp, :::443->8443/tcp
  redis               goharbor/redis-photon:v2.8.4         "redis-server /etc/r…"   redis               49 minutes ago      Up 49 minutes (healthy)   
  registry            goharbor/registry-photon:v2.8.4      "/home/harbor/entryp…"   registry            49 minutes ago      Up 49 minutes (healthy)   
  registryctl         goharbor/harbor-registryctl:v2.8.4   "/home/harbor/start.…"   registryctl         49 minutes ago      Up 49 minutes (healthy)   
  ```
  
  如果在这里把数据库连接的密码给改了，那么harbor-core等其他需要连接数据库的harbor组件必定连不上数据库，导致服务不可用，这就是两者为什么都能改而执行prepare结果不同的核心原因
  harbor admin 的属于前台，已经持久化在数据库了，配置修改不会再做持久化（harbor db 的密码也不会再做持久化，但是这个密码会有别的组件引用），所以修改配置文件中密码值不影响原密码登录

* prepare脚本鉴赏
  
  ```shell
  #!/bin/bash
  set -e
  
  # If compiling source code this dir is harbor's make dir.
  # If installing harbor via package, this dir is harbor's root dir.
  if [[ -n "$HARBOR_BUNDLE_DIR" ]]; then
      harbor_prepare_path=$HARBOR_BUNDLE_DIR
  else
      # 获取脚本当前路径
      harbor_prepare_path="$( cd "$(dirname "$0")" ; pwd -P )"
  fi
  echo "prepare base dir is set to ${harbor_prepare_path}"
  
  # Clean up input dir
  rm -rf ${harbor_prepare_path}/input
  # Create a input dirs
  mkdir -p ${harbor_prepare_path}/input
  input_dir=${harbor_prepare_path}/input
  
  # Copy harbor.yml to input dir
  # 双中括号 正则匹配字符串比较
  if [[ ! "$1" =~ ^\-\- ]] && [ -f "$1" ]
  then
      cp $1 $input_dir/harbor.yml
      shift
  else
      if [ -f "${harbor_prepare_path}/harbor.yml" ];then
          cp ${harbor_prepare_path}/harbor.yml $input_dir/harbor.yml
      else
          echo "no config file: ${harbor_prepare_path}/harbor.yml"
          exit 1
      fi
  fi
  
  # awk '{print $NF}' NF这里表示最后一列 NR表示行
  data_path=$(grep '^[^#]*data_volume:' $input_dir/harbor.yml | awk '{print $NF}')
  
  # If previous secretkeys exist, move it to new location
  previous_secretkey_path=/data/secretkey
  previous_defaultalias_path=/data/defaultalias
  
  if [ -f $previous_secretkey_path ]; then
      mkdir -p $data_path/secret/keys
      mv $previous_secretkey_path $data_path/secret/keys
  fi
  if [ -f $previous_defaultalias_path ]; then
      mkdir -p $data_path/secret/keys
      mv $previous_defaultalias_path $data_path/secret/keys
  fi
  
  
  # Create secret dir
  secret_dir=${data_path}/secret
  config_dir=$harbor_prepare_path/common/config
  
  # Run prepare script
  docker run --rm -v $input_dir:/input \
                      -v $data_path:/data \
                      -v $harbor_prepare_path:/compose_location \
                      -v $config_dir:/config \
                      -v /:/hostfs \
                      --privileged \
                      goharbor/prepare:v2.8.4 prepare $@ 
                      # 所有位置参数 默认为空 可以加点别的试试
  
  echo "Clean up the input dir"
  # Clean up input dir
  rm -rf ${harbor_prepare_path}/input
  
  ```
  
  
