### k8s命令补全
```
- 安装 bash-completion补全工具
centos系统
yum install -y bash-completion

ubuntu系统使用以下命令
apt-get install bash-completion

2.加载shell并运行
source /usr/share/bash-completion/bash_completion

3.启用 kubectl 自动补全
source <(kubectl completion bash)

echo "source <(kubectl completion bash)" >> ~/.bashrc
```
