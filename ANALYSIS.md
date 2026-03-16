# NGINX 工程深度分析文档

> 本文档对 NGINX 开源项目进行全面的模块分析、整体运行流程梳理及二次开发注意事项说明。

---

## 目录

1. [项目概述](#1-项目概述)
2. [工程目录结构](#2-工程目录结构)
3. [模块体系详解](#3-模块体系详解)
   - 3.1 [模块注册机制](#31-模块注册机制)
   - 3.2 [Core 核心模块](#32-core-核心模块)
   - 3.3 [Event 事件模块](#33-event-事件模块)
   - 3.4 [HTTP 模块](#34-http-模块)
   - 3.5 [Stream 模块](#35-stream-模块)
   - 3.6 [Mail 模块](#36-mail-模块)
   - 3.7 [OS 抽象层](#37-os-抽象层)
4. [整体运行流程](#4-整体运行流程)
   - 4.1 [进程模型](#41-进程模型)
   - 4.2 [启动流程](#42-启动流程)
   - 4.3 [配置解析流程](#43-配置解析流程)
   - 4.4 [事件循环](#44-事件循环)
   - 4.5 [HTTP 请求处理流程](#45-http-请求处理流程)
   - 4.6 [信号处理与热重载](#46-信号处理与热重载)
5. [二次开发注意事项](#5-二次开发注意事项)
   - 5.1 [自定义模块开发流程](#51-自定义模块开发流程)
   - 5.2 [模块结构详解](#52-模块结构详解)
   - 5.3 [内存管理规范](#53-内存管理规范)
   - 5.4 [HTTP 处理阶段挂载](#54-http-处理阶段挂载)
   - 5.5 [过滤器链开发](#55-过滤器链开发)
   - 5.6 [Upstream 模块开发](#56-upstream-模块开发)
   - 5.7 [动态模块开发](#57-动态模块开发)
   - 5.8 [常见陷阱与最佳实践](#58-常见陷阱与最佳实践)
6. [关键数据结构速查](#6-关键数据结构速查)

---

## 1. 项目概述

NGINX（读作 "engine x"）是世界上使用最广泛的 Web 服务器，同时也是高性能的反向代理、负载均衡器、API 网关和内容缓存。

| 属性 | 说明 |
|------|------|
| 许可证 | BSD 2-Clause |
| 语言 | C（少量汇编） |
| 架构 | 事件驱动、异步非阻塞、Master-Worker 多进程 |
| 支持协议 | HTTP/1.0、HTTP/1.1、HTTP/2、HTTP/3(QUIC)、TCP/UDP(Stream)、IMAP、POP3、SMTP |
| 平台支持 | Linux、FreeBSD、macOS、Solaris、Windows（PoC） |
| 源文件数量 | 约 395 个 C/H 文件 |

**设计哲学：**

- **异步非阻塞**：基于事件驱动（epoll/kqueue），单 Worker 进程可处理数万并发连接。
- **模块化**：所有功能以模块形式组织，核心极简，功能通过模块按需扩展。
- **零拷贝**：大量使用 `sendfile`、链式缓冲区（`ngx_chain_t`）减少内存复制。
- **内存池**：使用内存池（`ngx_pool_t`）统一管理连接/请求生命周期内的内存，避免碎片。

---

## 2. 工程目录结构

```
nginx/
├── auto/                   # 构建配置脚本（configure 系统）
│   ├── configure           # 主配置入口脚本
│   ├── options             # 编译选项定义
│   ├── modules             # 模块选择逻辑
│   ├── sources             # 源文件列表
│   ├── make                # Makefile 生成
│   ├── cc/                 # 编译器（GCC/Clang/MSVC）特定配置
│   ├── lib/                # 依赖库（OpenSSL、PCRE、zlib）检测
│   └── os/                 # 操作系统特定构建配置
│
├── conf/                   # 默认配置文件模板
│   ├── nginx.conf          # 主配置文件示例
│   ├── mime.types          # MIME 类型映射
│   ├── fastcgi*.conf       # FastCGI 参数
│   └── *_params            # SCGI/uWSGI 参数
│
├── src/                    # 源代码（核心）
│   ├── core/               # 核心框架（内存池、日志、模块、数据结构等）
│   ├── event/              # 事件引擎（epoll、kqueue、QUIC 等）
│   │   ├── modules/        # 平台事件模块（epoll、kqueue、select 等）
│   │   └── quic/           # QUIC/HTTP3 实现
│   ├── http/               # HTTP 协议实现
│   │   ├── modules/        # HTTP 功能模块（60+ 个）
│   │   ├── v2/             # HTTP/2 实现
│   │   └── v3/             # HTTP/3 实现
│   ├── stream/             # TCP/UDP 流代理模块
│   ├── mail/               # 邮件代理模块（IMAP/POP3/SMTP）
│   ├── os/                 # 操作系统抽象层
│   │   ├── unix/           # Unix/Linux/macOS 实现（35 个文件）
│   │   └── win32/          # Windows 实现（21 个文件）
│   └── misc/               # 杂项（Google perftools 模块等）
│
├── docs/                   # 文档源文件
├── contrib/                # 社区贡献（VIM 语法高亮等）
├── misc/                   # 杂项脚本
└── README.md               # 项目说明
```

---

## 3. 模块体系详解

### 3.1 模块注册机制

NGINX 所有功能单元均以**模块**形式存在，通过统一的 `ngx_module_t` 结构体描述。

#### `ngx_module_t` 核心结构（`src/core/ngx_module.h`）

```c
struct ngx_module_s {
    ngx_uint_t            ctx_index;    // 同类型模块的序号
    ngx_uint_t            index;        // 所有模块的全局序号
    char                 *name;         // 模块名称
    ngx_uint_t            version;      // 模块版本
    const char           *signature;    // 编译特征签名（用于动态模块兼容性校验）

    void                 *ctx;          // 模块上下文（依模块类型而定）
    ngx_command_t        *commands;     // 该模块支持的配置指令数组
    ngx_uint_t            type;         // 模块类型（CORE/EVENT/HTTP/STREAM/MAIL）

    // 生命周期回调
    ngx_int_t           (*init_master)(ngx_log_t *log);
    ngx_int_t           (*init_module)(ngx_cycle_t *cycle);
    ngx_int_t           (*init_process)(ngx_cycle_t *cycle);
    ngx_int_t           (*init_thread)(ngx_cycle_t *cycle);
    void                (*exit_thread)(ngx_cycle_t *cycle);
    void                (*exit_process)(ngx_cycle_t *cycle);
    void                (*exit_master)(ngx_cycle_t *cycle);
};
```

#### 模块类型

| 类型常量 | 说明 |
|---------|------|
| `NGX_CORE_MODULE` | 核心模块（core、events、http、stream、mail 顶层块） |
| `NGX_EVENT_MODULE` | 事件模块（epoll、kqueue 等） |
| `NGX_HTTP_MODULE` | HTTP 功能模块 |
| `NGX_STREAM_MODULE` | Stream 功能模块 |
| `NGX_MAIL_MODULE` | Mail 功能模块 |

#### 模块索引初始化

所有静态模块在编译时由 `auto/configure` 脚本生成到 `objs/ngx_modules.c`，以 `ngx_modules[]` 数组存储；启动时依次调用 `ngx_preinit_modules()` → `ngx_init_modules()` 完成注册和初始化。

---

### 3.2 Core 核心模块

位置：`src/core/`，约 39 个文件，是整个 NGINX 的基础框架。

#### 关键组件一览

| 文件 | 功能 |
|------|------|
| `nginx.c` | 程序入口 `main()`，进程初始化，启动 Master/Worker |
| `ngx_cycle.c/h` | `ngx_cycle_t`：运行时全局状态，连接池、监听套接字、模块配置 |
| `ngx_conf_file.c/h` | 配置文件词法/语法解析，指令分发 |
| `ngx_module.c/h` | 模块注册、初始化、动态加载（`load_module`） |
| `ngx_connection.c/h` | 连接池管理，`ngx_connection_t` 生命周期 |
| `ngx_palloc.c/h` | **内存池**，`ngx_pool_t`、`ngx_palloc()`/`ngx_pcalloc()` |
| `ngx_buf.c/h` | **缓冲区**，`ngx_buf_t`、链式缓冲区 `ngx_chain_t` |
| `ngx_log.c/h` | 日志框架，多级别（debug/info/notice/warn/error/crit） |
| `ngx_hash.c/h` | 哈希表（含通配符哈希），用于 server_name、变量等快速查找 |
| `ngx_rbtree.c/h` | 红黑树，用于定时器管理、cache 等需要有序集合的场景 |
| `ngx_queue.c/h` | 双向循环队列 |
| `ngx_array.c/h` | 动态数组 |
| `ngx_list.c/h` | 单向链表 |
| `ngx_radix_tree.c/h` | 基数树，IP 段匹配（`geo` 模块使用） |
| `ngx_string.c/h` | `ngx_str_t`（`{data, len}` 非 NUL 终止字符串）及操作函数 |
| `ngx_resolver.c/h` | 异步 DNS 解析器 |
| `ngx_slab.c/h` | 共享内存 slab 分配器 |
| `ngx_open_file_cache.c/h` | 文件描述符/元信息缓存 |
| `ngx_proxy_protocol.c/h` | PROXY Protocol（获取真实客户端 IP） |
| `ngx_thread_pool.c/h` | 线程池（用于磁盘 I/O 卸载到线程） |
| `ngx_times.c/h` | 时间缓存（每次 epoll 循环更新，避免频繁 `gettimeofday`） |
| `ngx_event_openssl.c/h` | （实际在 `src/event/`）OpenSSL 集成 |
| `ngx_crc32/md5/sha1/murmurhash` | 哈希算法实现 |

#### 核心数据结构关系

```
ngx_cycle_t
├── pool           ←  ngx_pool_t（内存池）
├── connections[]  ←  ngx_connection_t 数组（预分配连接池）
├── free_connections  ←  空闲连接链表
├── listening      ←  ngx_listening_t 数组（监听套接字）
├── conf_ctx[]     ←  各顶层模块配置指针
└── log            ←  ngx_log_t
```

---

### 3.3 Event 事件模块

位置：`src/event/`，负责**异步 I/O 多路复用**，是 NGINX 高并发的核心。

#### 子目录结构

```
src/event/
├── ngx_event.c/h               # 事件核心，ngx_event_t、ngx_event_actions_t
├── ngx_event_timer.c/h         # 定时器（基于红黑树）
├── ngx_event_posted.c/h        # Posted 事件队列（延迟执行）
├── ngx_event_accept.c/h        # 新连接接受
├── ngx_event_connect.c/h       # 主动连接（upstream）
├── ngx_event_pipe.c/h          # 上游/下游缓冲管道
├── ngx_event_openssl.c/h       # OpenSSL/TLS 集成
├── ngx_event_openssl_cache.c   # SSL Session 缓存
├── ngx_event_openssl_stapling.c# OCSP Stapling
├── ngx_event_udp.c/h           # UDP 事件处理
├── modules/                    # 平台事件后端
│   ├── ngx_epoll_module.c      # Linux epoll（主流生产选择）
│   ├── ngx_kqueue_module.c     # BSD/macOS kqueue
│   ├── ngx_select_module.c     # POSIX select（备用）
│   ├── ngx_poll_module.c       # POSIX poll（备用）
│   ├── ngx_devpoll_module.c    # Solaris /dev/poll
│   ├── ngx_eventport_module.c  # Solaris Event Ports
│   └── ngx_iocp_module.c       # Windows IOCP
└── quic/                       # QUIC 协议栈（HTTP/3 基础）
    ├── ngx_event_quic.c        # QUIC 核心
    ├── ngx_event_quic_ack.c    # ACK 处理
    ├── ngx_event_quic_frames.c # 帧处理
    ├── ngx_event_quic_transport.c # 传输层
    ├── ngx_event_quic_streams.c   # 流管理
    ├── ngx_event_quic_ssl.c       # QUIC TLS 1.3
    ├── ngx_event_quic_protection.c# 包保护（加密/解密）
    └── ...（共 18 个文件）
```

#### 事件抽象接口 `ngx_event_actions_t`

```c
// 所有事件后端（epoll/kqueue 等）必须实现的接口
typedef struct {
    ngx_int_t  (*add)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);
    ngx_int_t  (*del)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);
    ngx_int_t  (*enable)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);
    ngx_int_t  (*disable)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);
    ngx_int_t  (*add_conn)(ngx_connection_t *c);
    ngx_int_t  (*del_conn)(ngx_connection_t *c, ngx_uint_t flags);
    ngx_int_t  (*notify)(ngx_event_handler_pt handler);
    ngx_int_t  (*process_events)(ngx_cycle_t *cycle, ngx_msec_t timer,
                                  ngx_uint_t flags);
    ngx_int_t  (*init)(ngx_cycle_t *cycle, ngx_msec_t timer);
    void       (*done)(ngx_cycle_t *cycle);
} ngx_event_actions_t;
```

#### 定时器实现

NGINX 使用**红黑树**（`ngx_rbtree_t`）管理定时器，key 为过期时间（毫秒）。每次 `process_events` 时传入最近到期时间作为 epoll_wait 超时，兼顾精度与效率。

#### QUIC/HTTP3 架构要点

- QUIC 使用 UDP 套接字，在 `ngx_event_udp.c` 基础上实现多路复用。
- 连接 ID（Connection ID）管理由 `ngx_event_quic_connid.c` 负责，支持连接迁移。
- 与 OpenSSL/BoringSSL 通过 `ngx_event_quic_ssl.c` 集成 TLS 1.3。

---

### 3.4 HTTP 模块

位置：`src/http/`，是 NGINX 最复杂的子系统，包含核心框架、60+ 功能模块、HTTP/2 和 HTTP/3 实现。

#### HTTP 模块上下文结构 `ngx_http_module_t`（`src/http/ngx_http_config.h`）

```c
typedef struct {
    ngx_int_t   (*preconfiguration)(ngx_conf_t *cf);   // 配置解析前
    ngx_int_t   (*postconfiguration)(ngx_conf_t *cf);  // 配置解析后（注册 handler/filter）

    void       *(*create_main_conf)(ngx_conf_t *cf);   // 创建 http{} 级配置
    char       *(*init_main_conf)(ngx_conf_t *cf, void *conf);

    void       *(*create_srv_conf)(ngx_conf_t *cf);    // 创建 server{} 级配置
    char       *(*merge_srv_conf)(ngx_conf_t *cf, void *prev, void *conf);

    void       *(*create_loc_conf)(ngx_conf_t *cf);    // 创建 location{} 级配置
    char       *(*merge_loc_conf)(ngx_conf_t *cf, void *prev, void *conf);
} ngx_http_module_t;
```

#### HTTP 请求处理阶段（`ngx_http_phases` 枚举）

NGINX 将请求处理分为 **11 个阶段**，模块可在各阶段挂载 handler：

| 序号 | 阶段枚举 | 说明 |
|------|---------|------|
| 0 | `NGX_HTTP_POST_READ_PHASE` | 读取请求头后（`realip` 模块在此阶段工作） |
| 1 | `NGX_HTTP_SERVER_REWRITE_PHASE` | server{} 级别 rewrite |
| 2 | `NGX_HTTP_FIND_CONFIG_PHASE` | 查找匹配的 location（框架内部，不可挂载） |
| 3 | `NGX_HTTP_REWRITE_PHASE` | location{} 级别 rewrite |
| 4 | `NGX_HTTP_POST_REWRITE_PHASE` | rewrite 后处理（框架内部） |
| 5 | `NGX_HTTP_PREACCESS_PHASE` | 访问控制前（`limit_req`、`limit_conn` 在此） |
| 6 | `NGX_HTTP_ACCESS_PHASE` | 访问控制（`access`、`auth_basic` 在此） |
| 7 | `NGX_HTTP_POST_ACCESS_PHASE` | access 后处理（框架内部） |
| 8 | `NGX_HTTP_PRECONTENT_PHASE` | 内容生成前（`try_files`、`mirror` 在此） |
| 9 | `NGX_HTTP_CONTENT_PHASE` | **内容生成**（`proxy`、`static`、`fastcgi` 等在此） |
| 10 | `NGX_HTTP_LOG_PHASE` | 请求日志记录（`access_log` 在此） |

#### HTTP 过滤器链

内容生成后，响应经过**过滤器链**（filter chain）依次处理，两条链：

- **响应头过滤器链**：`ngx_http_top_header_filter` → ... → `ngx_http_header_filter`
- **响应体过滤器链**：`ngx_http_top_body_filter` → ... → `ngx_http_write_filter`

常见内置过滤器（按执行顺序，从最后注册到最先注册）：

```
write_filter           ← 最终写入到客户端
chunked_filter         ← Chunked 编码
range_body_filter      ← Range 请求
range_header_filter    ← Range 请求头
not_modified_filter    ← 304 Not Modified
slice_filter           ← 大文件分片
copy_filter            ← 缓冲区复制/sendfile
headers_filter         ← 添加/修改响应头
gzip_filter            ← GZIP 压缩
sub_filter             ← 文本替换
ssi_filter             ← SSI 处理
charset_filter         ← 字符集转换
addition_filter        ← 内容追加
xslt_filter            ← XSLT 转换
image_filter           ← 图片裁剪/缩放
```

#### HTTP 核心文件说明

| 文件 | 功能 |
|------|------|
| `ngx_http.c` | HTTP 模块初始化，配置树构建，阶段引擎初始化 |
| `ngx_http_core_module.c` | 核心指令（listen、server_name、location 等） |
| `ngx_http_request.c/h` | 请求解析、阶段调度（`ngx_http_process_request_line` 等） |
| `ngx_http_request_body.c` | 请求体读取，支持缓冲/磁盘 |
| `ngx_http_parse.c` | HTTP/1.x 请求行/请求头解析状态机 |
| `ngx_http_upstream.c/h` | 上游连接管理，请求转发，响应接收 |
| `ngx_http_upstream_round_robin.c` | 轮询负载均衡基础实现 |
| `ngx_http_variables.c` | 内置变量（`$uri`、`$host`、`$remote_addr` 等） |
| `ngx_http_script.c` | 变量替换脚本引擎（rewrite/proxy_pass 等） |
| `ngx_http_file_cache.c` | 缓存存储/命中/过期逻辑 |
| `ngx_http_special_response.c` | 错误页面生成 |

#### HTTP/2（`src/http/v2/`）

- 实现 HPACK 压缩（`ngx_http_huff_encode/decode.c`）
- 多路复用（stream 管理）、流控制
- 支持作为客户端（proxy_http_version 2）和服务端

#### HTTP/3（`src/http/v3/`）

- 基于 QUIC 传输层（`src/event/quic/`）
- QPACK 头部压缩（`ngx_http_v3_table.c`）
- 单向流（控制流、推送流、QPACK 动态表）

#### HTTP 功能模块速查

**代理与上游：**

| 模块文件 | 功能 |
|---------|------|
| `ngx_http_proxy_module.c` | HTTP 反向代理（最常用） |
| `ngx_http_proxy_v2_module.c` | HTTP/2 代理 |
| `ngx_http_fastcgi_module.c` | FastCGI 协议（PHP-FPM） |
| `ngx_http_scgi_module.c` | SCGI 协议 |
| `ngx_http_uwsgi_module.c` | uWSGI 协议（Python） |
| `ngx_http_grpc_module.c` | gRPC 代理 |
| `ngx_http_memcached_module.c` | Memcached 读取 |
| `ngx_http_upstream_hash_module.c` | 哈希负载均衡（`hash` 指令） |
| `ngx_http_upstream_ip_hash_module.c` | IP 哈希负载均衡 |
| `ngx_http_upstream_least_conn_module.c` | 最少连接负载均衡 |
| `ngx_http_upstream_random_module.c` | 随机负载均衡 |
| `ngx_http_upstream_sticky_module.c` | 会话保持 |
| `ngx_http_upstream_keepalive_module.c` | 上游连接复用 |
| `ngx_http_upstream_zone_module.c` | 共享内存上游状态 |

**静态内容：**

| 模块文件 | 功能 |
|---------|------|
| `ngx_http_static_module.c` | 静态文件服务 |
| `ngx_http_index_module.c` | 目录索引文件 |
| `ngx_http_autoindex_module.c` | 目录列表 |
| `ngx_http_dav_module.c` | WebDAV 支持 |
| `ngx_http_mp4_module.c` | MP4 伪流式 |
| `ngx_http_flv_module.c` | FLV 伪流式 |

**安全与访问控制：**

| 模块文件 | 功能 |
|---------|------|
| `ngx_http_access_module.c` | IP 访问控制（allow/deny） |
| `ngx_http_auth_basic_module.c` | HTTP Basic 认证 |
| `ngx_http_auth_request_module.c` | 子请求认证委托 |
| `ngx_http_ssl_module.c` | SSL/TLS |
| `ngx_http_realip_module.c` | 真实客户端 IP 还原 |
| `ngx_http_referer_module.c` | Referer 防盗链 |
| `ngx_http_secure_link_module.c` | 签名 URL 验证 |

**限流：**

| 模块文件 | 功能 |
|---------|------|
| `ngx_http_limit_req_module.c` | 请求速率限制（漏桶算法） |
| `ngx_http_limit_conn_module.c` | 连接数限制 |

**其他：**

| 模块文件 | 功能 |
|---------|------|
| `ngx_http_rewrite_module.c` | URL 重写（rewrite/return/if） |
| `ngx_http_headers_filter_module.c` | 响应头操作（add_header 等） |
| `ngx_http_log_module.c` | 访问日志 |
| `ngx_http_stub_status_module.c` | 基础状态页（`/nginx_status`） |
| `ngx_http_gzip_filter_module.c` | GZIP 压缩 |
| `ngx_http_map_module.c` | 变量映射 |
| `ngx_http_geo_module.c` | IP 到变量映射 |
| `ngx_http_split_clients_module.c` | A/B 测试流量分配 |
| `ngx_http_mirror_module.c` | 请求镜像 |

---

### 3.5 Stream 模块

位置：`src/stream/`，约 26 个文件，提供 **TCP/UDP 四层代理与负载均衡**。

#### 架构特点

- 与 HTTP 模块架构类似，但无 HTTP 解析，直接转发 TCP/UDP 字节流。
- 支持 SSL/TLS 终止（`ngx_stream_ssl_module.c`）和 SSL 预读（`ngx_stream_ssl_preread_module.c`，用于 SNI 路由）。
- 处理阶段：`POST_ACCEPT` → `PREACCESS` → `ACCESS` → `SSL_PREREAD` → `CONTENT` → `LOG`

#### 主要模块

| 文件 | 功能 |
|------|------|
| `ngx_stream.c` | Stream 模块核心初始化 |
| `ngx_stream_core_module.c` | 核心指令（listen、server 等） |
| `ngx_stream_proxy_module.c` | TCP/UDP 代理 |
| `ngx_stream_upstream.c/h` | 上游管理 |
| `ngx_stream_ssl_module.c` | SSL/TLS |
| `ngx_stream_ssl_preread_module.c` | SNI/ALPN 预读，无需终止 SSL |
| `ngx_stream_access_module.c` | IP 访问控制 |
| `ngx_stream_limit_conn_module.c` | 连接限制 |
| `ngx_stream_log_module.c` | 访问日志 |
| `ngx_stream_return_module.c` | 直接返回数据 |
| `ngx_stream_map_module.c` | 变量映射 |
| `ngx_stream_geo_module.c` | GeoIP |
| `ngx_stream_realip_module.c` | 真实 IP |
| `ngx_stream_upstream_*_module.c` | 各种负载均衡算法 |

---

### 3.6 Mail 模块

位置：`src/mail/`，约 14 个文件，提供**邮件协议代理**。

#### 主要功能

- 支持 IMAP、POP3、SMTP 协议代理
- 客户端认证委托给外部 HTTP 认证服务（`ngx_mail_auth_http_module.c`）
- 支持 SSL/TLS（`ngx_mail_ssl_module.c`）
- 主要用途：企业邮件系统前端代理，实现认证、SSL 卸载

#### 主要文件

| 文件 | 功能 |
|------|------|
| `ngx_mail.c` | Mail 模块初始化 |
| `ngx_mail_core_module.c` | 核心指令（server、listen、protocol 等） |
| `ngx_mail_handler.c` | 连接处理、协议识别 |
| `ngx_mail_imap_module.c/handler.c` | IMAP 协议解析与处理 |
| `ngx_mail_pop3_module.c/handler.c` | POP3 协议解析与处理 |
| `ngx_mail_smtp_module.c/handler.c` | SMTP 协议解析与处理 |
| `ngx_mail_proxy_module.c` | 到后端邮件服务器代理 |
| `ngx_mail_auth_http_module.c` | HTTP 认证服务委托 |
| `ngx_mail_ssl_module.c` | SSL/TLS 支持 |
| `ngx_mail_realip_module.c` | 真实 IP |
| `ngx_mail_parse.c` | 协议解析工具函数 |

---

### 3.7 OS 抽象层

位置：`src/os/`，将系统调用封装为平台无关的接口。

#### Unix 层（`src/os/unix/`，35 个文件）

| 文件类别 | 文件 |
|---------|------|
| 内存 | `ngx_alloc.c`（`malloc`/`free` 包装） |
| 进程 | `ngx_process.c`、`ngx_process_cycle.c`（Master/Worker 循环） |
| 文件 I/O | `ngx_files.c`、`ngx_file_aio_read.c`（Linux AIO）、`ngx_linux_aio_read.c` |
| 网络 | `ngx_socket.c`、`ngx_recv.c`、`ngx_send.c`、`ngx_readv_chain.c`、`ngx_writev_chain.c` |
| sendfile | `ngx_linux_sendfile_chain.c`、`ngx_freebsd_sendfile_chain.c`、`ngx_darwin_sendfile_chain.c` |
| 共享内存 | `ngx_shmem.c`（mmap/shmget） |
| 时间 | `ngx_time.c` |
| 线程 | `ngx_thread.c`、`ngx_thread_cond.c`、`ngx_thread_mutex.c` |
| 用户 | `ngx_user.c`（setuid/setgid） |
| 守护进程 | `ngx_daemon.c` |
| IPC | `ngx_channel.c`（socketpair 进程间通信） |
| 初始化 | `ngx_linux_init.c`、`ngx_freebsd_init.c`、`ngx_darwin_init.c` 等 |

#### Windows 层（`src/os/win32/`，21 个文件）

对应功能的 Windows 实现，使用 Winsock、IOCP、Windows Service API 等。

---

## 4. 整体运行流程

### 4.1 进程模型

```
                  ┌─────────────────┐
                  │   Master Process │
                  │  (管理员进程)     │
                  │ - 读取配置文件   │
                  │ - 管理 Worker   │
                  │ - 处理信号      │
                  └────────┬────────┘
                           │ fork()
          ┌────────────────┼────────────────┐
          │                │                │
   ┌──────┴──────┐  ┌──────┴──────┐  ┌─────┴───────┐
   │ Worker[0]   │  │ Worker[1]   │  │ Worker[N-1] │
   │ (工作进程)  │  │ (工作进程)  │  │ (工作进程)  │
   │ - 接收连接  │  │ - 接收连接  │  │ - 接收连接  │
   │ - 处理请求  │  │ - 处理请求  │  │ - 处理请求  │
   │ - 事件循环  │  │ - 事件循环  │  │ - 事件循环  │
   └─────────────┘  └─────────────┘  └─────────────┘
          │                │                │
          └────────────────┴────────────────┘
                  共享内存（限流、缓存等）
```

- **Master 进程**：不处理任何请求，只管理 Worker 进程和配置。
- **Worker 进程**：每个 Worker 独立运行事件循环，通过 `accept_mutex`（或 `SO_REUSEPORT`）避免惊群。
- **Cache Manager/Loader 进程**：若启用文件缓存，额外 fork 用于缓存维护的进程。

### 4.2 启动流程

```
main()
  ├── ngx_strerror_init()         // 初始化错误信息
  ├── ngx_get_options()           // 解析命令行参数
  ├── ngx_time_init()             // 初始化时间缓存
  ├── ngx_regex_init()            // 初始化 PCRE
  ├── ngx_log_init()              // 初始化日志
  ├── ngx_ssl_init()              // 初始化 OpenSSL
  ├── ngx_os_init()               // OS 相关初始化（页大小、CPU 数等）
  ├── ngx_preinit_modules()       // 为所有模块分配索引
  ├── ngx_init_cycle()            // ★ 核心：解析配置、初始化所有模块、监听端口
  │     ├── 创建内存池
  │     ├── 遍历所有模块调用 create_conf()
  │     ├── 解析配置文件（ngx_conf_parse）
  │     ├── 遍历所有模块调用 init_conf()
  │     ├── 打开监听套接字
  │     └── 调用 init_module() 回调
  ├── ngx_init_signals()          // 注册信号处理器
  ├── ngx_daemon()                // 守护进程化（若配置了）
  └── ngx_master_process_cycle()  // ★ 进入 Master 循环
        └── ngx_start_worker_processes()
              └── fork() → ngx_worker_process_cycle()
                    ├── ngx_worker_process_init()
                    │     └── 遍历模块调用 init_process()
                    └── for(;;) ngx_process_events_and_timers()
                          ├── ngx_event_actions.process_events()  // epoll_wait
                          └── 处理 posted 事件队列
```

### 4.3 配置解析流程

```
配置文件
  └── ngx_conf_parse()
        ├── 词法分析（token 化）
        └── 遍历 token 找到对应模块的 ngx_command_t
              └── 调用 command->set() 回调存储配置值
                    └── 配置值写入对应模块的 conf 结构体
                          (main_conf / srv_conf / loc_conf)
```

**配置层级与继承：**

```
http {                          # main_conf
    server {                    # srv_conf（继承 main_conf）
        location / {            # loc_conf（继承 srv_conf → main_conf）
        }
    }
}
```

子级配置未设置时，通过 `merge_srv_conf()`/`merge_loc_conf()` 从父级继承。

### 4.4 事件循环

```
Worker 进程事件循环（每次迭代）：

ngx_process_events_and_timers(cycle)
  ├── 计算最近定时器到期时间 (ngx_event_find_timer)
  ├── epoll_wait(epfd, events, MAX_EVENTS, timeout)  // 等待 I/O 事件或超时
  ├── ngx_time_update()                               // 更新时间缓存
  ├── 遍历就绪事件，调用 event->handler()            // 如 ngx_http_wait_request_handler
  ├── ngx_event_expire_timers()                       // 处理超时定时器
  └── ngx_event_process_posted(cycle, &ngx_posted_events) // 处理 posted 事件
```

**事件 handler 回调链（HTTP 为例）：**

```
新连接到达
  → ngx_event_accept()
  → 分配 ngx_connection_t
  → c->read->handler = ngx_http_wait_request_handler

收到数据（可读事件）
  → ngx_http_wait_request_handler()
  → ngx_http_process_request_line()   // 解析请求行
  → ngx_http_process_request_headers() // 解析请求头
  → ngx_http_process_request()         // 调度到处理阶段
  → ngx_http_core_run_phases()         // 依次执行各阶段 handler
  → ngx_http_finalize_request()        // 发送响应、清理
```

### 4.5 HTTP 请求处理流程

```
客户端请求
  ↓
[网络层] 建立 TCP 连接
  ↓
[事件层] accept 新连接，分配 ngx_connection_t
  ↓
[HTTP 层] 读取并解析 HTTP 请求行/请求头
  ↓
[阶段 0] POST_READ          → realip 模块还原真实 IP
  ↓
[阶段 1] SERVER_REWRITE     → server 级 rewrite 规则
  ↓
[阶段 2] FIND_CONFIG        → 匹配 location（前缀/正则/精确）
  ↓
[阶段 3] REWRITE            → location 级 rewrite 规则
  ↓
[阶段 4] POST_REWRITE       → 若有重写则循环，否则继续
  ↓
[阶段 5] PREACCESS          → limit_req / limit_conn 限流检查
  ↓
[阶段 6] ACCESS             → auth_basic / access / auth_request 认证
  ↓
[阶段 7] POST_ACCESS        → satisfy any/all 处理
  ↓
[阶段 8] PRECONTENT         → try_files / mirror
  ↓
[阶段 9] CONTENT            → proxy/fastcgi/static 等内容生成 ←★ 核心
  ↓
响应通过过滤器链
  ↓
[阶段 10] LOG               → access_log 记录
  ↓
发送给客户端
```

### 4.6 信号处理与热重载

Master 进程通过信号与外部通信：

| 信号 | 效果 |
|------|------|
| `SIGHUP` | 热重载配置（不中断服务，优雅替换 Worker） |
| `SIGUSR1` | 重新打开日志文件（日志切割） |
| `SIGUSR2` | 升级可执行文件（不中断服务）第一步 |
| `SIGWINCH` | 优雅停止 Worker 进程 |
| `SIGTERM` | 快速停止 |
| `SIGQUIT` | 优雅停止 |

**热重载流程（`nginx -s reload`）：**

```
1. 向 Master 发送 SIGHUP
2. Master 解析新配置文件（ngx_init_cycle）
3. Master fork 新 Worker（使用新配置）
4. Master 向老 Worker 发送 SIGQUIT（优雅退出）
5. 老 Worker 处理完当前请求后退出
6. 切换完成，服务无中断
```

---

## 5. 二次开发注意事项

### 5.1 自定义模块开发流程

**步骤概述：**

1. 创建模块源文件（如 `ngx_http_mymodule_module.c`）
2. 创建 `config` 文件描述模块编译规则
3. 执行 `auto/configure --add-module=path/to/module` 将模块加入编译
4. `make && make install`

**最小 HTTP Content 模块示例：**

```c
// ngx_http_hello_module.c

#include <ngx_config.h>
#include <ngx_core.h>
#include <ngx_http.h>

static ngx_int_t ngx_http_hello_handler(ngx_http_request_t *r);

// 1. 配置指令定义
static ngx_command_t ngx_http_hello_commands[] = {
    {
        ngx_string("hello"),                      // 指令名
        NGX_HTTP_LOC_CONF | NGX_CONF_NOARGS,      // 适用范围
        ngx_http_hello,                           // 解析回调
        0, 0, NULL
    },
    ngx_null_command
};

// 2. HTTP 模块上下文
static ngx_http_module_t ngx_http_hello_module_ctx = {
    NULL,  /* preconfiguration */
    NULL,  /* postconfiguration */
    NULL,  /* create_main_conf */
    NULL,  /* init_main_conf */
    NULL,  /* create_srv_conf */
    NULL,  /* merge_srv_conf */
    NULL,  /* create_loc_conf */
    NULL   /* merge_loc_conf */
};

// 3. 模块定义
ngx_module_t ngx_http_hello_module = {
    NGX_MODULE_V1,
    &ngx_http_hello_module_ctx,  /* module context */
    ngx_http_hello_commands,     /* module directives */
    NGX_HTTP_MODULE,             /* module type */
    NULL, NULL, NULL, NULL, NULL, NULL, NULL,
    NGX_MODULE_V1_PADDING
};

// 4. 指令解析回调
static char *ngx_http_hello(ngx_conf_t *cf, ngx_command_t *cmd, void *conf) {
    ngx_http_core_loc_conf_t *clcf =
        ngx_http_conf_get_module_loc_conf(cf, ngx_http_core_module);
    clcf->handler = ngx_http_hello_handler;  // 注册内容 handler
    return NGX_CONF_OK;
}

// 5. 请求处理 handler
static ngx_int_t ngx_http_hello_handler(ngx_http_request_t *r) {
    ngx_int_t     rc;
    ngx_buf_t    *b;
    ngx_chain_t   out;

    // 只处理 GET/HEAD
    if (!(r->method & (NGX_HTTP_GET | NGX_HTTP_HEAD))) {
        return NGX_HTTP_NOT_ALLOWED;
    }

    // 发送响应头
    r->headers_out.status = NGX_HTTP_OK;
    r->headers_out.content_length_n = 13;
    rc = ngx_http_send_header(r);
    if (rc == NGX_ERROR || rc > NGX_OK || r->header_only) {
        return rc;
    }

    // 构造响应体
    b = ngx_pcalloc(r->pool, sizeof(ngx_buf_t));
    b->pos = (u_char *) "Hello, World!";
    b->last = b->pos + 13;
    b->memory = 1;
    b->last_buf = 1;

    out.buf = b;
    out.next = NULL;

    return ngx_http_output_filter(r, &out);  // 发送响应体（经过 filter 链）
}
```

**对应 `config` 文件：**

```sh
ngx_addon_name=ngx_http_hello_module
HTTP_MODULES="$HTTP_MODULES ngx_http_hello_module"
NGX_ADDON_SRCS="$NGX_ADDON_SRCS $ngx_addon_dir/ngx_http_hello_module.c"
```

---

### 5.2 模块结构详解

#### `ngx_command_t` 指令定义

```c
struct ngx_command_s {
    ngx_str_t             name;      // 指令名（如 ngx_string("proxy_pass")）
    ngx_uint_t            type;      // 适用范围+参数数量（位掩码）
    char               *(*set)(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
    ngx_uint_t            conf;      // 配置结构偏移（NGX_HTTP_LOC_CONF_OFFSET 等）
    ngx_uint_t            offset;    // 字段在配置结构体中的偏移（offsetof）
    void                 *post;      // 后处理回调（可为 NULL）
};
```

**常用内置 set 函数（可直接在 `offset` 处写入值）：**

| 函数 | 用途 |
|------|------|
| `ngx_conf_set_flag_slot` | on/off → `ngx_flag_t` |
| `ngx_conf_set_str_slot` | 字符串 → `ngx_str_t` |
| `ngx_conf_set_num_slot` | 整数 → `ngx_int_t` |
| `ngx_conf_set_msec_slot` | 时间（毫秒） → `ngx_msec_t` |
| `ngx_conf_set_size_slot` | 大小（支持 k/m） → `size_t` |
| `ngx_conf_set_off_slot` | 文件大小 → `off_t` |
| `ngx_conf_set_enum_slot` | 枚举值 |
| `ngx_conf_set_bitmask_slot` | 位掩码 |

#### 指令 `type` 位掩码

```
作用范围：NGX_HTTP_MAIN_CONF | NGX_HTTP_SRV_CONF | NGX_HTTP_LOC_CONF
参数数量：NGX_CONF_NOARGS | NGX_CONF_TAKE1 | NGX_CONF_TAKE2 | NGX_CONF_1MORE ...
块指令：  NGX_CONF_BLOCK
```

---

### 5.3 内存管理规范

NGINX 使用**内存池**（`ngx_pool_t`）而非直接 `malloc/free`，开发者必须遵守以下规范：

#### 内存池层级

| 内存池 | 生命周期 | 使用场景 |
|--------|---------|---------|
| `cf->pool`（配置内存池） | 从配置解析到 `ngx_init_cycle` 完成 | 配置解析期临时内存 |
| `cycle->pool` | 整个 cycle 生命周期 | 需要跨请求存活的全局配置数据 |
| `r->pool`（请求内存池） | 单个 HTTP 请求 | **最常用**，请求相关所有数据 |
| `c->pool`（连接内存池） | TCP 连接生命周期 | 连接级数据（如 SSL 状态） |

#### 关键 API

```c
// 分配内存（对齐）
void *ngx_palloc(ngx_pool_t *pool, size_t size);

// 分配内存（对齐，清零）
void *ngx_pcalloc(ngx_pool_t *pool, size_t size);

// 注册清理回调（池销毁时调用，用于释放外部资源）
ngx_pool_cleanup_t *ngx_pool_cleanup_add(ngx_pool_t *pool, size_t size);

// 释放大块内存（> 4095 字节）
void ngx_pfree(ngx_pool_t *pool, void *p);
```

#### ⚠️ 注意事项

- **不要使用** `malloc/free`，所有内存分配通过内存池。
- **不要在请求 handler 中保存指向 `r->pool` 内存的全局指针**，请求结束后池会被销毁。
- 需要跨请求的数据应分配在 `cycle->pool` 或共享内存中。
- 需要释放外部资源（如文件句柄、第三方库资源）时，使用 `ngx_pool_cleanup_add()` 注册回调。

---

### 5.4 HTTP 处理阶段挂载

在模块的 `postconfiguration` 回调中注册阶段 handler：

```c
static ngx_int_t ngx_http_mymodule_init(ngx_conf_t *cf) {
    ngx_http_handler_pt        *h;
    ngx_http_core_main_conf_t  *cmcf;

    cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);

    // 挂载到 ACCESS 阶段
    h = ngx_array_push(&cmcf->phases[NGX_HTTP_ACCESS_PHASE].handlers);
    if (h == NULL) {
        return NGX_ERROR;
    }
    *h = ngx_http_mymodule_access_handler;

    return NGX_OK;
}
```

#### 阶段 handler 返回值

| 返回值 | 含义 |
|--------|------|
| `NGX_OK` | 本 handler 完成，继续执行同阶段下一个 handler |
| `NGX_DECLINED` | 本 handler 不处理，跳过（最常见） |
| `NGX_AGAIN` | 需要等待更多数据 |
| `NGX_ERROR` | 出错，终止请求 |
| `NGX_HTTP_*`（如 `NGX_HTTP_FORBIDDEN`） | 返回对应 HTTP 状态码 |

**注意**：`CONTENT_PHASE` 中只有**第一个返回非 `NGX_DECLINED` 的 handler** 会处理请求，后续 handler 不再执行（与其他阶段略有不同）。

---

### 5.5 过滤器链开发

过滤器模块在 `postconfiguration` 中将自己插入链首：

```c
static ngx_http_output_header_filter_pt ngx_http_next_header_filter;
static ngx_http_output_body_filter_pt   ngx_http_next_body_filter;

static ngx_int_t ngx_http_myfilter_init(ngx_conf_t *cf) {
    // 保存当前链首，然后将自己设为链首
    ngx_http_next_header_filter = ngx_http_top_header_filter;
    ngx_http_top_header_filter  = ngx_http_myfilter_header_filter;

    ngx_http_next_body_filter = ngx_http_top_body_filter;
    ngx_http_top_body_filter  = ngx_http_myfilter_body_filter;
    return NGX_OK;
}

static ngx_int_t ngx_http_myfilter_header_filter(ngx_http_request_t *r) {
    // 处理响应头...
    // 必须调用链中下一个 filter
    return ngx_http_next_header_filter(r);
}

static ngx_int_t ngx_http_myfilter_body_filter(ngx_http_request_t *r,
    ngx_chain_t *in) {
    // 处理响应体...
    // 必须调用链中下一个 filter
    return ngx_http_next_body_filter(r, in);
}
```

**⚠️ 过滤器开发注意事项：**

- 过滤器注册越晚，执行越早（后进先出）。
- 修改响应体时注意正确处理 `ngx_buf_t` 的 `last_buf`/`last_in_chain` 标志。
- `Content-Length` 在修改内容后须重新设置（设为 -1 使用 chunked 编码）。

---

### 5.6 Upstream 模块开发

自定义协议代理（类似 `proxy_pass`、`fastcgi_pass`）需实现 upstream 接口：

```c
// 初始化 upstream（在 content handler 中调用）
ngx_http_upstream_t *u;
if (ngx_http_upstream_create(r) != NGX_OK) {
    return NGX_HTTP_INTERNAL_SERVER_ERROR;
}
u = r->upstream;

// 设置 upstream 回调
u->create_request   = my_create_request;    // 构造发给后端的请求
u->reinit_request   = my_reinit_request;    // 重试时重置
u->process_header   = my_process_header;    // 解析后端响应头
u->abort_request    = my_abort_request;     // 连接中止
u->finalize_request = my_finalize_request;  // 请求完成清理

// 设置后端地址
ngx_http_upstream_pass(r, &my_upstream);    // 或使用 peer
```

**关键回调说明：**

- `create_request`：在 `r->upstream->request_bufs` 中构造协议报文。
- `process_header`：解析后端响应头，设置 `u->headers_in`，返回 `NGX_OK`（完整）或 `NGX_AGAIN`（不完整）。
- 若后端是 HTTP，可直接复用 `ngx_http_proxy_module` 的大部分逻辑。

---

### 5.7 动态模块开发

NGINX 1.9.11+ 支持动态模块（`.so` 文件），无需重新编译主程序。

```sh
# 编译为动态模块
auto/configure --add-dynamic-module=path/to/module
make modules

# 产出：objs/ngx_http_mymodule_module.so

# nginx.conf 中加载
load_module modules/ngx_http_mymodule_module.so;
```

**动态模块要求：**

- 模块签名（`signature` 字段）必须与主程序编译特征完全一致。
- 不能修改 `ngx_module_t` 中的 `spare_hook*` 字段以外的扩展字段（兼容性原因）。
- 模块定义必须使用 `NGX_MODULE_V1` 和 `NGX_MODULE_V1_PADDING` 宏。

---

### 5.8 常见陷阱与最佳实践

#### 内存安全

```c
// ❌ 错误：使用栈内存存储需要长期保存的数据
char buf[256];
snprintf(buf, sizeof(buf), "value");
conf->str.data = (u_char *)buf;  // 栈内存在函数退出后失效！

// ✅ 正确：从内存池分配
conf->str.data = ngx_pnalloc(cf->pool, 256);
```

#### 字符串操作

```c
// NGINX 使用 ngx_str_t = {data, len}，不以 NUL 结尾
// ❌ 不能直接用 strlen/strcmp/printf("%s")

// ✅ 使用 NGINX 提供的宏和函数
ngx_str_t s = ngx_string("hello");  // 静态字符串初始化
ngx_str_set(&s, "hello");           // 设置字符串

// 格式化输出
ngx_snprintf(buf, size, "%V", &ngx_str);  // %V 打印 ngx_str_t
ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
              "error: %V", &some_str);
```

#### 子请求（Subrequest）

```c
// 创建子请求（用于 auth_request、mirror、SSI 等）
ngx_http_subrequest(r, &uri, &args, &sr, &psr_callback, flags);
// 注意：父请求在子请求完成前会被挂起（NGX_DONE）
```

#### 共享内存

```c
// limit_req、limit_conn 等需要跨 Worker 共享状态时使用 slab 分配器
// 在 ngx_http_module_t.init_main_conf 中初始化共享内存
shm_zone = ngx_shared_memory_add(cf, &name, size, &my_module);
shm_zone->init = my_shm_init;
// my_shm_init 中通过 ngx_slab_pool_t 分配
```

#### 异步操作

```c
// NGINX 是单线程事件循环，handler 绝不能阻塞！
// ❌ 禁止的阻塞操作：read()、write()、sleep()、DNS 查询（同步）

// ✅ 磁盘 I/O → 使用 aio（thread pool 或 Linux AIO）
// ✅ 网络请求 → 使用 upstream 机制
// ✅ 需要延迟执行 → 使用定时器（ngx_add_timer）
// ✅ DNS 解析 → 使用 ngx_resolver（异步）
```

#### 日志规范

```c
// 调试日志（编译时需要 --with-debug）
ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
               "my module: value=%ui", value);

// 错误日志
ngx_log_error(NGX_LOG_ERR, r->connection->log, ngx_errno,
              "open() \"%s\" failed", filename);
```

#### 热重载兼容性

- 模块在 `init_process` 中分配的资源需要在 `exit_process` 中释放。
- 共享内存在 `ngx_init_cycle` 时重新 attach，注意指针不可跨 cycle 持有。
- `init_module` 在每次 `ngx_init_cycle` 时都会调用（热重载时也调用），需要幂等处理。

#### 编译选项与兼容性

```sh
# 开发阶段推荐加入 debug 和 asan
./auto/configure \
    --with-debug \
    --with-cc-opt="-fsanitize=address -g -O0" \
    --with-ld-opt="-fsanitize=address"
```

---

## 6. 关键数据结构速查

| 结构体 | 文件 | 说明 |
|--------|------|------|
| `ngx_cycle_t` | `src/core/ngx_cycle.h` | 全局运行时状态 |
| `ngx_connection_t` | `src/core/ngx_connection.h` | 单个 TCP 连接 |
| `ngx_event_t` | `src/event/ngx_event.h` | I/O 事件（读/写/定时器） |
| `ngx_http_request_t` | `src/http/ngx_http_request.h` | 单个 HTTP 请求 |
| `ngx_http_upstream_t` | `src/http/ngx_http_upstream.h` | 到上游的连接与状态 |
| `ngx_pool_t` | `src/core/ngx_palloc.h` | 内存池 |
| `ngx_buf_t` | `src/core/ngx_buf.h` | 数据缓冲区 |
| `ngx_chain_t` | `src/core/ngx_buf.h` | 链式缓冲区 |
| `ngx_str_t` | `src/core/ngx_string.h` | 长度+指针字符串 |
| `ngx_conf_t` | `src/core/ngx_conf_file.h` | 配置解析上下文 |
| `ngx_module_t` | `src/core/ngx_module.h` | 模块描述符 |
| `ngx_command_t` | `src/core/ngx_conf_file.h` | 配置指令描述符 |
| `ngx_http_module_t` | `src/http/ngx_http_config.h` | HTTP 模块上下文 |
| `ngx_http_core_loc_conf_t` | `src/http/ngx_http_core_module.h` | location 配置 |
| `ngx_rbtree_t` | `src/core/ngx_rbtree.h` | 红黑树（定时器等） |
| `ngx_slab_pool_t` | `src/core/ngx_slab.h` | 共享内存 slab 分配器 |

---

> 本文档基于 NGINX 当前主干代码（mainline）编写，细节可能随版本更新有所变化。
> 更多信息请参考官方开发指南：https://nginx.org/en/docs/dev/development_guide.html
