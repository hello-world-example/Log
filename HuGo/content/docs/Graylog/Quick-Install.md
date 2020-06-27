# 快速安装



## Docker 方式安装 (2.3.1)

- mongo:3.6.18
- elasticsearch:5.6.12
- graylog/graylog:2.3.1-2

```bash
# mongo:3.6.18
$ docker run --name mongo -d mongo:3.6.18

# elasticsearch:5.6.12
$ docker run --name elasticsearch \
    -e "http.host=0.0.0.0" \
    -e "xpack.security.enabled=false" \
    -p 19200:9200 \
    -d elasticsearch:5.6.12

# graylog/graylog:2.3.1-2
$ docker run --link mongo --link elasticsearch \
    -p 9000:9000 -p 12201:12201 -p 514:514 -p 55555:55555/udp \
    -e GRAYLOG_WEB_ENDPOINT_URI="http://127.0.0.1:9000/api" \
    --name graylog -d graylog/graylog:2.3.1-2
```

验证

- 访问： http://localhost:9000/
- 默认账户： admin / admin
- 创建 Input
  - 进入： http://localhost:9000/system/inputs
  - 选择 `GELF UDP` > `Launch new input`
  - 勾选 `Global` 
  - Port `55555`
  - `Save` 保存
- 写入日志：`echo -n '{ "version": "1.1", "host": "example.org", "short_message": "A short message", "level": 5, "_some_info": "foo" }' | nc -w0 -u localhost 55555`



## Read More

- [Installing Graylog » Docker](https://docs.graylog.org/en/latest/pages/installation/docker.html)