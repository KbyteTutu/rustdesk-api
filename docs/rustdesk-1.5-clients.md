# RustDesk 1.4.8+ / 1.5.x 客户端登录后无法连接的适配说明

## 现象

RustDesk 客户端 **1.4.8 及以上（含 1.5.x）** 登录 API 账号后，所有连接一直停留在「连接中」，
最终失败并提示：

```text
Failed to secure tcp: deadline has elapsed
```

而**退出账号登录**后可以正常连接（参考 [#524](https://github.com/lejianwen/rustdesk-api/issues/524)、
[#514](https://github.com/lejianwen/rustdesk-api/issues/514)）。

## 根因

客户端登录后携带 `access_token`，此时 `client.rs` 中的 `legacy_secure` 为真：
TCP 连上 hbbs（21116 端口）后，客户端**先等待服务器发起 rendezvous key exchange**，
握手成功才会把 token 加密发出。

开源版 rustdesk-server（hbbs，含最新 1.1.16）**从不发起该交换**——这是 rustdesk-server-pro
独有的功能。于是客户端等待 `READ_TIMEOUT`（18 秒）后超时，**主动放弃连接以保护 token 不明文发送**。

> 官方客户端曾有三个修复 PR（rustdesk#15743 / #15809 / #16700，改为容忍不发握手的
> 服务器），全部被官方关闭未合并——官方立场是要求服务器端实现握手。

**本 API 服务器不在打洞/中继协议路径上，无法在 API 侧修复此问题**，必须让 hbbs
学会发起 key exchange。

## 解决方案：使用打了补丁的 rustdesk-server

社区已在 rustdesk-server 上实现了服务端握手（v0 交换，兼容 1.4.x 和 1.5.x 客户端，
不影响未登录/不握手的老客户端）：

- [rustdesk-server#745](https://github.com/rustdesk/rustdesk-server/pull/745)（推荐，
  最小实现 +179/-20，持续维护，已实测 1.5.0/1.5.1/1.4.9 登录态连接成功）
- [rustdesk-server#744](https://github.com/rustdesk/rustdesk-server/pull/744)
- [rustdesk-server#734](https://github.com/rustdesk/rustdesk-server/pull/734)（含 IPv6/WebRTC/UDP 打洞）
- [rustdesk-server#706](https://github.com/rustdesk/rustdesk-server/pull/706)

> 注意：`lejianwen/rustdesk-server`（server-s6 镜像）基于 2025-09 的旧代码，
> **不包含**此修复，其「已解决链接超时」指旧的 websocket 问题（#92）。

### 第一步：构建打了补丁的 rustdesk-server-s6 镜像

```bash
git clone --recurse-submodules https://github.com/Lyapnev/rustdesk-server \
  -b feat/hbbs-kx-v1
cd rustdesk-server

# 构建 hbbs/hbbr（以 amd64 为例）
rustup target add x86_64-unknown-linux-musl
cargo build --release --target x86_64-unknown-linux-musl

# 组装 s6 镜像
cp target/x86_64-unknown-linux-musl/release/hbbs \
   target/x86_64-unknown-linux-musl/release/hbbr \
   target/x86_64-unknown-linux-musl/release/rustdesk-utils \
   docker/rootfs/usr/bin/
docker build -f docker/Dockerfile --build-arg S6_ARCH=x86_64 \
  -t <你的镜像仓库>/rustdesk-server-s6:kx docker/
docker push <你的镜像仓库>/rustdesk-server-s6:kx
```

其他架构把 target 换成对应的 musl 目标（arm64 为 `aarch64-unknown-linux-musl` /
`S6_ARCH=aarch64`，armv7 为 `armv7-unknown-linux-musleabihf` / `S6_ARCH=armhf`）。

### 第二步（full-s6 部署）：用补丁镜像构建 full-s6

本仓库的 `Dockerfile_full_s6` 已支持通过构建参数替换底层的 rustdesk-server-s6 镜像：

```bash
# 先构建 API 二进制
./build.sh
mkdir -p amd64 && cp -r release amd64/

docker buildx build -f Dockerfile_full_s6 \
  --build-arg BUILDARCH=amd64 \
  --build-arg RUSTDESK_SERVER_IMAGE=<你的镜像仓库>/rustdesk-server-s6:kx \
  -t <你的镜像仓库>/rustdesk-api:full-s6-kx .
```

或直接在 GitHub Actions 手动触发 **Build** 工作流，填写输入参数：

| 参数 | 值 |
| --- | --- |
| `RUSTDESK_SERVER_IMAGE` | `<你的镜像仓库>/rustdesk-server-s6:kx` |
| `SKIP_DOCKER_HUB` / `SKIP_GHCR` | 按需 |

> 同时修复了上游 build.yml 中 GHCR full-s6 步骤误用 `Dockerfile` 的问题。

### 第二步（分体部署）

如果 hbbs/hbbr 与 API 分开部署，直接用第一步构建的镜像（或其中的二进制）替换官方
`rustdesk/rustdesk-server(-s6)` 即可，API 侧无需任何改动。

## 验证

补丁生效后，hbbs 日志中会出现：

```text
key exchange ok from <client-addr>
```

客户端日志出现 `Connection secured`，登录状态下可以正常连接。

## 临时规避（不换服务器时）

- 退出账号登录（失去地址簿同步等功能）；
- 或让客户端走 wss（反向代理 + WebSocket），`secure_tcp` 对 ws 传输直接跳过。
