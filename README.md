### 使用方式


> 重要提示：示例环境使用的是阿里云 Kubernetes 服务，如使用其他 K8S 环境会有些差异，可自行修改 yaml 文件相应的 storageClassName 和 Service  type 等

**1. 创建单独的 namespace**
```
# kubectl apply -f 00-ns.yaml
```

**2. 生成开启 x-pack 时的 ssl 证书**
```
// 参考：https://github.com/elastic/helm-charts/blob/master/elasticsearch/examples/security/Makefile#L24-L35
# docker run --name es-certutil -i -w /tmp docker.elastic.co/elasticsearch/elasticsearch:7.14.0 /bin/sh -c  \
    "elasticsearch-certutil ca --out /tmp/es-ca.p12 --pass '' && \
    elasticsearch-certutil cert --name security-master --dns \
    security-master --ca /tmp/es-ca.p12 --pass '' --ca-pass '' --out /tmp/elastic-certificates.p12"
# docker cp es-certutil:/tmp/elastic-certificates.p12 ./
# kubectl -n ns-elk create secret generic elastic-certificates --from-file=./elastic-certificates.p12
```


**3. 部署 elasticsearch master 节点**
```
# kubectl -n ns-elk apply -f 01-es-master.yaml
```

**4. 部署 elasticsearch data 节点**
```
# kubectl -n ns-elk apply -f 02-es-data.yaml
```

**5. 部署 elasticsearch client/ingest 节点**
```
# kubectl -n ns-elk apply -f 03-es-client.yaml
```

**6. 暴露 elasticsearch service**
```
# kubectl -n ns-elk apply -f 04-es-service.yaml
```

**7. 设置 elasticsearch 的密码**
```
// 记得保存这些初始密码
# kubectl -n ns-elk exec -it $(kubectl -n ns-elk get pods | grep elasticsearch-client | sed -n 1p | awk '{print $1}') -- bin/elasticsearch-setup-passwords auto -b
Changed password for user apm_system
PASSWORD apm_system = MxlMXbMah7x54c4YQjPj

Changed password for user kibana_system
PASSWORD kibana_system = dgPCuR2ayG9FPCYLHlav

Changed password for user kibana
PASSWORD kibana = dgPCuR2ayG9FPCYLHlav

Changed password for user logstash_system
PASSWORD logstash_system = KgynQ5D3pD3OXDmV5IMA

Changed password for user beats_system
PASSWORD beats_system = ZMTRWXeVkEsrKU3BPl27

Changed password for user remote_monitoring_user
PASSWORD remote_monitoring_user = Vp5WI34HRO8XfRgAzBrC

Changed password for user elastic
PASSWORD elastic = 03sWFWzGOjNOCioqcbV3
```

**8. 配置 kibana.yaml 连接 elasticsearch 的 secret**
```
// 使用上面生成的用户 elastic 的密码 03sWFWzGOjNOCioqcbV3（后续访问 elasticsearch 或者 登录 kibana 都是用这个用户和密码）
# kubectl -n ns-elk create secret generic elasticsearch-password --from-literal password=03sWFWzGOjNOCioqcbV3 
```

**9. 部署 kibana**
```
# kubectl -n ns-elk apply -f 05-kibana.yaml
```

**10. 部署 zookeeper**
```
# helm -n ns-elk install zookeeper -f 06-zk.yaml ./zookeeper
```

**11. 部署 kafka**
```
# helm -n ns-elk install kafka -f 07-kafka.yaml ./kafka
```

**12. 部署 filebeat**
```
# kubectl -n ns-elk apply -f 08-filebeat.yaml
```

**13. 部署 logstach**
```
# kubectl -n ns-elk apply -f 09-logstach.yaml
```

**14. 其他**

如果拉取不了 docker.elastic.co 的镜像，可以使用 https://hub.docker.com/u/awker 下的相关镜像
