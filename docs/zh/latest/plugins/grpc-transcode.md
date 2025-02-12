---
title: grpc-transcode
---

<!--
#
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
-->

## 描述

HTTP(s) -> APISIX -> gRPC server

## Proto

### 参数

* `content`: `.proto` 或 `.pb` 文件的内容

### 添加proto

路径中最后的数字，会被用作 proto 的 id 做唯一标识，比如下面示例的 proto `id` 是 `1` ：

```shell
curl http://127.0.0.1:9080/apisix/admin/proto/1 -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "content" : "syntax = \"proto3\";
    package helloworld;
    service Greeter {
        rpc SayHello (HelloRequest) returns (HelloReply) {}
    }
    message HelloRequest {
        string name = 1;
    }
    message HelloReply {
        string message = 1;
    }"
}'
```

如果你的 `.proto` 文件包含 import，或者你想把多个 `.proto` 文件合并成一个 proto。
你可以使用 `.pb` 文件来创建 proto。

假设我们有一个 `.proto` 叫 `proto/helloworld.proto`，它导入了另一个 proto 文件：

```proto
syntax = "proto3";

package helloworld;
import "proto/import.proto";
...
```

首先，让我们从 `.proto`文件创建一个`.pb`文件。

```shell
protoc --include_imports --descriptor_set_out=proto.pb proto/helloworld.proto
```

输出的二进制文件 `proto.pb` 将同时包含 `helloworld.proto` 和 `import.proto`。

然后我们可以将 `proto.pb` 的内容作为 proto 的 `content` 字段提交。

由于内容是二进制的，我们需要先对其进行 base64 编码。这里我们用一个 Python 脚本来做。

```python
#!/usr/bin/env python
# coding: utf-8
# save this file as upload_pb.py
import base64
import sys
# sudo pip install requests
import requests

if len(sys.argv) <= 1:
    print("bad argument")
    sys.exit(1)
with open(sys.argv[1], 'rb') as f:
    content = base64.b64encode(f.read())
id = sys.argv[2]
api_key = "edd1c9f034335f136f87ad84b625c8f1" # Change it

reqParam = {
    "content": content,
}
resp = requests.put("http://127.0.0.1:9080/apisix/admin/proto/" + id, json=reqParam, headers={
    "X-API-KEY": api_key,
})
print(resp.status_code)
print(resp.text)
```

创建proto：

```bash
chmod +x ./upload_pb.pb
./upload_pb.py proto.pb 1
# 200
# {"node":{"value":{"create_time":1643879753,"update_time":1643883085,"content":"CmgKEnByb3RvL2ltcG9ydC5wcm90bxIDcGtnIhoKBFVzZXISEgoEbmFtZRgBIAEoCVIEbmFtZSIeCghSZXNwb25zZRISCgRib2R5GAEgASgJUgRib2R5QglaBy4vcHJvdG9iBnByb3RvMwq9AQoPcHJvdG8vc3JjLnByb3RvEgpoZWxsb3dvcmxkGhJwcm90by9pbXBvcnQucHJvdG8iPAoHUmVxdWVzdBIdCgR1c2VyGAEgASgLMgkucGtnLlVzZXJSBHVzZXISEgoEYm9keRgCIAEoCVIEYm9keTI5CgpUZXN0SW1wb3J0EisKA1J1bhITLmhlbGxvd29ybGQuUmVxdWVzdBoNLnBrZy5SZXNwb25zZSIAQglaBy4vcHJvdG9iBnByb3RvMw=="},"key":"\/apisix\/proto\/1"},"action":"set"}
```

## 参数列表

| 名称      | 类型                                                                       | 必选项 | 默认值 | 有效值 | 描述                       |
| --------- | -------------------------------------------------------------------------- | ------ | ------ | ------ | -------------------------- |
| proto_id  | string/integer                                                             | 必须   |        |        | `.proto` 内容的 id         |
| service   | string                                                                     | 必须   |        |        | grpc 服务名                |
| method    | string                                                                     | 必须   |        |        | grpc 服务中要调用的方法名  |
| deadline  | number                                                                     | 可选   | 0      |        | grpc deadline, ms          |
| pb_option | array[string([pb_option_def](#使用-grpc-transcode-插件的-pb_option-选项))] | 可选   |        |        | proto 编码过程中的转换选项 |

## 示例

### 使用 grpc-transcode 插件

在指定 route 中，代理 grpc 服务接口:

* 注意： 这个 route 对应的 upstream 的属性 `scheme` 必须设置为 `grpc`
* 代理 grpc 服务例子可参考：[grpc_server_example](https://github.com/api7/grpc_server_example)

```shell
curl http://127.0.0.1:9080/apisix/admin/routes/111 -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "methods": ["GET"],
    "uri": "/grpctest",
    "plugins": {
        "grpc-transcode": {
         "proto_id": "1",
         "service": "helloworld.Greeter",
         "method": "SayHello"
        }
    },
    "upstream": {
        "scheme": "grpc",
        "type": "roundrobin",
        "nodes": {
            "127.0.0.1:50051": 1
        }
    }
}'
```

### 测试

访问上面配置的 route：

```shell
$ curl -i http://127.0.0.1:9080/grpctest?name=world
HTTP/1.1 200 OK
Date: Fri, 16 Aug 2019 11:55:36 GMT
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive
Server: APISIX web server
Proxy-Connection: keep-alive

{"message":"Hello world"}
```

这表示已成功代理。

## 使用 grpc-transcode 插件的 pb_option 选项

在指定 route 中，代理 grpc 服务接口:

### 选项清单

* 枚举类型
    * enum_as_name
    * enum_as_value

* 64位整型
    * int64_as_number
    * int64_as_string
    * int64_as_hexstring

* 使用默认值
    * auto_default_values
    * no_default_values
    * use_default_values
    * use_default_metatable

* Hooks开关
    * enable_hooks
    * disable_hooks

```shell
curl http://127.0.0.1:9080/apisix/admin/routes/23 -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "methods": ["GET"],
    "uri": "/zeebe/WorkflowInstanceCreate",
    "plugins": {
        "grpc-transcode": {
            "proto_id": "1",
            "service": "gateway_protocol.Gateway",
            "method": "CreateWorkflowInstance",
            "pb_option":["int64_as_string"]
        }
    },
    "upstream": {
        "scheme": "grpc",
        "type": "roundrobin",
        "nodes": {
            "127.0.0.1:26500": 1
        }
    }
}'
```

### 测试 pb_option 参数

访问上面配置的 route：

```shell
$ curl -i "http://127.0.0.1:9080/zeebe/WorkflowInstanceCreate?bpmnProcessId=order-process&version=1&variables=\{\"orderId\":\"7\",\"ordervalue\":99\}"
HTTP/1.1 200 OK
Date: Wed, 13 Nov 2019 03:38:27 GMT
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive
grpc-encoding: identity
grpc-accept-encoding: gzip
Server: APISIX web server
Trailer: grpc-status
Trailer: grpc-message

{"workflowKey":"#2251799813685260","workflowInstanceKey":"#2251799813688013","bpmnProcessId":"order-process","version":1}
```

`"workflowKey":"#2251799813685260"` 表示已成功。

## 禁用插件

在插件设置页面中删除相应的 json 配置即可禁用 `grpc-transcode` 插件。APISIX 的插件是热加载的，因此无需重启 APISIX 服务。

```shell
curl http://127.0.0.1:9080/apisix/admin/routes/111 -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "uri": "/grpctest",
    "plugins": {},
    "upstream": {
        "scheme": "grpc",
        "type": "roundrobin",
        "nodes": {
            "127.0.0.1:50051": 1
        }
    }
}'
```
