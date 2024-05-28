#### harbor后悔药之修改用户名密码

* 背景
  
  初次使用harbor，不熟悉harbor配置文件harbor.yml，导致配置文件里出现了弱密码，事后需要修改。

* 详细说明
  
  harbor.yml.tmpl--harbor配置模板文件里有两处带密码的地方
  harbor_admin_password--如注释说明为harbor管理员密码，安装成功后可以从UI上再次修改
  database password--harbor数据库的密码，准确的说是harbor数据库postgres用户的密码，生产环境使用前需改掉
  
* 两处密码的不同处--关键在于执行prepare脚本然后docker-compose down/up -d
  harbor_admin_password--只在第一次部署安装的时候起作用，安装完此处密码写到数据库里持久化，然后登录UI后再修改，修改后的密码仍然写到数据库持久化，也就是说再执行prepare时，不关配置文件里harbor_admin_password的值是多少，都不再往数据库里写，所以修改此处再执行prepare时无任何影响
  database password--也是第一次部署的时候起作用，安装完此处密码写到数据库持久化，也就是数据库postgres用户的密码，与前者不同的是，如果改动了此处的值再执行prepare，然后docker-compose down/up -d，那么会出现数据库密码连接不正确的错误，导致整个harbor服务不可用
  
  * 后悔药
    如果第一次部署确实忘记了修改此处的root123，那么也是有解决办法的。
    通过上述分析，此处的值就是数据库postgres用户的密码，改密码已经持久化在数据库，所以修改配置文件再重启时就对不上，所以只要保证配置文件里的密码和持久化的postgres用户的密码一致即可，修改方式如下：
    进入harbor-db数据库容器，psql进入数据库，输入\\password修改密码，两次确认
    当然alter修改的方式也可行
  
* 扩展--修改管理员admin密码
  某种特种情况下需要改admin的密码，思路同上，进数据库直接修改admin用户的密码值，问题是数据库存的密码都是md5加密过的，这就需要“撞库”，找一个已知明文密码的md5加密存储值，然后把admin的改成此样
  
* harbor一个特殊的坑
  如果你登录harbor的时候一直提示用户名密码正确，并且排查完成确实用的正确的用户名密码，那么请你回忆下最近有没有重启过harbor，如果重启harbor的时候直接用的docker-compose up -d，而不是down后在up -d，那么恭喜你！碰到了跟我一样的故障，具体原因不详，但是解决方法很简单，如上，先down再up起来就好了~
