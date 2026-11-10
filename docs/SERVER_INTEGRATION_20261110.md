# 信号服务器对接草案 · 2026-11-10

服务器由执行方同事维护，尚未建立。本轮交付消息合同建议、配置模板、合成消息和联调用例；未部署接收/发送服务，未产生网络投递或消费验收。正式域名、IP、鉴权材料和账户数据通过私有通道配置，不放公开库。模型、特征、选择阈值和前瞻记录规则不变。

## 1. 默认架构

```mermaid
flowchart LR
 A[我方公开行情采集与冻结v3] --> B[原前瞻记录器留痕]
 B --> C[待发送队列]
 C -->|HTTPS推送| D[同事维护的接收服务]
 D --> E[持久化消息与消费游标]
 E --> F[内网候选读取]
 F --> G[结合实际账户运行RC2与风控]
 G --> H[执行系统与真实回执]
 H -->|私有回传通道| I[我方状态对账与复盘]
```

不要求外部机器直接连内网交易服务。候选服务器只负责接收、留存和转发，不取得下单/借币/划转能力。内网下单属于执行方；我方策略接收实际账户状态后给出JSON决策。是否由我方进程或执行侧受控进程运行RC2，在测试网络建好后确认，字段和权限边界相同。

现有`score.py`只是离线评分入口，不等于完整125特征采集/前瞻记录服务。完整机器迁移须按EXTERNAL_ASSETS和绑定校验接入授权资产，不能只把历史样例或GitHub页面当实时服务。

## 2. 三类业务消息分开

| 消息 | 作用 | 生成与有效期 | 交易权限 |
|---|---|---|---|
| v3候选/预警批次 | 提前观察及借币候选参考 | 每日00UTC=北京时间08:00；原网格后5分钟内实际发出并留痕；单条输出`valid_until_utc=asof+24h`，受模型到期进一步限制 | WATCH_ONLY，false；不直接下单 |
| RC2策略决策 | 当前开/加/减/退/保持及敞口修复 | 按实际行情与账户状态计算；输入最大年龄10秒、输出TTL60秒，按各自合同严格检查 | 当前冻结参考仍为false；生产许可另行验收 |
| 账户/成交/资金回执 | 更新事实和账本 | 事件即时＋完整快照；含实际event/received时点、版本和缺口 | 不因决策过期丢弃已经发生的成交 |

72h/168h/240h/1008h是各预测目标的观察/成熟窗口，不是信号可下单TTL。五分钟是原始前瞻发出记录期限，不是“送晚了可以把时间改回去”。

若原预测在五分钟内真实生成并留痕，但网络晚送，保留原`issued_at`，另记服务器实际接收时间和LATE_DELIVERY；不得改写前瞻原账本。若首次生成就迟到，只发缺失/迟到状态，不补造该网格预测。消费端不把迟到重传当新鲜候选；接收超时不改变原本有效期。

2026-12-01 00UTC模型过期后继续采数、跟踪已发预测和运行健康信息，停止新预测，不自动重训。服务重启、迁移及接服务器都不刷新模型有效期。

## 3. 传输层建议，待双方确认

| 项目 | 草案 |
|---|---|
| 接收 | `POST /v1/signal-batches`；返回接收结果和最高连续落盘序号 |
| 内网读取 | `GET /v1/signal-batches?after_sequence=...&limit=...`；只能按白名单类型读取，不混入账户密钥 |
| 消费确认 | `POST /v1/consumer-acks`；consumer_id、message_id、payload_sha256、消费状态/时间 |
| 健康 | 独立health/heartbeat；健康不等于本日有预测或覆盖完整 |
| 投递 | 至少一次，生产端持久待发送队列，接收端按唯一键幂等；不承诺网络层exactly-once |
| 唯一键 | 稳定stream_id＋message_id，附持久递增sequence；进程重启不把已发送序号归零 |
| 重复 / 冲突 | 同ID同原始payload字节为重复，返回原确认；同ID不同hash为冲突，隔离并告警 |
| 确认层次 | DURABLY_STORED、CONSUMED、策略计算/执行结果分别记录；HTTP200不证明持久化或成交 |
| 断线 | 按原ID/序号/原内容重传，退避、尊重服务Retry-After；过期消息留审计，不激活新增暴露 |
| 鉴权 | HTTPS＋双方选择mTLS或HMAC；HMAC建议覆盖method、path、transport_time、nonce、body_hash；密钥从服务端安全配置读取，不进入模型JSON |
| 防重放 | transport时间/nonce用于请求鉴权；与原业务issued_at分开，重传可用新transport nonce但业务ID/内容不变 |
| 消费游标 | 最高连续序号持久化；缺序不能悄悄跳过，补传/快照恢复后再确认 |

这些路径和传输配置是DRAFT，不代表同事已实现。限流、body大小、心跳和超时在测试服务阶段协商；不据此变更模型网格、原始记录期限或RC2时效。

## 4. 外层消息，不修改冻结业务JSON

建议字段：`transport_schema`、`message_type`、`message_id`、`stream_id`、`sequence`、`environment`、`synthetic`、`issued_at_utc`、`asof_utc`、`recording_deadline_utc`、`valid_until_utc`、`model_version`、`strategy_id`、`payload_encoding`、`payload_sha256`、`payload_utf8`。

`payload_utf8`保存完整、不可改写的UTF-8 JSON原文，hash按UTF-8原始字节计算；服务端不得靠自行重排JSON键来重算另一份内容。真实批次payload包含batch_status、selection_contract_version、原记录器已选取的signals及预测/源/模型证据引用；不在服务器再次训练、重打分或重新Top10。

批次状态至少区分READY、NO_CANDIDATES、DATA_MISSING、MODEL_EXPIRED、PAUSED和TEST_FIXTURE。READY不能用空列表伪装成功；没有合格候选、没有可用数据、进程没运行是不同状态。空预测不能回填0概率。合成消息`synthetic=true`、`environment=integration_test`，接收服务不得把它加入真实预测统计。

传输消息只引用不含秘密的source/model哈希和内部受控证据ID；原始账户、库存和供应商凭证不进入公开推送样例。账户及执行回执使用单独的受控通道和Schema，见[策略侧账户清单](https://github.com/leslieshen1/negative-funding-strategy/blob/main/docs/ACCOUNT_DATA_REQUEST_20261110.md)。

参考文件：[配置模板](../integration/server-profile.template.json)、[合成空批次消息](../integration/message.synthetic.json)、[验收用例](../integration/ACCEPTANCE_CASES.md)。文件是准备材料，尚无可部署的收发服务；JSON语法和hash检查不等于服务联通或落盘/幂等验收。

## 5. 对方服务器准备好后，先确认这些信息

测试地址及网络边界；接收路径/返回值和接口负责人；mTLS或HMAC及私下发放方式；内网消费方式/消费者标识；落盘位置、留存及灾备；限流/超时/心跳；时间同步与日志关联。随后按验收清单跑空批次、正常合成样例、重复/乱序/过期/断线恢复，取得落盘和消费确认，再开始真实候选的只读影子接收。

当前保留WATCH_ONLY及`submission_allowed=false`。服务器接入完成不自动代表模型效果、交易策略或实盘生产准入通过。
