```bash
set key value [EX seconds] [PX milliseconds] [NX|XX]
EX seconds：设置失效时长，单位秒
PX milliseconds：设置失效时长，单位毫秒
NX：key不存在时设置value，成功返回OK，失败返回(nil)
XX：key存在时设置value，成功返回OK，失败返回(nil)
案例：设置name=liuxinglin，失效时长100s，不存在时设置
1.1.1.1:6379> set name liuxinglin ex 100 nx
OK
1.1.1.1:6379> get name
"liuxinglin"
1.1.1.1:6379> ttl name
(integer) 94
```

 