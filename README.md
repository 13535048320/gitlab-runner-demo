# Gitlab-runner使用
参考文章：
https://www.webq.top/doc/ci#a7
## 1.安装
### 1.1 gitlab 服务器普通搭建方式
<b>下载地址:https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el7</b>
<b>下载RPM包，并安装，例子</b>
```
wget https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el7/gitlab-ce-12.2.1-ce.0.el7.x86_64.rpm
rpm -i gitlab-ce-12.2.1-ce.0.el7.x86_64.rpm
```
<b>配置</b>
```
vi /etc/gitlab/gitlab.rb
    external_url 'http://<IP>'
```

### 1.2 gitlab 服务器容器搭建方式
<b>启动</b>
```
docker run -d \
   -p 80:80 \
   -p 443:443 \
   -p 122:22 \
   --name gitlab \
   --restart unless-stopped \
   -v /data/gitlab/config:/etc/gitlab \
   -v /data/gitlab/logs:/var/log/gitlab \
   -v /data/gitlab/data:/var/opt/gitlab \
   gitlab/gitlab-ce:latest
```
<b>配置</b>
```
vi /data/gitlab/config/gitlab.rb
    external_url 'http://<IP>'
```
<b>配置后重启</b>
```
docker restart gitlab
```

### 1.3 gitlab-runner容器安装方式
```
docker run -d --name gitlab-runner --restart always \
-v /data/gitlab-runner:/etc/gitlab-runner \
-v /var/run/docker.sock:/var/run/docker.sock \
gitlab/gitlab-runner:latest
```
使用容器部署，可能有部分shell命令无法使用，例如mvn，需要另外构建安装maven的镜像

### 1.4 gitlab-runner普通安装
<b>下载命令</b>
```
sudo curl -L --output /usr/local/bin/gitlab-runner https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-amd64
```
<b>执行权限</b>
```
sudo chmod +x /usr/local/bin/gitlab-runner
```
<b>创建用户</b>
```
sudo useradd --comment 'GitLab Runner' --create-home gitlab-runner --shell /bin/bash
```
<b>启动</b>
gitlab-runner install --user=gitlab-runner --working-directory=/home/gitlab-runner
sudo gitlab-runner start

## 2.配置
### 2.1 查看token
![picture1](https://github.com/13535048320/gitlab-runner-demo/blob/master/images/%E5%9B%BE%E7%89%871.png)
![picture2](https://github.com/13535048320/gitlab-runner-demo/blob/master/images/%E5%9B%BE%E7%89%872.png)

### 2.2 配置runner
执行命令
<b>普通方式</b>
```
gitlab-runner register
```
<b>容器方式</b>
```
docker exec -it gitlab-runner gitlab-runner register
```


<b>根据2.1显示的输入</b>
![picture3](https://github.com/13535048320/gitlab-runner-demo/blob/master/images/%E5%9B%BE%E7%89%873.png)
注意输入要使用的执行方式，可以直接使用shell，也可以使用docker和ssh但是需要其他配置
![picture4](https://github.com/13535048320/gitlab-runner-demo/blob/master/images/%E5%9B%BE%E7%89%874.png)

配置完成后，等待一会儿，gitlab服务器页面会显示runner，绿色为正常状态
![picture5](https://github.com/13535048320/gitlab-runner-demo/blob/master/images/%E5%9B%BE%E7%89%875.png)


## 3.编写.gitlab-ci.yml文件
### 3.1项目结构
![picture6](https://github.com/13535048320/gitlab-runner-demo/blob/master/images/%E5%9B%BE%E7%89%876.png)

### 3.2.gitlab-ci.yml文件内容
```
# 在作业之前执行的脚本或命令
before_script:
  - source ~/.bashrc
  
# 在作业之后执行的脚本或命令 
after_script:
  - echo "build success"

# 依赖的docker镜像
#image: 

# 依赖的docker服务
#services:

# 定义工作场景，和场景顺序
stages:
  - build
  - push
  - deploy
  
# 定义任务，任务名字随意
build_image:
  # 定义所属场景,与stages相对于
  stage: build
  # 定义任务脚本
  script: 
    - mvn clean install
    - docker build -t register.ptl-harbor.com/demo/test-prd:0.0.$CI_PIPELINE_ID .
  # 指定runner，tags为runner标签
  tags:
    - node1
  # 指定分支
  only:
    - master
    - schedules
  # artifacts 配置的paths文件可以在gitlab服务器上下载
  artifacts:
    paths:
      - target/*.jar
    # 打包文件的压缩包重命名
    name: test-prd-$CI_PIPELINE_ID
    when: on_success
    # 过期时间
    expire_in: 1 days

push_images:
  stage: push
  script:
    - docker push register.ptl-harbor.com/demo/test-prd:0.0.$CI_PIPELINE_ID
    - docker rmi register.ptl-harbor.com/demo/test-prd:0.0.$CI_PIPELINE_ID
  tags:
    - node1
```

## 4.推送并构建
![picture7](https://github.com/13535048320/gitlab-runner-demo/blob/master/images/%E5%9B%BE%E7%89%877.png)

## 5.高级用法
### 5.1定时构建
![picture8](https://github.com/13535048320/gitlab-runner-demo/blob/master/images/%E5%9B%BE%E7%89%878.png)
![picture9](https://github.com/13535048320/gitlab-runner-demo/blob/master/images/%E5%9B%BE%E7%89%879.png)
```
  job:
    only:
      - schedules # 定时构建触发
```

### 5.2非master的其他分支构建
```
  job:
    only:
      - branches@gitlab-org/gitlab-ce
    except:
      - master@gitlab-org/gitlab-ce
```

### 5.3Gitlab Ci自带变量
Variable | GitLab | Runner | Description
:--:|:--:|:--:|:--
CI | all | 0.4 | 标识该job是在CI环境中执行
CI_COMMIT_REF_NAME | 9.0 | all | 用于构建项目的分支或tag名称
CI_COMMIT_REF_SLUG | 9.0 | all | 先将$CI_COMMIT_REF_NAME的值转换成小写，最大不能超过63个字节，然后把除了0-9和a-z的其他字符转换成-。在URLs和域名名称中使用。
CI_COMMIT_SHA | 9.0 | all | commit的版本号
CI_COMMIT_TAG | 9.0 | 0.5 | commit的tag名称。只有创建了tags才会出现。
CI_DEBUG_TRACE | 9.0 | 1.7 | debug tracing开启时才生效
CI_ENVIRONMENT_NAME | 8.15 | all | job的环境名称
CI_ENVIRONMENT_SLUG | 8.15 | all | 环境名称的简化版本，适用于DNS，URLs，Kubernetes labels等
CI_JOB_ID | 9.0 | all | GItLab CI内部调用job的一个唯一ID
CI_JOB_MANUAL | 8.12 | all | 表示job启用的标识
CI_JOB_NAME | 9.0 | 0.5 | .gitlab-ci.yml中定义的job的名称
CI_JOB_STAGE | 9.0 | 0.5 | .gitlab-ci.yml中定义的stage的名称
CI_JOB_TOKEN | 9.0 | 1.2 | 用于同GitLab容器仓库验证的token
CI_REPOSITORY_URL | 9.0 | all | git仓库地址，用于克隆
CI_RUNNER_DESCRIPTION | 8.10 | 0.5 | GitLab中存储的Runner描述
CI_RUNNER_ID | 8.10 | 0.5 | Runner所使用的唯一ID
CI_RUNNER_TAGS | 8.10 | 0.5 | Runner定义的tags
CI_PIPELINE_ID | 8.10 | 0.5 | GitLab CI 在内部使用的当前pipeline的唯一ID
CI_PIPELINE_TRIGGERED | all | all | 用于指示该job被触发的标识
CI_PROJECT_DIR | all | all | 仓库克隆的完整地址和job允许的完整地址
CI_PROJECT_ID | all | all | GitLab CI在内部使用的当前项目的唯一ID
CI_PROJECT_NAME | 8.10 | 0.5 | 当前正在构建的项目名称（事实上是项目文件夹名称）
CI_PROJECT_NAMESPACE | 8.10 | 0.5 | 当前正在构建的项目命名空间（用户名或者是组名称）
CI_PROJECT_PATH | 8.10 | 0.5 | 命名空间加项目名称
CI_PROJECT_PATH_SLUG | 9.3 | all | $CI_PROJECT_PATH小写字母、除了0-9和a-z的其他字母都替换成-。用于地址和域名名称。
CI_PROJECT_URL | 8.10 | 0.5 | 项目的访问地址（http形式）
CI_REGISTRY | 8.10 | 0.5 | 如果启用了Container Registry，则返回GitLab的Container Registry的地址
CI_REGISTRY_IMAGE | 8.10 | 0.5 | 如果为项目启用了Container Registry，它将返回与特定项目相关联的注册表的地址
CI_REGISTRY_PASSWORD | 9.0 | all | 用于push containers到GitLab的Container Registry的密码
CI_REGISTRY_USER | 9.0 | all | 用于push containers到GItLab的Container Registry的用户名
CI_SERVER | all | all | 标记该job是在CI环境中执行
CI_SERVER_NAME | all | all | 用于协调job的CI服务器名称
CI_SERVER_REVISION | all | all | 用于调度job的GitLab修订版
CI_SERVER_VERSION | all | all | 用于调度job的GItLab版本
ARTIFACT_DOWNLOAD_ATTEMPTS | 8.15 | 1.9 | 尝试运行下载artifacts的job的次数
GET_SOURCES_ATTEMPTS | 8.15 | 1.9 | 尝试运行获取源的job次数
GITLAB_CI | all | all | 用于指示该job是在GItLab CI环境中运行
GITLAB_USER_ID | 8.12 | all | 开启该job的用户ID
GITLAB_USER_EMAIL | 8.12 | all | 开启该job的用户邮箱
RESTORE_CACHE_ATTEMPTS | 8.15 | 1.9 | 尝试运行存储缓存的job的次数