FROM openjdk:11-jdk-slim as builder

# 1. 编译部分
# gradle start
ENV GRADLE_VER=6.1.1
RUN apt-get update -qqy \
  && apt-get install -qqy \
   --no-install-recommends \
   curl \
   ca-certificates \
   unzip

RUN mkdir -p /opt/ && cd /opt/ \
    && echo "Downloading gradle.." \
    && curl -sSfL "https://services.gradle.org/distributions/gradle-${GRADLE_VER}-bin.zip" -o gradle-$GRADLE_VER-bin.zip \
    && unzip gradle-$GRADLE_VER-bin.zip -d /opt/ \
    && rm gradle-$GRADLE_VER-bin.zip

# Export some environment variables
ENV GRADLE_HOME=/opt/gradle-$GRADLE_VER/
ENV PATH=$PATH:$GRADLE_HOME/bin

RUN mkdir -p /home/app/libs

ENV GRADLE_OPTS="-Dorg.gradle.daemon=false"
WORKDIR /home/app

COPY . /home/app/

RUN gradle build
RUN find . 
# gradle end

# 2. 反向代理部分
# of-watchdog start
# https://github.com/openfaas/of-watchdog
# HTTP 微服务和 STDIO 的反向代理：该 of-watchdog 工具实现了一个侦听端口 8080 的 HTTP 服务器，并充当运行功能和微服务的反向代理。它可以单独使用，也可以用作带有 OpenFaaS 的容器的入口点。
FROM openfaas/of-watchdog:0.7.6 as watchdog

FROM openjdk:11-jre-slim as ship
RUN apt-get update -qqy \
  && apt-get install -qqy \
   --no-install-recommends \
   unzip
RUN addgroup --system app \
    && adduser --system --ingroup app app

COPY --from=watchdog /fwatchdog /usr/bin/fwatchdog
RUN chmod +x /usr/bin/fwatchdog

WORKDIR /home/app
COPY --from=builder /home/app/function/build/distributions/function-1.0.zip ./function-1.0.zip
user app
RUN unzip ./function-1.0.zip
# of-watchdog end

# 3. 运行部分
WORKDIR /home/app/

ENV upstream_url="http://127.0.0.1:8082"
ENV mode="http"
ENV CLASSPATH="/home/app/function-1.0/function-1.0.jar:/home/app/function-1.0/lib/*"

# set app and expose port
ENV fprocess="java -XX:+UseContainerSupport com.openfaas.entrypoint.App"
EXPOSE 8080

# 4. 心跳部分
# healthcheck
HEALTHCHECK --interval=5s CMD [ -e /tmp/.lock ] || exit 1

CMD ["fwatchdog"]
