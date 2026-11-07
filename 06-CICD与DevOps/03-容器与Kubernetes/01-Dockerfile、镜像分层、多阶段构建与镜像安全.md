# 01. Dockerfile、镜像分层、多阶段构建与镜像安全

## 多阶段构建

Build Stage 包含 Compiler/Dependency，Runtime Stage 只复制产物：

```dockerfile
FROM eclipse-temurin:21-jdk AS build
WORKDIR /src
COPY . .
RUN ./gradlew bootJar

FROM eclipse-temurin:21-jre
COPY --from=build /src/build/libs/app.jar /app.jar
USER 10001
ENTRYPOINT ["java","-jar","/app.jar"]
```

减少 Size 和 Attack Surface。

## Layer

把变化少的 Dependency 与变化频繁的 Source 分层，提高 Cache Hit。使用 `.dockerignore` 排除 `.git`、Secret 和 Build 垃圾。

## 可重复性

固定 Base Image Digest、Dependency Lock 和 Tool Version。Tag 可读，部署使用 Digest。镜像不包含环境配置。

## 安全

- Non-root User。
- 最小 Base Image。
- 无 Package Manager/Shell（可行时）。
- Read-only Root FS。
- 不在 Layer 写 Secret。
- Vulnerability Scan、SBOM、签名。
- 定期 Rebuild 获取 Patch。

## Java

设置容器感知 Heap/Headroom，使用 Exec Form 让 Signal 正确传递，配置 Graceful Shutdown。不要把 `Xmx` 等同容器全部 Memory。

## 总结

镜像是不可变运行制品。多阶段、固定来源、最小权限和供应链证据共同提高性能、安全与可追溯性。
