
# 一、先查看公司证书的颁发者

这一步仅用于诊断，使用 `-k` 没有问题，因为不会下载执行文件：

```bash
curl -vkI --max-time 15 https://claude.ai 2>&1 \
    | grep -Ei 'subject:|issuer:|SSL certificate'
```

一般会看到类似：

```text
subject: CN=claude.ai
issuer: CN=某公司安全根证书
```

重点记住 `issuer` 中的名称。

# 二、从 Windows 导出公司根证书

在 Windows 中按：

```text
Win + R
```

输入：

```text
certmgr.msc
```

依次打开：

```text
受信任的根证书颁发机构
└── 证书
```

根据刚才 `issuer` 显示的名称，寻找公司证书，可能包含：

```text
{Company}
NetentSec
企业安全
Enterprise
Root CA
```

找到以后：

```text
右键证书
→ 所有任务
→ 导出
→ 不导出私钥
→ Base-64 编码 X.509 (.CER)
```

保存到 Windows 下载目录，例如：

```text
C:\Users\你的用户名\Downloads\corp-root.cer
```

如果在 `certmgr.msc` 里找不到，再运行：

```text
certlm.msc
```

然后到同样的：

```text
受信任的根证书颁发机构 → 证书
```

里面寻找。这是计算机级别的证书库。

# 三、在 WSL 中找到 Windows 用户目录

执行：

```bash
WIN_HOME=$(wslpath "$(cmd.exe /c 'echo %USERPROFILE%' 2>/dev/null | tr -d '\r')")
echo "$WIN_HOME"
```

应该输出类似：

```text
/mnt/c/Users/usrname
```

检查导出的证书：

```bash
CERT_FILE="$WIN_HOME/Downloads/corp-root.cer"

ls -l "$CERT_FILE"
```

正常应该能看到文件。

# 四、将证书安装到 WSL

直接执行下面这一整段：

```bash
CERT_FILE="$WIN_HOME/Downloads/corp-root.cer"
TARGET_FILE="/usr/local/share/ca-certificates/{company}-corp-root.crt"

if grep -q "BEGIN CERTIFICATE" "$CERT_FILE"; then
    echo "[install_corp_ca] 检测到 Base64/PEM 格式证书"
    sudo cp "$CERT_FILE" "$TARGET_FILE"
else
    echo "[install_corp_ca] 检测到 DER 格式证书，开始转换"
    openssl x509 \
        -inform DER \
        -in "$CERT_FILE" \
        -out /tmp/{company}-corp-root.crt

    sudo cp /tmp/{company}-corp-root.crt "$TARGET_FILE"
fi

sudo chmod 644 "$TARGET_FILE"
sudo update-ca-certificates
```

正常会出现类似：

```text
1 added, 0 removed
```

检查证书是否已经加入：

```bash
ls -l /etc/ssl/certs | grep -i huawei
```

这里不一定能匹配到 `huawei`，因为链接名可能根据证书名称生成。关键还是看 `update-ca-certificates` 是否显示 `1 added`。

# 五、给 Claude Code 配置证书路径

Anthropic 官方说明，在企业代理使用自定义证书时，需要配置 `SSL_CERT_FILE` 和 `NODE_EXTRA_CA_CERTS`。Claude Code支持通过 `HTTP_PROXY`、`HTTPS_PROXY` 连接企业代理。([Claude Platform Docs](https://docs.anthropic.com/en/docs/claude-code/corporate-proxy?utm_source=chatgpt.com "Corporate proxy configuration - Anthropic"))

执行：

```bash
cat >> ~/.bashrc <<'EOF'

# Claude Code / Node.js corporate CA
export SSL_CERT_FILE=/etc/ssl/certs/ca-certificates.crt
export NODE_EXTRA_CA_CERTS=/etc/ssl/certs/ca-certificates.crt
EOF
```

重新加载：

```bash
source ~/.bashrc
```

确认：

```bash
echo "$SSL_CERT_FILE"
echo "$NODE_EXTRA_CA_CERTS"
```

应该都输出：

```text
/etc/ssl/certs/ca-certificates.crt
```

你当前的代理配置可以保留：

```bash
export http_proxy=http://172.23.48.1:18080
export https_proxy=http://172.23.48.1:18080
export HTTP_PROXY=$http_proxy
export HTTPS_PROXY=$https_proxy
```

# 六、重新测试 HTTPS

执行：

```bash
curl -I --max-time 15 https://claude.ai
```

正常情况下，不再出现：

```text
self-signed certificate in certificate chain
```

而是看到类似：

```text
HTTP/1.1 200 Connection established

HTTP/2 200
```

再测试安装脚本：

```bash
curl -I --max-time 15 https://claude.ai/install.sh
```

只要得到 `200`、`301` 或 `302`，并且没有证书错误，就可以继续。

# 七、安装 Claude Code

建议先下载，再执行：

```bash
curl -fsSL https://claude.ai/install.sh \
    -o /tmp/claude-install.sh
```

确认文件存在：

```bash
ls -lh /tmp/claude-install.sh
head -n 10 /tmp/claude-install.sh
```

然后执行：

```bash
bash /tmp/claude-install.sh
```

安装完成后：

```bash
export PATH="$HOME/.local/bin:$PATH"
hash -r

claude --version
```

永久加入 PATH：

```bash
grep -qxF 'export PATH="$HOME/.local/bin:$PATH"' ~/.bashrc || \
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc

source ~/.bashrc
```

检查：

```bash
which claude
claude --version
claude doctor
```

Claude Code官方支持 WSL，并可在项目目录中直接运行 `claude`；安装完成后可以使用 `claude doctor` 检查安装状态。([Claude Platform Docs](https://docs.anthropic.com/en/docs/claude-code/getting-started?utm_source=chatgpt.com "Set up Claude Code - Anthropic"))

# 八、修复 apt 的 Docker 源超时

你现在把原来的 `99proxy` 禁用了：

```text
/etc/apt/apt.conf.d/99proxy.disabled
```

而 `sudo` 默认不会保留当前用户的代理环境变量，所以 `apt` 对 `download.docker.com` 使用了直连，最终超时。

公司证书安装完成后，可以把 apt 代理改为正确的 `18080`：

```bash
sudo tee /etc/apt/apt.conf.d/99proxy >/dev/null <<'EOF'
Acquire::http::Proxy  "http://172.23.48.1:18080/";
Acquire::https::Proxy "http://172.23.48.1:18080/";
EOF
```

检查：

```bash
cat /etc/apt/apt.conf.d/99proxy
```

重新执行：

```bash
sudo apt update
```

如果 Docker 官方源依然访问不了，但你暂时不需要通过 apt 安装 Docker，可以先找到对应配置：

```bash
grep -Rni "download.docker.com" \
    /etc/apt/sources.list \
    /etc/apt/sources.list.d 2>/dev/null
```

假设结果显示文件是：

```text
/etc/apt/sources.list.d/docker.list
```

可以临时禁用：

```bash
sudo mv /etc/apt/sources.list.d/docker.list \
             /etc/apt/sources.list.d/docker.list.disabled
```

然后：

```bash
sudo apt update
```

当前最核心的一步是：​**在 Windows 中导出公司根证书，再导入 WSL**​。证书问题解决后，Claude Code安装程序和后续 Claude Code 网络请求才能稳定工作。


#wsl证书校验 #证书校验 #wsl