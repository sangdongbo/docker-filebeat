# Laravel + ELK Stack 日志系统使用指南

## 📋 架构概览

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         你的服务器 / 本地开发环境                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────┐                                                    │
│  │   Laravel App   │                                                    │
│  │                 │                                                    │
│  │  storage/logs/  │                                                    │
│  │  ├── laravel.log│──┐                                                 │
│  │  ├── api.log    │  │                                                 │
│  │  ├── payment.log│  │                                                 │
│  │  └── request.log│  │                                                 │
│  └─────────────────┘  │                                                 │
│                       │                                                 │
│                       ▼                                                 │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    Docker ELK Stack                              │   │
│  │  ┌──────────┐    ┌──────────┐    ┌─────────────┐   ┌─────────┐  │   │
│  │  │ Filebeat │───>│ Logstash │───>│Elasticsearch│<──│ Kibana  │  │   │
│  │  │  :5044   │    │  :5044   │    │   :9200     │   │  :5601  │  │   │
│  │  │ 日志采集  │    │ 日志处理  │    │  存储搜索   │   │  可视化  │  │   │
│  │  └──────────┘    └──────────┘    └─────────────┘   └─────────┘  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## 🚀 快速开始

### 方式一：使用现有的 docker-elk

如果你已经通过 `docker-elk` 项目启动了 ELK Stack，只需要添加 Filebeat：

```bash
# 1. 查看 ELK 网络名称
docker network ls | grep elk

# 2. 启动 Filebeat（确保在项目根目录）
cd /Users/sangdongbo/Develop/lanjing/my-laravel12-app
docker-compose -f docker/docker-compose.filebeat.yml up -d

# 3. 查看 Filebeat 日志
docker logs -f laravel_filebeat
```

### 方式二：复制 Logstash Pipeline 到 docker-elk

将 Laravel 的 Logstash 配置复制到你的 docker-elk 项目：

```bash
# 复制 pipeline 配置
cp docker/logstash/pipeline/laravel.conf /path/to/docker-elk/logstash/pipeline/

# 重启 Logstash
docker-compose -f /path/to/docker-elk/docker-compose.yml restart logstash
```

## 📁 文件结构

```
my-laravel12-app/
├── docker/
│   ├── filebeat/
│   │   └── filebeat.yml              # Filebeat 配置
│   ├── logstash/
│   │   └── pipeline/
│   │       └── laravel.conf          # Logstash 处理规则
│   └── docker-compose.filebeat.yml   # Filebeat Docker 配置
├── scripts/
│   └── setup-elk.sh                  # 一键配置脚本
└── storage/
    └── logs/                         # Laravel 日志目录
        ├── laravel.log
        ├── api.log
        ├── payment.log
        └── request.log
```

## 🔧 配置说明

### 1. Filebeat 配置 (docker/filebeat/filebeat.yml)

负责监控日志文件并发送到 Logstash：

- **监控路径**: `/var/log/laravel/*.log`（容器内路径）
- **多行处理**: 自动合并 Laravel 的堆栈跟踪
- **输出目标**: Logstash:5044

### 2. Logstash Pipeline (docker/logstash/pipeline/laravel.conf)

负责解析和处理日志：

- **解析格式**: `[时间] 环境.级别: 消息 {JSON上下文}`
- **字段提取**: timestamp, level, message, context
- **索引策略**: `laravel-logs-{频道}-{日期}`

### 3. 日志频道映射

| Laravel 日志文件 | ES 索引前缀 |
|-----------------|------------|
| laravel.log | laravel-logs-default-* |
| api.log | laravel-logs-api-* |
| payment.log | laravel-logs-payment-* |
| request.log | laravel-logs-request-* |

## 📊 Kibana 使用

### 1. 创建 Index Pattern

1. 打开 Kibana: http://localhost:5601
2. 点击左侧菜单 → Management → Stack Management
3. 选择 Data → Index Management 查看索引
4. 选择 Kibana → Data Views（或 Index Patterns）
5. 点击 "Create data view"
6. 输入 `laravel-logs-*`
7. 选择 `@timestamp` 作为时间字段
8. 保存

### 2. 在 Discover 中查看日志

1. 点击左侧菜单 → Discover
2. 选择你创建的 Index Pattern
3. 使用 KQL 搜索，例如：
   - `level: ERROR` - 查看错误日志
   - `log_channel: payment` - 查看支付日志
   - `context.ip: 127.0.0.1` - 按 IP 搜索

### 3. 创建 Dashboard

1. 点击左侧菜单 → Dashboard
2. 创建可视化：
   - 日志级别分布（饼图）
   - 每小时错误数量（折线图）
   - 热门 API 接口（表格）
   - 响应状态码分布（柱状图）

## 🔍 常用搜索语法 (KQL)

```
# 搜索错误日志
level: ERROR OR level: CRITICAL

# 搜索特定频道
log_channel: "api"

# 搜索包含某关键词的日志
message: *支付*

# 搜索特定 IP
context.ip: "192.168.1.100"

# 组合搜索
level: ERROR AND log_channel: payment

# 时间范围（在界面上选择更方便）
@timestamp >= "2024-01-20" AND @timestamp < "2024-01-21"
```

## 🛠 运维命令

### 查看服务状态

```bash
# 查看所有相关容器
docker ps | grep -E "elastic|kibana|logstash|filebeat"

# 查看 Filebeat 日志
docker logs -f laravel_filebeat

# 查看 Logstash 日志
docker logs -f docker-elk_logstash_1
```

### Elasticsearch 操作

```bash
# 检查集群健康
curl http://localhost:9200/_cluster/health?pretty

# 查看所有索引
curl http://localhost:9200/_cat/indices?v

# 查看 Laravel 日志索引
curl http://localhost:9200/_cat/indices/laravel-logs-*?v

# 搜索最近的日志
curl http://localhost:9200/laravel-logs-*/_search?pretty -H "Content-Type: application/json" -d '
{
  "query": { "match_all": {} },
  "size": 5,
  "sort": [{ "@timestamp": "desc" }]
}'

# 删除旧索引（保留最近30天）
curl -X DELETE "http://localhost:9200/laravel-logs-*-2024.01.01"
```

### 索引生命周期管理 (ILM)

创建自动清理策略：

```bash
curl -X PUT "localhost:9200/_ilm/policy/laravel-logs-policy" -H "Content-Type: application/json" -d '
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_age": "7d",
            "max_size": "5gb"
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "allocate": {
            "number_of_replicas": 0
          }
        }
      },
      "delete": {
        "min_age": "30d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}'
```

## ⚠️ 常见问题

### 1. Filebeat 连接不上 Logstash

```bash
# 检查网络
docker network inspect docker-elk_elk

# 确保 Filebeat 在同一网络
docker network connect docker-elk_elk laravel_filebeat
```

### 2. 日志没有出现在 Kibana

```bash
# 1. 检查 Filebeat 是否在发送数据
docker logs laravel_filebeat | tail -20

# 2. 检查 Logstash 是否在接收数据
docker logs docker-elk_logstash_1 | tail -20

# 3. 检查索引是否创建
curl http://localhost:9200/_cat/indices/laravel-logs-*?v
```

### 3. 日志解析失败

```bash
# 查看解析失败的日志（带有 _grokparsefailure 标签）
curl http://localhost:9200/laravel-logs-*/_search?pretty -H "Content-Type: application/json" -d '
{
  "query": {
    "term": { "tags": "_grokparsefailure_laravel" }
  }
}'
```

## 📈 生产环境建议

1. **资源配置**
   - Elasticsearch: 至少 4GB 内存
   - Logstash: 至少 2GB 内存
   - 使用 SSD 存储

2. **安全配置**
   - 启用 X-Pack Security
   - 配置 HTTPS
   - 设置用户认证

3. **高可用**
   - Elasticsearch 集群至少 3 节点
   - 配置副本分片

4. **监控**
   - 使用 Metricbeat 监控 ELK 自身
   - 设置告警规则

## 📚 相关链接

- [Filebeat 官方文档](https://www.elastic.co/guide/en/beats/filebeat/current/index.html)
- [Logstash 官方文档](https://www.elastic.co/guide/en/logstash/current/index.html)
- [Elasticsearch 官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
- [Kibana 官方文档](https://www.elastic.co/guide/en/kibana/current/index.html)
