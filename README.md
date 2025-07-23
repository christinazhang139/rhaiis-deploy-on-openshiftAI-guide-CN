# 在 OpenShift 上部署LLM，使用Red Hat Inference Server 完整部署指南

## 环境确认 ✅
确保以下组件已就绪：
- KServe 控制器运行正常
- Knative Serving 所有组件运行正常
- Istio 系统组件运行正常
- DataScienceCluster 状态为 Ready

**必需的Operators已安装**：
- ✅ **NVIDIA GPU Operator** - 提供GPU支持
- ✅ **Red Hat OpenShift AI** - 提供AI/ML平台功能
- ✅ **Red Hat OpenShift Serverless** - 提供Knative Serving支持
- ✅ **Red Hat OpenShift Service Mesh 2** - 提供Istio服务网格支持
- ✅ **Node Feature Discovery Operator** - 自动发现节点特性
- ✅ **Package Server** - 管理Operator包

**验证Operators状态**：
```bash
# 检查必需的Operators状态
oc get csv -A | grep -E "(gpu-operator|rhods|serverless|servicemesh|nfd)"

# 查看DataScienceCluster状态
oc get datasciencecluster -A
```

---

## 第一步：创建并切换到工作命名空间

```bash
# 创建专用命名空间
oc new-project ai-inference-demo

# 确认当前项目
oc project ai-inference-demo
```

---

## 第二步：配置命名空间为Service Mesh成员

```bash
# 给命名空间添加Istio injection标签
oc label namespace ai-inference-demo istio-injection=enabled

# 检查是否有ServiceMeshMemberRoll需要更新
oc get servicemeshmemberroll -A

# 如果存在ServiceMeshMemberRoll，将命名空间添加到成员列表
oc patch servicemeshmemberroll default -n istio-system --type='json' -p='[{"op": "add", "path": "/spec/members/-", "value": "ai-inference-demo"}]'

# 验证命名空间标签
oc get namespace ai-inference-demo --show-labels

# 启用 anyuid SCC，避免 token 和权限问题
oc adm policy add-scc-to-user anyuid -z default -n ai-inference-demo
```

---

## 第三步：配置Red Hat Registry镜像拉取权限

```bash
# 创建Red Hat Registry的pull secret（需要有效的Red Hat Customer Portal凭证）
oc create secret docker-registry redhat-registry-secret \
    --docker-server=registry.redhat.io \
    --docker-username=YOUR_RH_USERNAME \
    --docker-password='YOUR_RH_PASSWORD' \
    --docker-email=YOUR_EMAIL

# 将secret链接到默认service account
oc secrets link default redhat-registry-secret --for=pull
oc secrets link deployer redhat-registry-secret --for=pull

# 验证secret创建
oc get secret redhat-registry-secret
```

**注意**: 请替换为您的实际Red Hat Customer Portal凭证

---

## 第四步：创建ServingRuntime

**镜像版本说明**: `registry.redhat.io/rhaiis/vllm-cuda-rhel9:3.2.0-1752784628` 是目前最新版本

💡 **提示**: 最新版本可在 [Red Hat Catalog](https://catalog.redhat.com/en) 搜索关键字 `rhaiis` 获取

```bash
cat <<EOF | oc apply -f -
apiVersion: serving.kserve.io/v1alpha1
kind: ServingRuntime
metadata:
  name: red-hat-vllm-runtime
  namespace: ai-inference-demo
spec:
  supportedModelFormats:
    - name: vllm
      version: "1"
      autoSelect: true
    - name: pytorch
      version: "1"
      autoSelect: true
  containers:
    - name: kserve-container
      image: registry.redhat.io/rhaiis/vllm-cuda-rhel9:3.2.0-1752784628
      ports:
        - containerPort: 8080
          name: http1
          protocol: TCP
      command: ["python", "-m", "vllm.entrypoints.openai.api_server"]
      args:
        - "--model"
        - "/mnt/models/DialoGPT-small"
        - "--host"
        - "0.0.0.0"
        - "--port"
        - "8080"
        - "--served-model-name"
        - "DialoGPT-small"
        - "--max-model-len"
        - "1024"
        - "--disable-log-requests"
      env:
        - name: VLLM_CPU_KVCACHE_SPACE
          value: "4"
        - name: HF_HUB_OFFLINE
          value: "1"
        - name: TRANSFORMERS_OFFLINE
          value: "1"
      resources:
        requests:
          cpu: "1"
          memory: "4Gi"
          nvidia.com/gpu: "1"
        limits:
          cpu: "2"
          memory: "8Gi"
          nvidia.com/gpu: "1"
      readinessProbe:
        httpGet:
          path: /health
          port: 8080
        initialDelaySeconds: 120
        periodSeconds: 10
        timeoutSeconds: 10
      livenessProbe:
        httpGet:
          path: /health
          port: 8080
        initialDelaySeconds: 180
        periodSeconds: 30
        timeoutSeconds: 10
EOF
```

---

## 第五步：验证ServingRuntime

```bash
# 检查ServingRuntime状态
oc get servingruntime red-hat-vllm-runtime

# 查看详细信息
oc describe servingruntime red-hat-vllm-runtime
```

---

## 第六步：创建PVC用于模型存储

```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: model-storage-pvc
  namespace: ai-inference-demo
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: gp3-csi
EOF

# 验证PVC创建
oc get pvc model-storage-pvc
```

---

## 第七步：下载模型到PVC

```bash
cat <<EOF | oc apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: dialogpt-model-downloader
  namespace: ai-inference-demo
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: downloader
        image: python:3.12-slim
        command:
        - /bin/sh
        - -c
        - |
          set -e
          export HOME=/tmp
          pip install --no-cache-dir --user huggingface_hub
          export PATH="\$HOME/.local/bin:\$PATH"
          mkdir -p /models/DialoGPT-small
          python3 -c "from huggingface_hub import hf_hub_download; files = ['config.json', 'pytorch_model.bin', 'tokenizer_config.json', 'vocab.json', 'merges.txt']; [hf_hub_download(repo_id='microsoft/DialoGPT-small', filename=f, local_dir='/models/DialoGPT-small') for f in files]"
          rm /models/DialoGPT-small/tokenizer.json || true
          ls -la /models/DialoGPT-small/
          du -sh /models/DialoGPT-small/pytorch_model.bin
        volumeMounts:
        - name: model-storage
          mountPath: /models
        env:
        - name: HF_TOKEN
          value: "YOUR_HF_TOKEN_HERE"
      volumes:
      - name: model-storage
        persistentVolumeClaim:
          claimName: model-storage-pvc
EOF
```

**注意**: 如果需要访问私有模型，请替换 `YOUR_HF_TOKEN_HERE` 为您的Hugging Face Token

---

## 第八步：监控模型下载进度

```bash
# 查看Job状态
oc get jobs

# 查看下载日志
oc logs job/dialogpt-model-downloader -f

# 等待看到 "Download completed!" 消息
```

---

## 第九步：验证模型文件位置

```bash
# 创建调试Pod检查PVC中的模型文件位置
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: pvc-explorer
  namespace: ai-inference-demo
spec:
  restartPolicy: Never
  containers:
  - name: explorer
    image: busybox:latest
    imagePullPolicy: IfNotPresent
    command: ["sleep", "300"]
    volumeMounts:
    - name: model-storage
      mountPath: /data
  volumes:
  - name: model-storage
    persistentVolumeClaim:
      claimName: model-storage-pvc
EOF

# 检查模型文件位置 - 确定PVC上已经有下载的LLM
oc exec pvc-explorer -- ls -la /data/
oc exec pvc-explorer -- ls -la /data/DialoGPT-small/
oc exec pvc-explorer -- find /data -name "config.json"
oc exec pvc-explorer -- du -h /data/DialoGPT-small/pytorch_model.bin

# 验证内容（必须）
oc exec pvc-explorer -- head -n 5 /data/DialoGPT-small/config.json
oc exec pvc-explorer -- head -n 5 /data/DialoGPT-small/tokenizer_config.json

# 清理调试Pod
oc delete pod pvc-explorer
```

**预期结果**：应该看到 `/data/DialoGPT-small/` 目录包含以下文件：
- `config.json`
- `pytorch_model.bin`
- `tokenizer_config.json`
- `vocab.json`
- `merges.txt`

---

## 第十步：创建InferenceService

```bash
cat <<EOF | oc apply -f -
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: dialogpt-small-service
  namespace: ai-inference-demo
  annotations:
    sidecar.istio.io/inject: "false"  # 禁用 Istio sidecar，避免 envoy 报错
    serving.kserve.io/enable-service-account-token-mount: "true"  # 挂载 token，解决认证失败
spec:
  predictor:
    model:
      modelFormat:
        name: pytorch
      runtime: red-hat-vllm-runtime
      storageUri: pvc://model-storage-pvc
      resources:
        requests:
          cpu: "1"
          memory: "4Gi"
          nvidia.com/gpu: "1"
        limits:
          cpu: "2"
          memory: "8Gi"
          nvidia.com/gpu: "1"
      env:
        - name: VLLM_GPU_MEMORY_UTILIZATION
          value: "0.5"
EOF
```

**模型信息**：
- **microsoft/DialoGPT-small**: 117MB，117M参数
- **本地存储**: 从PVC加载，启动快速且稳定
- **对话生成**: 适合测试推理功能
- **vLLM优化**: 使用vLLM推理引擎，性能更好

---

## 第十一步：监控部署状态

```bash
# 实时监控InferenceService状态
oc get inferenceservice dialogpt-small-service -w

# 查看相关pods
oc get pods -l serving.kserve.io/inferenceservice=dialogpt-small-service

# 查看详细状态
oc describe inferenceservice dialogpt-small-service

# 查看事件
oc get events --sort-by='.lastTimestamp' | head -20
```

**成功标志**：当看到 `READY=True` 时，表示服务已成功启动。

---

## 第十二步：简单对话测试

**重要说明**: DialoGPT-small是一个小型对话模型（117M参数），回复质量有限，有时可能生成不够连贯的内容，这是正常现象。

### 最简单的测试方法

```bash
# 设置变量
PREDICTOR_POD=$(oc get pods -l serving.kserve.io/inferenceservice=dialogpt-small-service -o jsonpath='{.items[0].metadata.name}')

# 基础健康检查
echo "=== 健康检查 ==="
oc exec $PREDICTOR_POD -c kserve-container -- curl -s localhost:8080/health

# 对话测试1：简单问候
echo -e "\n=== 我问：Hello, how are you? ==="
oc exec $PREDICTOR_POD -c kserve-container -- curl -s -X POST localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "DialoGPT-small",
    "messages": [{"role": "user", "content": "Hello, how are you?"}],
    "max_tokens": 30,
    "temperature": 0.7
  }'

# 对话测试2：询问名字
echo -e "\n=== 我问：What is your name? ==="
oc exec $PREDICTOR_POD -c kserve-container -- curl -s -X POST localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "DialoGPT-small",
    "messages": [{"role": "user", "content": "What is your name?"}],
    "max_tokens": 20,
    "temperature": 0.8
  }'

# 对话测试3：简单问题
echo -e "\n=== 我问：Hi ==="
oc exec $PREDICTOR_POD -c kserve-container -- curl -s -X POST localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "DialoGPT-small",
    "messages": [{"role": "user", "content": "Hi"}],
    "max_tokens": 10,
    "temperature": 0.5
  }'
```

---

## 第十三步：性能和监控检查

### 查看资源使用情况

```bash
# 查看Pod资源使用（需要metrics-server支持）
oc adm top pod -l serving.kserve.io/inferenceservice=dialogpt-small-service

# 如果上面命令不工作，使用以下替代方法：

# 查看Pod的资源配置和限制
PREDICTOR_POD=$(oc get pods -l serving.kserve.io/inferenceservice=dialogpt-small-service -o jsonpath='{.items[0].metadata.name}')
oc describe pod $PREDICTOR_POD | grep -A10 -B5 "Limits\|Requests"

# 查看Pod状态和运行时间
oc get pod $PREDICTOR_POD -o wide

# 查看节点资源使用情况
oc adm top nodes

# 如果metrics-server不可用，查看Pod基本信息
oc get pod $PREDICTOR_POD -o jsonpath='{.status.containerStatuses[*].restartCount}'
echo " (重启次数)"
```

### 服务状态检查

```bash
# 检查InferenceService整体状态
oc get inferenceservice dialogpt-small-service -o yaml | grep -A20 status

# 查看所有相关资源状态
oc get pods,svc,inferenceservice -l serving.kserve.io/inferenceservice=dialogpt-small-service

# 查看最近的集群事件
oc get events --sort-by='.lastTimestamp' | head -20

# 检查服务端点
oc get endpoints dialogpt-small-service-predictor
```

---

## 常见问题排查

### 1. 推理服务无法访问

```bash
# 检查service状态
oc get svc | grep dialogpt-small-service

# 检查endpoints
oc get endpoints dialogpt-small-service-predictor

# 检查pods状态
oc get pods -l serving.kserve.io/inferenceservice=dialogpt-small-service
```

### 2. 模型加载失败

```bash
# 查看pod事件
oc describe pod $PREDICTOR_POD

# 检查模型文件
oc exec $PREDICTOR_POD -c kserve-container -- ls -la /mnt/models/DialoGPT-small/

# 查看vLLM启动日志
oc logs $PREDICTOR_POD -c kserve-container | grep -i error
```

### 3. 内存或GPU资源不足

```bash
# 检查节点资源
oc describe nodes | grep -A5 -B5 "Allocated resources"

# 降低资源要求
oc patch inferenceservice dialogpt-small-service --type='merge' -p='{
  "spec": {
    "predictor": {
      "model": {
        "resources": {
          "requests": {"cpu": "500m", "memory": "2Gi"},
          "limits": {"cpu": "1", "memory": "4Gi"}
        }
      }
    }
  }
}'
```

---

## 清理资源

```bash
# 删除测试Pod
oc delete pod inference-test-client

# 删除InferenceService
oc delete inferenceservice dialogpt-small-service

# 删除ServingRuntime
oc delete servingruntime red-hat-vllm-runtime

# 删除下载Job
oc delete job dialogpt-model-downloader

# 删除PVC（注意：这会删除所有下载的模型）
oc delete pvc model-storage-pvc

# 删除Pull Secret
oc delete secret redhat-registry-secret

# 删除整个项目
oc delete project ai-inference-demo
```

---

## 总结

这个指南提供了一个完整的Red Hat Inference Server部署和内部测试流程：

✅ **优势**：
- 🔒 **安全**: 所有测试都在集群内部进行，无需暴露外部端点
- ⚡ **高效**: 使用PVC本地存储，模型加载快速
- 🛠️ **灵活**: 支持多种测试方法和交互方式
- 📊 **可观测**: 提供详细的监控和日志查看方法

✅ **适用场景**：
- 开发和测试环境验证
- 内部API集成测试
- 模型性能评估
- 安全合规环境下的AI服务部署

按照这个指南，您可以在不创建外部路由的情况下，完整地部署和测试Red Hat Inference Server！
