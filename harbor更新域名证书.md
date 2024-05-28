#### harbor更新域名证书


Docker-compose restart失败，但是down后再up –d就能起来

1、 重新生成证书cert 和 key

2、 修改harbor配置文件 ssl配置相关

3、 若前端有nginx 也要修改nginx配置文件ssl相关

4、 ./prepare重新配置

5、 若有需要 docker-compose重启容器    


