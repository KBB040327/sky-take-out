# ============================================================
# 苍穹外卖 —— 应用镜像
# 说明：jar 包在宿主机上用 Maven 打好后复制过来，这里只负责运行
# ============================================================
FROM eclipse-temurin:17-jre-jammy

LABEL maintainer="koubinbin" \
      description="sky-take-out backend"

# 时区 + 编码 + JVM 参数
# ⚠️ -Xmx 必须限制：VM 只有 3.3G 内存，不限制会被 OOM Killer 干掉
ENV TZ=Asia/Shanghai \
    LANG=C.UTF-8 \
    JAVA_OPTS="-Xms256m -Xmx384m -XX:+UseSerialGC -XX:MaxMetaspaceSize=128m -Duser.timezone=Asia/Shanghai -Dfile.encoding=UTF-8"

WORKDIR /app

# 复制 jar（构建前由部署脚本从 target 目录拷成 sky-server.jar）
COPY sky-server.jar /app/app.jar

EXPOSE 8080

# 健康检查：用 bash 的 /dev/tcp 探端口，避免额外装 curl
HEALTHCHECK --interval=20s --timeout=5s --start-period=90s --retries=6 \
  CMD bash -c 'exec 3<>/dev/tcp/127.0.0.1/8080' || exit 1

ENTRYPOINT ["sh", "-c", "exec java $JAVA_OPTS -jar /app/app.jar"]
