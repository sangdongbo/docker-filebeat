# Docker ELK 日志采集配置

> Laravel 应用日志采集到 ELK Stack (Elasticsearch, Logstash, Kibana) 的完整配置

## 📁 目录结构

```
docker/
├── README.md                      # 本文档
├── docker-compose.filebeat.yml    # Filebeat 容器编排配置
├── filebeat/
│   └── filebeat.yml              # Filebeat 日志采集配置
└── logstash/
    └── pipeline/
        └── laravel.conf          # Logstash 日志处理管道配置
```

## 🎯 功能概述

这套配置实现了 Laravel 应用日志的自动采集、解析和存储：

1. **Filebeat** - 从 Laravel `storage/logs` 目录采集日志文件
2. **Logstash** - 解析 Laravel 日志格式，提取结构化字段
3. **Elasticsearch** - 存储日志数据，按频道和日期分索引
4. **Kibana** - 可视化查看和分析日志

### 支持的日志格式

- Laravel 默认日志格式：`[2026-01-20 10:30:45] local.INFO: 消息内容`
- 多行日志（异常堆栈跟踪）自动合并
- JSON 上下文自动解析

### 日志频道支持

- `default` - 默认日志 (laravel.log)
- `api` - API 日志 (api.log)
- `payment` - 支付日志 (payment.log)
- `request` - 请求日志 (request.log)

## 🚀 快速开始

### 前置要求

1. **已安装并运行 docker-elk**
   ```bash
   # 克隆 docker-elk 项目
   git clone https://github.com/deviantony/docker-elk.git
   cd docker-elk
   
   # 启动 ELK Stack
   docker-compose up -d
   ```

2. **确认 ELK 服务正常运行**
   ```bash
   # 检查容器状态
   docker ps | grep -E "elasticsearch|logstash|kibana"
   
   # 访问 Kibana: http://localhost:5601
   # 用户名: elastic
   # 密码: 查看 docker-elk/.env 文件
   ```

### 启动步骤

#### 1. 修改网络配置

编辑 `docker-compose.filebeat.yml`，确认网络名称正确：

```bash
# 查看 ELK 网络名称
docker network ls | grep elk

# 修改 docker-compose.filebeat.yml 中的 networks.elk_network.name
# 通常是: docker-elk_elk 或 elk_default
```

#### 2. 修改日志路径（如果需要）

编辑 `filebeat/filebeat.yml`，确认日志路径：

```yaml
paths:
  - /var/log/laravel/*.log  # 容器内路径
```

编辑 `docker-compose.filebeat.yml`，确认卷挂载：

```yaml
volumes:
  - ../storage/logs:/var/log/laravel:ro  # 宿主机路径:容器路径
```

#### 3. 启动 Filebeat

```bash
# 进入 docker 目录
cd /path/to/your/project/docker

# 启动 Filebeat
docker-compose -f docker-compose.filebeat.yml up -d

# 查看日志（确认启动成功）
docker-compose -f docker-compose.filebeat.yml logs -f filebeat
```

#### 4. 验证日志采集

```bash
# 方法 1: 生成测试日志（如果配置了测试路由）
curl http://localhost:8080/api/test-logs

# 方法 2: 手动写入日志
echo '[2026-01-20 10:30:45] local.INFO: 测试日志' >> ../storage/logs/laravel.log

# 等待 10-15 秒后查询 Elasticsearch
curl -u elastic:Es123456 \
  "http://localhost:9200/laravel-logs-*/_search?pretty" \
  -H "Content-Type: application/json" \
  -d '{"size":1,"sort":[{"@timestamp":"desc"}]}'
```

#### 5. 在 Kibana 中查看日志

1. 访问 http://localhost:5601
2. 登录（用户名: elastic, 密码: 查看 docker-elk 配置）
3. 进入 **Management** → **Stack Management** → **Data Views**
4. 创建 Data View：
   - Name: `Laravel Logs`
   - Index pattern: `laravel-logs-*`
   - Timestamp field: `@timestamp`
5. 进入 **Analytics** → **Discover** 查看日志

## 📝 配置文件说明

### docker-compose.filebeat.yml

Filebeat 容器的 Docker Compose 配置文件。

**需要修改的配置：**
- `networks.elk_network.name` - ELK 网络名称
- `volumes` - Laravel 日志目录路径
- `environment.APP_ENV` - 应用环境标识

**详细说明：** 查看文件内的注释

### filebeat/filebeat.yml

Filebeat 的核心配置文件，定义日志采集规则和输出目标。

**需要修改的配置：**
- `filebeat.inputs.paths` - 日志文件路径模式
- `output.logstash.hosts` - Logstash 服务地址
- `fields.app` - 应用标识（区分不同应用）

**支持的输出方式：**
- ✅ Logstash（推荐）- 支持复杂的日志解析和转换
- ⚪ Elasticsearch - 直接输出，架构简单但功能有限
- ⚪ Kafka - 用于流式处理或多级架构

**详细说明：** 查看文件内的注释

### logstash/pipeline/laravel.conf

Logstash 管道配置，定义日志的解析、转换和路由规则。

**核心功能：**
- 解析 Laravel 日志格式（时间戳、环境、级别、消息）
- 提取 JSON 上下文数据
- 根据日志级别添加标签
- 根据文件名识别日志频道
- 添加地理位置信息（IP → 城市/国家）
- 按频道和日期创建 Elasticsearch 索引

**需要修改的配置：**
- `output.elasticsearch.hosts` - Elasticsearch 地址
- `output.elasticsearch.user` - 用户名
- `output.elasticsearch.password` - 密码（环境变量）

**详细说明：** 查看文件内的注释

## 🔧 常用命令

### 容器管理

```bash
# 启动 Filebeat
docker-compose -f docker-compose.filebeat.yml up -d

# 停止 Filebeat
docker-compose -f docker-compose.filebeat.yml down

# 重启 Filebeat
docker-compose -f docker-compose.filebeat.yml restart filebeat

# 查看日志
docker-compose -f docker-compose.filebeat.yml logs -f filebeat

# 查看容器状态
docker-compose -f docker-compose.filebeat.yml ps
```

### 配置验证

```bash
# 验证 Filebeat 配置文件
docker exec laravel_filebeat filebeat test config

# 验证 Filebeat 输出连接
docker exec laravel_filebeat filebeat test output

# 验证 Logstash 配置文件
docker exec docker-elk_logstash_1 logstash --config.test_and_exit \
  --path.config /usr/share/logstash/pipeline/
```

### 日志查询

```bash
# 查看所有 Laravel 日志索引
curl -u elastic:Es123456 "http://localhost:9200/_cat/indices/laravel-logs-*?v"

# 查询最新的 10 条日志
curl -u elastic:Es123456 \
  "http://localhost:9200/laravel-logs-*/_search?pretty" \
  -H "Content-Type: application/json" \
  -d '{"size":10,"sort":[{"@timestamp":"desc"}]}'

# 查询错误日志
curl -u elastic:Es123456 \
  "http://localhost:9200/laravel-logs-*/_search?pretty" \
  -H "Content-Type: application/json" \
  -d '{"query":{"match":{"level":"ERROR"}},"size":10}'

# 查询特定频道的日志
curl -u elastic:Es123456 \
  "http://localhost:9200/laravel-logs-*/_search?pretty" \
  -H "Content-Type: application/json" \
  -d '{"query":{"match":{"log_channel":"api"}},"size":10}'

# 查看日志总数
curl -u elastic:Es123456 "http://localhost:9200/laravel-logs-*/_count?pretty"
```

### 索引管理

```bash
# 删除指定日期的索引
curl -X DELETE -u elastic:Es123456 \
  "http://localhost:9200/laravel-logs-*-2026.01.15"

# 删除 30 天前的所有索引
curl -X DELETE -u elastic:Es123456 \
  "http://localhost:9200/laravel-logs-*-2025.12.*"

# 查看索引磁盘使用情况
curl -u elastic:Es123456 \
  "http://localhost:9200/_cat/indices/laravel-logs-*?v&h=index,store.size&s=index"
```

## 🐛 故障排查

### Filebeat 无法启动

**问题：** 容器启动后立即退出

**排查步骤：**
```bash
# 1. 查看容器日志
docker logs laravel_filebeat

# 2. 检查配置文件语法
docker run --rm \
  -v $(pwd)/filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml \
  docker.elastic.co/beats/filebeat:8.11.0 \
  filebeat test config

# 3. 检查文件权限
ls -la filebeat/filebeat.yml
# 应该可读: -rw-r--r--

# 4. 检查日志目录挂载
docker exec laravel_filebeat ls -la /var/log/laravel/
```

**常见原因：**
- 配置文件语法错误（YAML 格式问题）
- 日志目录挂载失败（路径不存在）
- 权限不足（无法读取日志文件）

### Filebeat 连接不上 Logstash

**问题：** 日志显示 `connection refused` 或 `no such host`

**排查步骤：**
```bash
# 1. 检查 Logstash 是否运行
docker ps | grep logstash

# 2. 检查 Logstash 端口是否监听
docker exec docker-elk_logstash_1 netstat -tln | grep 5044

# 3. 检查网络连接
docker exec laravel_filebeat ping -c 3 docker-elk_logstash_1
docker exec laravel_filebeat nc -zv docker-elk_logstash_1 5044

# 4. 检查网络配置
docker network inspect docker-elk_elk | grep laravel_filebeat
```

**解决方法：**
- 确认 Logstash 容器名称正确（修改 filebeat.yml 中的 hosts）
- 确认 Filebeat 和 Logstash 在同一 Docker 网络
- 确认 Logstash 的 beats input 已启动

### 日志没有被采集

**问题：** Filebeat 运行正常，但 Elasticsearch 中没有数据

**排查步骤：**
```bash
# 1. 检查 Filebeat 是否在读取文件
docker exec laravel_filebeat cat /usr/share/filebeat/data/registry/filebeat/log.json

# 2. 生成测试日志
echo '[2026-01-20 10:30:45] local.INFO: 测试日志' >> ../storage/logs/laravel.log

# 3. 查看 Filebeat 日志
docker logs laravel_filebeat -f

# 4. 查看 Logstash 日志
docker logs docker-elk_logstash_1 -f

# 5. 检查 Logstash 是否收到数据
curl "http://localhost:9600/_node/stats/pipelines?pretty" | grep events
```

**常见原因：**
- 日志文件路径不匹配（检查 paths 配置）
- 日志格式不匹配（检查 grok pattern）
- Logstash 解析失败（查看 Logstash 日志中的错误）
- Elasticsearch 认证失败（检查用户名密码）

### 日志解析失败

**问题：** 日志进入 Elasticsearch 但字段不正确，有 `_grokparsefailure` 标签

**排查步骤：**
```bash
# 1. 查询解析失败的日志
curl -u elastic:Es123456 \
  "http://localhost:9200/laravel-logs-*/_search?pretty" \
  -H "Content-Type: application/json" \
  -d '{"query":{"match":{"tags":"_grokparsefailure_laravel"}},"size":1}'

# 2. 查看原始消息内容
# 从上面的结果中找到 message 字段

# 3. 测试 grok pattern
# 使用 Kibana Dev Tools 的 Grok Debugger
# 或访问: http://grokdebug.herokuapp.com/
```

**解决方法：**
- 检查日志格式是否与 Laravel 标准格式一致
- 修改 logstash/pipeline/laravel.conf 中的 grok pattern
- 查看 Laravel 的 logging.php 配置

### Kibana 中看不到日志

**问题：** Elasticsearch 有数据，但 Kibana 中看不到

**排查步骤：**
```bash
# 1. 确认索引存在
curl -u elastic:Es123456 "http://localhost:9200/_cat/indices/laravel-logs-*?v"

# 2. 确认有文档
curl -u elastic:Es123456 "http://localhost:9200/laravel-logs-*/_count?pretty"
```

**解决方法：**
1. 检查 Data View 的索引模式是否正确（`laravel-logs-*`）
2. 检查时间范围选择器（右上角）
3. 刷新 Data View 的字段列表
4. 检查是否有过滤器限制了结果

## 📚 相关文档

- **Kibana 使用指南**: [../docs/KIBANA-GUIDE.md](../docs/KIBANA-GUIDE.md)
- **ELK 日志系统完整文档**: [../docs/ELK-LOGGING-GUIDE.md](../docs/ELK-LOGGING-GUIDE.md)
- **测试脚本**: [../test-elk-logs.sh](../test-elk-logs.sh)

### 官方文档

- [Filebeat 官方文档](https://www.elastic.co/guide/en/beats/filebeat/current/index.html)
- [Logstash 官方文档](https://www.elastic.co/guide/en/logstash/current/index.html)
- [Elasticsearch 官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
- [Kibana 官方文档](https://www.elastic.co/guide/en/kibana/current/index.html)
- [docker-elk 项目](https://github.com/deviantony/docker-elk)

## 💡 最佳实践

### 1. 索引生命周期管理

定期清理旧日志，避免磁盘空间不足：

```bash
# 方式 1: 手动删除（推荐用于开发环境）
curl -X DELETE -u elastic:Es123456 \
  "http://localhost:9200/laravel-logs-*-2025.*"

# 方式 2: 使用 ILM 策略（推荐用于生产环境）
# 在 Kibana 中配置: Management → Stack Management → Index Lifecycle Policies
```

### 2. 日志级别控制

根据环境调整日志级别，减少不必要的日志：

```php
// config/logging.php
'level' => env('LOG_LEVEL', 'debug'),

// .env
// 开发环境
LOG_LEVEL=debug

// 生产环境
LOG_LEVEL=info
```

### 3. 敏感信息过滤

在 Logstash 中过滤敏感信息：

```ruby
# logstash/pipeline/laravel.conf
mutate {
  gsub => [
    "log_message", "password=\S+", "password=***",
    "log_message", "api_key=\S+", "api_key=***",
    "log_message", "token=\S+", "token=***"
  ]
}
```

### 4. 性能优化

```yaml
# filebeat.yml - 批量发送优化
output.logstash:
  bulk_max_size: 2048
  compression_level: 3
```

```ruby
# laravel.conf - 批量写入优化
elasticsearch {
  bulk_max_size: 500
  flush_size: 500
  idle_flush_time: 1
}
```

### 5. 监控和告警

在 Kibana 中创建告警规则：

1. 进入 **Stack Management** → **Rules and Connectors**
2. 创建规则：
   - 条件：5 分钟内 ERROR 日志超过 10 条
   - 动作：发送邮件/Slack/Webhook 通知

## 🔐 安全建议

1. **使用环境变量管理密码**
   ```bash
   # 不要在配置文件中硬编码密码
   # 使用环境变量或 Docker secrets
   ```

2. **启用 SSL/TLS**
   ```yaml
   # 生产环境建议启用 SSL
   output.logstash:
     ssl.enabled: true
     ssl.certificate_authorities: ["/etc/filebeat/ca.crt"]
   ```

3. **限制网络访问**
   ```yaml
   # 只允许必要的容器访问 ELK 网络
   # 使用 Docker 网络隔离
   ```

4. **定期更新**
   ```bash
   # 定期更新 ELK Stack 和 Filebeat 到最新版本
   # 关注安全公告
   ```

## 🤝 贡献

如果你发现配置问题或有改进建议，欢迎：
- 提交 Issue
- 发起 Pull Request
- 完善文档

## 📄 许可证

本配置遵循项目的许可证。

---

**维护者**: 项目团队  
**最后更新**: 2026-01-20  
**版本**: 1.0.0
