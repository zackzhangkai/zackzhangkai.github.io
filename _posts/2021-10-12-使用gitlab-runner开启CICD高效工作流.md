---
layout: post
published: true
title: 使用gitlab-runner开启CICD高效工作流
categories: [实战]
tags: [CICD]
---
* content
{:toc}

# 背景

Github action可以用于自动化测试，Gitlab CICD也有类似的功能。Github有免费的Runner可以使用，但Gitlab需要自己添加runner。添加完Runner后，用户同样只用写yaml定义自动化执行的job，极大的提升了生产效率。

# 添加Runner

Gitlab的runner是一个执行器，可以是一个可以执行自动化脚本的虚机（或是docker或是K8s的pod）。

![runner](/images/gitlab-runner.jpg)

选择手动添加

## 在 linux centos 上安装runner

添加 yum 仓库源
```bash
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.rpm.sh | sudo bash
```

安装

```bash
sudo yum install gitlab-runner -y
```

注册

```bash
gitlab-runner register
```

Gitlab页面上有相应的url及token

此时会生成配置文件 /etc/gitlab-runner

```bash
# ls /etc/gitlab-runner/
config.toml
```

启动服务

```bash
systemctl start gitlab-runner
```

添加成功后，在gitlab页面会显示成功

![status](/images/gitlab-runner-status.jpg)

如果本地开发环境，只能使用ssh clone 协议，不能使用https，就需要修改 runner 的配置文件

```toml
[[runners]]
  environment = ["GIT_STRATEGY=none"]
  pre_build_script = '''
    # Fetching using ssh (via pre_build_script in config.toml)
    if [ -d "${CI_PROJECT_DIR}" ]; then rm -rf "${CI_PROJECT_DIR}"; fi
    mkdir -p "${CI_PROJECT_DIR}"
    cd "${CI_PROJECT_DIR}"
    git init
    git remote add origin "git@code.xx.cn:csf/csm/xxx.git"
    git fetch origin "${CI_COMMIT_SHA}"
    git reset --hard FETCH_HEAD
  '''
  name = "m22"
  url = "https://code.xx.cn/"
  token = "_v5byRzxPaqWdBzW2XDx"
  executor = "shell"
  [runners.custom_build_dir]
  [runners.cache]
    [runners.cache.s3]
    [runners.cache.gcs]
    [runners.cache.azure]
```

然后重启gitlab-runner

# 编写 yaml 文件

gitlab默认读取root目录，下面的gitlab做了自动化测试，并将结果发送邮件。

```yaml
stages:
  - test

unit_tests:
  stage: test
  script:
    - make test

send_email_on_fail:
  script: mail -r "xxx@istio.com" -s "Failed - Code Test " `cat OWNERS` <<< "Gitlab Pipeline Test Failed . Refer to  https://code.private.cn/csf/csm/xxx/-/pipelines"
  rules:
    - when: on_failure

send_email_on_sucess:
  script: mail -r "xxx@istio.com" -s "Success - Code Test " `cat OWNERS` <<< "Gitlab Pipeline Test Success. Refer to  https://code.private.cn/csf/csm/xxx/-/pipelines"
  rules:
    - when: on_success
```

提交代码后，就会触发pipeline job

![pipeline](/images/gitlab-runner-pipeline.jpg)

当执行完测试后，会收到邮件通知

![gitlab-runner-mail](/images/gitlab-runner-mail.jpg)

# 总结

gitlab cicd 类似于 github action, 可以极大地升工程效率。

如果runner使用shell，本质上就是在linux上执行gitlab-runner命令，如果执行pipeline失败，可以直接去对应的目录下，执行对应的命令，查看报错信息。

```bash
# ps -ef|grep gitlab-runner
root     21285     1  0 10月11 ?      00:06:25 /usr/bin/gitlab-runner run --working-directory /home/gitlab-runner --config /etc/gitlab-runner/config.toml --service gitlab-runner --user gitlab-runner
```

另外发送邮件使用的是`mailx`命令，需要在Linux上安装对应的软件包。
