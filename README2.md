```bash
# 创建环境
$ py -m venv venv

# 启用环境
$ venv\Scripts\activate.bat

# 安装依赖
$ pip3 install litellm litellm[proxy] -i https://pypi.tuna.tsinghua.edu.cn/simple

# 将依赖写入 requirements 文件，以便下次使用
$ pip3 freeze > requirements.txt

# 下次直接安装
$ pip3 install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple

# 离开 venv
$ venv\Scripts\deactivate.bat
```
