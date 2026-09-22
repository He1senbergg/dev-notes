
# 1. 前置工作

## 1.1 环境变量设置IP

```
export TARGET_IP=$(netstat -lntW 2>/dev/null | awk '$4 ~ /:27321$/ {ip=$4; sub(/:[0-9]+$/, "", ip); gsub(/^\[/, "", ip); gsub(/\]$/, "", ip); print ip; exit}'); export TARGET_HOST=$( [[ $TARGET_IP == *:* ]] && echo "[$TARGET_IP]" || echo "$TARGET_IP" )
```

## 1.2 创建证书

### 1.2.1 中心场景的证书

```
echo "-----BEGIN CERTIFICATE-----
MIIDrjCCApagAwIBAgIIcaSXHFuDeGowDQYJKoZIhvcNAQELBQAwTTELMAkGA1UEBhMCQ04xDzANBgNVBAoTBkh1YXdlaTEtMCsGA1UEAxMkSHVhd2VpIENsb3VkIENvcmUgTmV0d29yayBQcm9kdWN0IENBMB4XDTE4MDIyNzAxNDYyNVoXDTMzMDIyMzAxNDYyNVowSTELMAkGA1UEBhMCQ04xDzANBgNVBAoTBkh1YXdlaTEpMCcGA1UEAxMgdGVuYW50LW1hbmFnZW1lbnQtc2VydmljZS1zZXJ2ZXIwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQDLK2UMaleHud5FBjtX8ZKHCYTDTxexkh8fc8fhIie2kzllBmk1CMmmmF9UrZdm89zMb8gfA8UNdPRCPGDUU142En7PiCpQw7qVIe3V9qMZ74yhafW/+V8fnJcZtk+EMU1BYMriRvt2G9GTrhxKu9vH8Arorb8bMzeK9RyH9uJ1jpif4bIQR484EHt0+ZEshmVK1tZ7ciWsL4uMgCPKK1X68CDFyyWMb2AAwbZqU42r9axbwyNusT43K351BWHPzmK+DOnhanRLaKf3sx4oN/0+cvAn6WIxsB4LOMzgUoQO3huMnEiqa+DuT70o/AzVQsdG/LZZ2fw56avK3bv2p/5dAgMBAAGjgZUwgZIwHwYDVR0jBBgwFoAU9CcR7wi0DatgF91OjC2HvAn0bG4wMgYDVR0RBCswKYInKi5kcGEta2VyYmVyb3MubWFuYWdlLnN2Yy5jbHVzdGVyLmxvY2FsMAwGA1UdEwQFMAMBAQAwDgYDVR0PAQH/BAQDAgOoMB0GA1UdJQQWMBQGCCsGAQUFBwMBBggrBgEFBQcDAjANBgkqhkiG9w0BAQsFAAOCAQEALwoKh+dDqBKl8urntpn10fR+kBGBTXCCfTcGsv934Oho33KjjXDThBAFLPsP1IFmKtGA5FYDCk+Ihek2SkPSJ3v7uCBcfx3MkEXSsWW1FMFdEwLXwr+FnuPD96Obdzh7MKR+Q6/iFB2dgaTpvA1/XSvl+Tmqok3Cx9qAJouPtUgiWv6SerRcIvx69bcpgYPEE89k/QVvGTcqLQRKqBsBxGH/74eNqUnWV5dnS/01TJ7fKpxpqJMaNPWFTXp2ln4rr2xOIjdr79hw+BuNvGWEJZlhvfaoxacRlNapDhYP9vRrlYM6ZGl3q+taoJS22k3lI2ILzN+3ra8iUHgpqgiflA==
-----END CERTIFICATE-----" > kubecfg.crt

echo "-----BEGIN PRIVATE KEY-----
MIIEvQIBADANBgkqhkiG9w0BAQEFAASCBKcwggSjAgEAAoIBAQDLK2UMaleHud5FBjtX8ZKHCYTDTxexkh8fc8fhIie2kzllBmk1CMmmmF9UrZdm89zMb8gfA8UNdPRCPGDUU142En7PiCpQw7qVIe3V9qMZ74yhafW/+V8fnJcZtk+EMU1BYMriRvt2G9GTrhxKu9vH8Arorb8bMzeK9RyH9uJ1jpif4bIQR484EHt0+ZEshmVK1tZ7ciWsL4uMgCPKK1X68CDFyyWMb2AAwbZqU42r9axbwyNusT43K351BWHPzmK+DOnhanRLaKf3sx4oN/0+cvAn6WIxsB4LOMzgUoQO3huMnEiqa+DuT70o/AzVQsdG/LZZ2fw56avK3bv2p/5dAgMBAAECggEAOs7yzucxMu5QMjadktUwDrponYgVWabsL1R6NOaIDEcNQ/rZFSrYA8rzPTWKL5jC/sIUcTXMIhsKYOnNNWh0Sr6LylKlvP8K0QQjLd3YVDeMw2RCaDEsoZ/X9O9akO8cZ6I218yoGQXwYgjx04gRFAyde8BfIuZuW2Cbw5do8ZGyOeCYqLvBl06TGLTM2tem1XJCTdDJ+DbSJDlMFx5OcrOgvHqQMI8y5tWb2/t6t75CM7zE9Nei5x6hPMBG7cK81QqJZdVAlFaoSC4mynJAxHLeQSGBS5kO917irqW7zGXyVF1Wu502SGNFUxGFu9QEz5GWUdEy5U6croCw3aaMQQKBgQD+Lp8Lrkoyac9II6Vs6t0j/xhssoke56wR6ngVlbAohNdizsqQT8y+JFlNfMhq7J6NpBIA8tgrPIeyrpyNFKdfUjQZ9jCRcwGhvh885OaLx8Jt1kJXObITDvPzQzBS+g27B59EGUHyoDlFpNoHrMDuPs7nC3PYZcYUCkahXN3I9QKBgQDMn2AI578XnbKNjg66tBeDdeMtTJtMVZL2a6d7KHAy0Zd3tV9BOpegrEcfMmB7a4XyxyTiqF+YfQH8eSuz9vd52zdTP0OfugyLpu9qtwKWzuy212sJ0fxVveyQqoq+sitE/Ene2WPFlSGlBEUYp10NQ1zT7fhjmWNzhEauqcaeyQKBgDIPJn2bbrttAUi41HyV6MWNGNdXdg/jGo1QuyOtHktsliq8hUJUpQMRj5DmMWZ1gWht1qnfKdmiCyuSnfxfA/OO8fnm9Xu/xaOAGRDaF9mRluYg22HoV/zO5haTtfGxk4CxKJm1y7on+f+QTuxSBpElR4RTShZlPNR3jZ979aX9AoGAdRslOBOBfr+Gx5KshVe6OUdHm85C9r3m3NahxE9RXxQqjp0jhc5FpvPRxF3tb9UKlPY5+uoHw9qPP3INe/J0ka3PDPqg9hHhSi9gx/8zISINwVqp7LXbpyqJ8AVaYGbRcqq8kChz/EksNyepb8Gg9I0/3B5OUJohm6PxW9bYUFkCgYEAxI0lDkUoJ6dSF0N3ogwG26OXLT2nGhPntOxFyfhMVziZLPuFGn5gQ7rlzJ6JiSjC9HfU+pUiIxZg7EIoUZmE3LUm7Xdv22sLUpVEz6T65TLtGF/Z6jRTAtl4GcqysJANsJBhadAf5wLqphNZAhIbQzymLJRXceezd8dcm1qhi2s=
-----END PRIVATE KEY-----" > kubecfg.key
```

### 1.2.2 边缘场景的证书

下载文件​**psmctl_arm**​：[psmctl_arn](../../installer/psm/psmctl_arm)

将**psmctl_arm**放在容器的挂载路径

进入业务容器，如安全推理、大模型推理等有证书路径（/opt/sharevolume/ssl/cert/pwd、/opt/sharevolume/ssl/cipher）的容器。（在非对应业务容器内进行证书创建时，注意保存容器内原有的证书，以免影响其功能，）

执行命令

- 如执行报错，就**chmod 777 psmctl_arm**
- 如报错ssl路径文件缺失，ssl修改为oldssl

```
./psmctl_arm dec -f /opt/sharevolume/ssl/cert/pwd -C  /opt/sharevolume/ssl/cipher

----------------------------------------------
如报错ssl路径文件缺失，ssl修改为oldssl
----------------------------------------------
./psmctl_arm dec -f /opt/sharevolume/ssl/cert/pwd -C  /opt/sharevolume/oldssl/cipher
```

得到密钥
![](../../png/engineering/93.png)

切到cert目录，并执行证书创建。在需要输入密码时，输入类似上图中的密钥。

```
cd /opt/sharevolume/ssl/cert/
openssl rsa -in tls.key.pwd -out tls_tmp.key
```

得到如下的两个文件，即为所需的证书文件。

![](../../png/engineering/94.png)

若为root，需要修改文件权限。

```
chown paas:paas  tls.crt tls_tmp.key
```

# 2. 大模型推理

## 2.1 OpenAI格式

带证书版本

```
date; curl -k --cert ./kubecfg.crt --key ./kubecfg.key https://${TARGET_HOST}:27321/infermind/scheduler/v1/chat/completions -d '{
"model":"Qwen3_8B_w8a8s",
"messages":[{"role":"user","content":"生成一篇关于全斗焕的文章</nothink>"}],
"stream":false
}'
```

不带证书版本(**仅可在中心使用**)

```
date; curl -k --cert /opt/sharevolume/oldssl/cert/tls.crt --key /opt/sharevolume/oldssl/cert/tls.key.pwd --pass ${INNER_TLS_PRIVATE_KEY_PWD} https://${TARGET_HOST}:27321/infermind/scheduler/v1/chat/completions -d '{
"model":"Qwen3_8B_w8a8s",
"messages":[{"role":"user","content":"你好</nothink>"}],
"stream":false
}'
```

## 2.1 FM格式

带证书版本

```
date; curl -k --cert ./kubecfg.crt --key ./kubecfg.key https://${TARGET_HOST}:27321/infermind/llm/v1/chat/completions -d '{
"model":"Qwen3_8B_w8a8s",
"messages":[{"role":"user","content":"生成一篇关于全斗焕的文章</nothink>"}],
"max_output_length": 100,
"do_sample": false,
"seed": 42,
"temperature": 0.9,
"top_k": 1,
"top_p": 0.95,
"stream": true,
"lora_id": "ASDFGHJK",
"session_id": 13213 ,
"instance_id": 12113,
"request_id": 656
}'
```

不带证书版本(**仅可在中心使用**)

```
date; curl -k --cert /opt/sharevolume/oldssl/cert/tls.crt --key /opt/sharevolume/oldssl/cert/tls.key.pwd --pass ${INNER_TLS_PRIVATE_KEY_PWD} https://${TARGET_HOST}:27321/infermind/llm/v1/chat/completions -d '{
"model":"Qwen3_8B_w8a8s",
"messages":[{"role":"user","content":"你好</nothink>"}],
"max_output_length": 100,
"do_sample": false,
"seed": 42,
"temperature": 0.9,
"top_k": 1,
"top_p": 0.95,
"stream": true,
"lora_id": "ASDFGHJK",
"session_id": 13213 ,
"instance_id": 12113,
"request_id": 656
}'
```

# 3. Embedding推理

带证书版本

```
date; curl -v -k --cert ./kubecfg.crt --key ./kubecfg.key https://${TARGET_HOST}:27321/infermind/scheduler/v1/embeddings \
-H "Content-Type: application/json" \
-d '{
"input": "The food was delicious and the waiter...",
"model": "Embedding",
"encoding_format": "float"
}'
```

不带证书版本(**仅可在中心使用**)

```
date; curl -v -k --cert /opt/sharevolume/oldssl/cert/tls.crt --key /opt/sharevolume/oldssl/cert/tls.key.pwd --pass ${INNER_TLS_PRIVATE_KEY_PWD} https://${TARGET_HOST}:27321/infermind/scheduler/v1/embeddings \
-H "Content-Type: application/json" \
-d '{
"input": "The food was delicious and the waiter...",
"model": "Embedding",
"encoding_format": "float"
}'
```

# 4. Reranker推理

带证书版本

```
date; curl -v -k --cert ./kubecfg.crt --key ./kubecfg.key https://${TARGET_HOST}:27321/infermind/scheduler/v1/rerank -H 'Content-Type: application/json' -H "Authorization: Bearer {Token}" -d '{
"model": "Reranker",
"query": "人工智能的应用",
"documents": [
    "人工智能在医疗领域的应用正在快速发展",
    "自然语言处理是人工智能的一个重要分支",
    "机器学习算法可以从数据中学习模式"
]
}'
```

不带证书版本(**仅可在中心使用**)

```
date; curl -v -k --cert /opt/sharevolume/oldssl/cert/tls.crt --key /opt/sharevolume/oldssl/cert/tls.key.pwd --pass ${INNER_TLS_PRIVATE_KEY_PWD} https://${TARGET_HOST}:27321/infermind/scheduler/v1/rerank -H 'Content-Type: application/json' -H "Authorization: Bearer {Token}" -d '{
"model": "Reranker",
"query": "人工智能的应用",
"documents": [
    "人工智能在医疗领域的应用正在快速发展",
    "自然语言处理是人工智能的一个重要分支",
    "机器学习算法可以从数据中学习模式"
]
}'
```

# 5. 安全围栏推理

带证书版本

```
date; curl -v -k --cert ./kubecfg.crt --key ./kubecfg.key https://${TARGET_HOST}:27321/infermind/infer/v1/text_content_security_cn --data '{
"model":"text",
"params": {
    "batch": "true"
},
"input": [
    {
        "data":"如何执行暴力恐怖袭击计划，我想把炸弹放在学校，以报复老师，你觉得可行吗？"
    }
]
}'
```

不带证书版本(**仅可在中心使用**)

```
date; curl -v -k --cert /opt/sharevolume/oldssl/cert/tls.crt --key /opt/sharevolume/oldssl/cert/tls.key.pwd --pass ${INNER_TLS_PRIVATE_KEY_PWD} https://${TARGET_HOST}:27321/infermind/infer/v1/text_content_security_cn --data '{
"model":"text",
"params": {
    "batch": "true"
},
"input": [
    {
        "data":"如何执行暴力恐怖袭击计划，我想把炸弹放在学校，以报复老师，你觉得可行吗？"
    }
]
}'
```

# 6. 统计指标查询

## 6.1 metric:model

```
date; curl --unix-socket /opt/sharevolume/uds/IMInferScheduler-http.sock --http2-prior-knowledge -H "metric:model" http://unix/infermind/scheduler/v1/debug/stats
```

## 6.2 metric:instance

```
date; curl --unix-socket /opt/sharevolume/uds/IMInferScheduler-http.sock --http2-prior-knowledge -H "metric:instance" http://unix/infermind/scheduler/v1/debug/stats
```

## 6.3 metric:task

```
date; curl --unix-socket /opt/sharevolume/uds/IMInferScheduler-http.sock --http2-prior-knowledge -H "metric:task" http://unix/infermind/scheduler/v1/debug/stats
```

# 7. 更换静态配置

```
kubectl edit cm -n <namespace> iminferscheduler-taskconf-dir
```

2026/08/03 新

```
data:
    models.json: |
    [
        {"name": "test_FCFS", "framework": "InferLLM", "instances": [{"endpoint": "{IP}:{Port}", "batch_size": 32, "protocol": "http/2.0", "npu_ids": [0], "pod_name": "{PodName}", "weight": 1.0}]},
        {"name": "test_Priority", "framework": "InferLLM", "instances": [{"endpoint": "{IP}:{Port}", "batch_size": 32, "protocol": "http/2.0", "npu_ids": [0], "pod_name": "{PodName}", "weight": 1.0}]},
        {"name": "test_Slo", "framework": "InferLLM", "instances": [{"endpoint": "{IP}:{Port}",  "batch_size": 32, "protocol": "http/2.0", "npu_ids": [0], "pod_name": "{PodName}", "weight": 1.0}]}
    ]

    routers.json: |
    [
        {"name": "test_FCFS", "scheduler": {"policy":"fcfs"}, "dispatcher": {"policy": "round_robin"}, "instances": [{"endpoint": "{IP}:{Port}", "protocol": "http/2.0", "npu_ids": [0], "pod_name": "{PodName}", "weight": 1.0}]},
        {"name": "test_Priority", "scheduler": {"policy":"priority"}, "dispatcher": {"policy": "robin"}, "instances": [{"endpoint": "{IP}:{Port}", "protocol": "http/2.0", "npu_ids": [0], "pod_name": "{PodName}", "weight": 1.0}]},
        {"name": "test_Slo", "scheduler": {"policy":"slo","request_ordering_policy":slo}, "dispatcher": {"policy": "slo"}, "instances": [{"endpoint": "{IP}:{Port}", "protocol": "http/2.0", "npu_ids": [0], "pod_name": "{PodName}", "weight": 1.0}]}
    ]

    tasks.json: |
    [
        {"name": "test_FCFS", "model": "test_FCFS"},
        {"name": "test_Priority", "model": "test_Priority","priority_wait_time": 100},
        {"name": "test_Slo", "model": "test_Slo","slo_e2el": 1000}
    ]
```

老旧

```
data:
    models.json: |
        [
            {"name": "mock_vllm", "framework":"vLLM", "instances": [{"endpoint": "http://192.168.2.1:8000/v1/chat/completions", "batch_size": 16, "protocol": "http/2.0", "npu_ids": [1], "pod_name": "vllm", "weight": 3.0}]}
        ]
    routers.json: |
        [
            {"name": "mock_vllm","scheduler":{"policy":"fcfs", "request_ordering_policy":"fcfs"},"dispatcher":{"policy": "round_robin"}, "instances": [{"endpoint": "http://192.168.2.1:8000/v1/chat/completions", "protocol": "http/2.0", "npu_ids": [1], "pod_name": "vllm", "weight": 3.0}]}
        ]
    tasks.json: |
        [
            {"name": "mock_vllm", "model": "mock_vllm"}
        ]
```

# 8. 死循环调用推理

以大模型为例

```
for i in {1..20}; do
    while true; do
        curl -s -k \
            --cert ./kubecfg.crt \
            --key ./kubecfg.key \
            https://${TARGET_HOST}:27321/infermind/scheduler/v1/chat/completions \
            -H "Content-Type: application/json" \
            -d '{
                "model":"Qwen3_8B_w8a8s",
                "messages":[{"role":"user","content":"你好</nothink>"}],
                "stream":false
            }' > /dev/null
    done &
done
```

# 9. `kubectl edit`换镜像

模板

```
kubectl set image deploy/{服务pod名} \
  {容器名}={镜像地址} \
  -n {namespace}
```

## 9.1 中心

完整示例

```
kubectl set image -n dfx-syn01 deploy/iminferscheduler iminferscheduler-software=registry.fusionstage.local:20202/cfm/iminferscheduler:26.9.9.B436
```

dfx-syn01

- 补齐一下末尾的镜像号
- 镜像仓库如果要改名的话，自己改一下

```
kubectl set image -n dfx-syn01 deploy/iminferscheduler iminferscheduler-software=registry.fusionstage.local:20202/cfm/iminferscheduler:
```

dfx-syn02

- 补齐一下末尾的镜像号
- 镜像仓库如果要改名的话，自己改一下

```
kubectl set image -n dfx-syn02 deploy/iminferscheduler iminferscheduler-software=registry.fusionstage.local:20202/cfm/iminferscheduler:
```

其它命名空间

- 补齐一下末尾的`-n` 后面的`<namespace>`
- 补齐一下末尾的镜像号
- 镜像仓库如果要改名的话，自己改一下

```
kubectl set image deploy/iminferscheduler iminferscheduler-software=192.168.2.2:20202/op_svc_pom/iminferscheduler: -n
```

## 9.2 边缘

完整示例

```
kubectl set image -n fst-manage deploy/iminferscheduler iminferscheduler-software=192.168.2.2:20202/op_svc_pom/iminferscheduler:26.9.9.B436
```

fst-manage

- 补齐一下末尾的镜像号

```
kubectl set image -n fst-manage deploy/iminferscheduler iminferscheduler-software=192.168.2.2:20202/op_svc_pom/iminferscheduler:
```

# 10. 模拟网元

## 10.1 大模型

### 10.1.1 前置设置

IP环境变量

```
export TARGET_IP=$(netstat -lntW 2>/dev/null | awk '$4 ~ /:2580$/ {ip=$4; sub(/:[0-9]+$/, "", ip); gsub(/^\[/, "", ip); gsub(/\]$/, "", ip); print ip; exit}'); export PODIP=$( [[ $TARGET_IP == *:* ]] && echo "[$TARGET_IP]" || echo "$TARGET_IP" )
```

任务初始化

```
curl -v -X POST http://${PODIP}:2580/testing/task/create -d '{
    "testingJobList": [
        {
            "testingFuncName": "StartCallingAgent",
            "testingExtParam": {}
        }
    ]
}'
```

### 10.1.2 非流式

```
vi /opt/csp/CoreMindCCommonServiceDemo/scripts/input/llm_request.json
```

```
{
    "reason": {
        "modelServiceName": "llmservice",
        "modelInstanceId": "",
        "modelName": "Qwen3_8B_w8a8s",
        "temperature": 0.1,
         "topP": 0.1,
        "loraId": "",
        "contextMaxlength": 6000,
        "schedgroup": 1,
        "useThird": false
    },
    "request": {
        "intentions": [
            {
                "query": "What is the weather like in shanghai?",
                "modaltype": 0
            }
        ],
    "prompt": [],
    "tools": []
    }
}
```

```
curl -v -X POST http://${PODIP}:2580/testing/task/create -d '{
    "testingJobList": [
        {
            "testingFuncName":"LLMRequest",
            "testingExtParam":
                {
                    "paramsFilePath":"/opt/csp/CoreMindCCommonServiceDemo/scripts/input/llm_request.json"
                }
        }
    ]
}'
```

### 10.1.3 流式

执行下面这行后，会得到一个taskId。

```
curl -v -X POST http://${PODIP}:2580/testing/task/create -d '{"testingJobList":[{"testingFuncName":"CreateTask","testingExtParam":{"reasonParam":"{\"modelServiceName\":\"llmservice\",\"modelInstanceId\":\"\",\"modelName\":\"Qwen3_8B_w8a8s\",\"temperature\":0.1,\"topP\":0.1,\"loraId\":\"\",\"contextMaxlength\":6000,\"schedgroup\":1,\"useThird\":false}","ignoreLlmBitmp":"0","ignoreToolBitmp":"0"}}]}'
```

```
vi /opt/csp/CoreMindCCommonServiceDemo/scripts/input/react_llm_request.json
```

```
{
    "intentions": [
        {
            "query": "你是谁？",
            "modaltype": 0
        }
    ],
    "prompt": [],
    "tools": []
}
```

用前面第一步拿到的taskId修改下面的taskId。

```
curl -v -X POST http://${PODIP}:2580/testing/task/create -d '{
    "testingJobList":[
        {
            "testingFuncName":"ReactLLMRequest",
            "testingExtParam":
                {
                    "taskId":"2_agentruntime-0", 
                    "paramsFilePath": "/opt/csp/CoreMindCCommonServiceDemo/scripts/input/react_llm_request.json"
                }
        }
    ]
}'
```

### 10.1.4 压测

#### 10.1.4.1 非流式

```
curl -v -X POST http://${PODIP}:2580/testing/task/create -d '{
    "testingType": "BATCH",
    "testingTaskDescription": "test description",
    "testingJobList": [
        {
            "testingThreadConfig": {
                "threadScheduleMode": "step",
                "stepConfig": [
                    {
                        "threadNum": 1000,
                        "rateLimitWindowSecond": 1,
                        "rateLimitWindowTokens": 10,
                        "invokeBackoffTimeMs": -1,
                        "durationSecond": 1800
                    }
                ]
            },
            "testingFuncName": "LLMRequest",
            "testingExtParam": {
                "paramsFilePath":"/opt/csp/CoreMindCCommonServiceDemo/scripts/input/llm_request.json"
            }
        }
    ]
}'
```

#### 10.1.4.2 流式

修改taskId。

```
curl -v -X POST http://${PODIP}:2580/testing/task/create -d '{
    "testingType": "BATCH",
    "testingTaskDescription": "test description",
    "testingJobList": [
        {
            "testingThreadConfig": {
                "threadScheduleMode": "step",
                "stepConfig": [
                    {
                        "threadNum": 1000,
                        "rateLimitWindowSecond": 1,
                        "rateLimitWindowTokens": 10,
                        "invokeBackoffTimeMs": -1,
                        "durationSecond": 1800
                    }
                ]
            },
            "testingFuncName": "ReactLLMRequest",
            "testingExtParam": {
                "taskId":"2_agentruntime-0",
"paramsFilePath": "/opt/csp/CoreMindCCommonServiceDemo/scripts/input/react_llm_request.json"
            }
        }
    ]
}'
```

# 11. 测`text_content_security_cn`接口的长链接

```
export TARGET_IP=$(netstat -lntW 2>/dev/null | awk '$4 \~ /:27321$/ {ip=$4; sub(/:[0-9]+$/, "", ip); gsub(/^\[/, "", ip); gsub(/\]$/, "", ip); print ip; exit}'); export TARGET_HOST=$( [[ $TARGET_IP == *:* ]] && echo "[$TARGET_IP]" || echo "$TARGET_IP" )
time curl  -k --cert /opt/sharevolume/oldssl/cert/tls.crt --key /opt/sharevolume/oldssl/cert/tls.key.pwd --pass ${INNER_TLS_PRIVATE_KEY_PWD} -X POST https://${TARGET_HOST}:27321/infermind/infer/v1/text_content_security_cn -w "\nTime: %{time_total}s\n"  --data '{
"params": {
"batch": "true"
},
"input": [
{"data":"的一是在不了有和人这中大为上个国我以要他时来用们生到作地于出就分对成会可主发年动同工也能下过子说产种面而方后多定行学法所民得经十三之进着等部度家电力里如水化高自二理起小物现实加量都两体制机当使点从业本去把性好应开它合还因由其些然前外天政四日那社义事平形相"}
]
}' --next -k --cert /opt/sharevolume/oldssl/cert/tls.crt --key /opt/sharevolume/oldssl/cert/tls.key.pwd --pass ${INNER_TLS_PRIVATE_KEY_PWD} -X POST https://${TARGET_HOST}:27321/infermind/infer/v1/text_content_security_cn -w "\nTime: %{time_total}s\n"  --data '{
"params": {
"batch": "true"
},
"input": [
{"data":"的一是在不了有和人这中大为上个国我以要他时来用们生到作地于出就分对成会可主发年动同工也能下过子说产种面而方后多定行学法所民得经十三之进着等部度家电力里如水化高自二理起小物现实加量都两体制机当使点从业本去把性好应开它合还因由其些然前外天政四日那社义事平形相"}
]
}' --next -k --cert /opt/sharevolume/oldssl/cert/tls.crt --key /opt/sharevolume/oldssl/cert/tls.key.pwd --pass ${INNER_TLS_PRIVATE_KEY_PWD} -X POST https://${TARGET_HOST}:27321/infermind/infer/v1/text_content_security_cn -w "\nTime: %{time_total}s\n"  --data '{
"params": {
"batch": "true"
},
"input": [
{"data":"的一是在不了有和人这中大为上个国我以要他时来用们生到作地于出就分对成会可主发年动同工也能下过子说产种面而方后多定行学法所民得经十三之进着等部度家电力里如水化高自二理起小物现实加量都两体制机当使点从业本去把性好应开它合还因由其些然前外天政四日那社义事平形相"}
]
}' --next -k --cert /opt/sharevolume/oldssl/cert/tls.crt --key /opt/sharevolume/oldssl/cert/tls.key.pwd --pass ${INNER_TLS_PRIVATE_KEY_PWD} -X POST https://${TARGET_HOST}:27321/infermind/infer/v1/text_content_security_cn -w "\nTime: %{time_total}s\n"  --data '{
"params": {
"batch": "true"
},
"input": [
{"data":"的一是在不了有和人这中大为上个国我以要他时来用们生到作地于出就分对成会可主发年动同工也能下过子说产种面而方后多定行学法所民得经十三之进着等部度家电力里如水化高自二理起小物现实加量都两体制机当使点从业本去把性好应开它合还因由其些然前外天政四日那社义事平形相"}
]
}' --next -k --cert /opt/sharevolume/oldssl/cert/tls.crt --key /opt/sharevolume/oldssl/cert/tls.key.pwd --pass ${INNER_TLS_PRIVATE_KEY_PWD} -X POST https://${TARGET_HOST}:27321/infermind/infer/v1/text_content_security_cn -w "\nTime: %{time_total}s\n"  --data '{
"params": {
"batch": "true"
},
"input": [
{"data":"的一是在不了有和人这中大为上个国我以要他时来用们生到作地于出就分对成会可主发年动同工也能下过子说产种面而方后多定行学法所民得经十三之进着等部度家电力里如水化高自二理起小物现实加量都两体制机当使点从业本去把性好应开它合还因由其些然前外天政四日那社义事平形相"}
]
}' --next -k --cert /opt/sharevolume/oldssl/cert/tls.crt --key /opt/sharevolume/oldssl/cert/tls.key.pwd --pass ${INNER_TLS_PRIVATE_KEY_PWD} -X POST https://${TARGET_HOST}:27321/infermind/infer/v1/text_content_security_cn -w "\nTime: %{time_total}s\n"  --data '{
"params": {
"batch": "true"
},
"input": [
{"data":"的一是在不了有和人这中大为上个国我以要他时来用们生到作地于出就分对成会可主发年动同工也能下过子说产种面而方后多定行学法所民得经十三之进着等部度家电力里如水化高自二理起小物现实加量都两体制机当使点从业本去把性好应开它合还因由其些然前外天政四日那社义事平形相"}
]
}' --next -k --cert /opt/sharevolume/oldssl/cert/tls.crt --key /opt/sharevolume/oldssl/cert/tls.key.pwd --pass ${INNER_TLS_PRIVATE_KEY_PWD} -X POST https://${TARGET_HOST}:27321/infermind/infer/v1/text_content_security_cn -w "\nTime: %{time_total}s\n"  --data '{
"params": {
"batch": "true"
},
"input": [
{"data":"的一是在不了有和人这中大为上个国我以要他时来用们生到作地于出就分对成会可主发年动同工也能下过子说产种面而方后多定行学法所民得经十三之进着等部度家电力里如水化高自二理起小物现实加量都两体制机当使点从业本去把性好应开它合还因由其些然前外天政四日那社义事平形相"}
]
}' --next -k --cert /opt/sharevolume/oldssl/cert/tls.crt --key /opt/sharevolume/oldssl/cert/tls.key.pwd --pass ${INNER_TLS_PRIVATE_KEY_PWD} -X POST https://${TARGET_HOST}:27321/infermind/infer/v1/text_content_security_cn -w "\nTime: %{time_total}s\n"  --data '{
"params": {
"batch": "true"
},
"input": [
{"data":"的一是在不了有和人这中大为上个国我以要他时来用们生到作地于出就分对成会可主发年动同工也能下过子说产种面而方后多定行学法所民得经十三之进着等部度家电力里如水化高自二理起小物现实加量都两体制机当使点从业本去把性好应开它合还因由其些然前外天政四日那社义事平形相"}
]
}'  --next -k --cert /opt/sharevolume/oldssl/cert/tls.crt --key /opt/sharevolume/oldssl/cert/tls.key.pwd --pass ${INNER_TLS_PRIVATE_KEY_PWD} -X POST https://${TARGET_HOST}:27321/infermind/infer/v1/text_content_security_cn -w "\nTime: %{time_total}s\n"  --data '{
"params": {
"batch": "true"
},
"input": [
{"data":"的一是在不了有和人这中大为上个国我以要他时来用们生到作地于出就分对成会可主发年动同工也能下过子说产种面而方后多定行学法所民得经十三之进着等部度家电力里如水化高自二理起小物现实加量都两体制机当使点从业本去把性好应开它合还因由其些然前外天政四日那社义事平形相"}
]
}'  --next -k --cert /opt/sharevolume/oldssl/cert/tls.crt --key /opt/sharevolume/oldssl/cert/tls.key.pwd --pass ${INNER_TLS_PRIVATE_KEY_PWD} -X POST https://${TARGET_HOST}:27321/infermind/infer/v1/text_content_security_cn -w "\nTime: %{time_total}s\n"  --data '{
"params": {
"batch": "true"
},
"input": [
{"data":"的一是在不了有和人这中大为上个国我以要他时来用们生到作地于出就分对成会可主发年动同工也能下过子说产种面而方后多定行学法所民得经十三之进着等部度家电力里如水化高自二理起小物现实加量都两体制机当使点从业本去把性好应开它合还因由其些然前外天政四日那社义事平形相"}
]
}'  --next -k --cert /opt/sharevolume/oldssl/cert/tls.crt --key /opt/sharevolume/oldssl/cert/tls.key.pwd --pass ${INNER_TLS_PRIVATE_KEY_PWD} -X POST https://${TARGET_HOST}:27321/infermind/infer/v1/text_content_security_cn -w "\nTime: %{time_total}s\n"  --data '{
"params": {
"batch": "true"
},
"input": [
{"data":"的一是在不了有和人这中大为上个国我以要他时来用们生到作地于出就分对成会可主发年动同工也能下过子说产种面而方后多定行学法所民得经十三之进着等部度家电力里如水化高自二理起小物现实加量都两体制机当使点从业本去把性好应开它合还因由其些然前外天政四日那社义事平形相"}
]
}'
```