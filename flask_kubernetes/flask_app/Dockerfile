#镜像定义
FROM python27_baseenv

# 作者声明
MAINTAINER guomaoqiu<guomaoqiu@sina.com>

#PORT
#EXPOSE 5000

# 应用工作目录
WORKDIR /usr/local/code/

# 拷贝
COPY ./requirements.txt $WORKDIR
 
# 安装应用第三方依赖软件
RUN pip install --no-cache-dir -i https://pypi.douban.com/simple -r requirements.txt

