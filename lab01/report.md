实验 01 报告：云端数据接口与可复现环境



一、基本信息



课程：云计算与大数据处理。实验：01 云端数据接口与可复现环境。姓名：李维畅。学号：2410311306。日期：2026/10/13个人工作目录：C:\\Users\\Administrator\\Desktop\\BDP-main\\BDP-main。



二、环境



操作系统为 Windows，使用 PowerShell。Python 版本 3.12.10，解释器路径为 C:\\Users\\Administrator\\Desktop\\BDP-main\\BDP-main\\.venv\\Scripts\\python.exe。关键包版本：boto3 1.43.93，moto 5.2.3，flask 3.1.3。pip check 结果为 No broken requirements found。环境证据保存在 outputs/lab01/evidence 下，解释器证据文件为 27be1fc0d0d2421989353e943391060c-probe-01.html，依赖检查证据文件为 f5be68b1a2554efc811b7127ef244dce-environment.html。



三、问题与数据



本实验围绕模拟服务使用分析与规则查询平台，使用合成服务记录和虚构规则练习对象存储往返与数据工程。数据由课程命令 coursekit seed --rows 20000 生成，随机种子为 42。数据路径为 data/raw/events.jsonl，格式为 JSONL，即一行一个 JSON 对象。总行数为 20102，其中 20000 条为合法独特事件，100 条为完全重复事件，2 条为不合法事件。原始文件 SHA-256 为 0d93bcd25e28047f804238c220bdb5b1d2dadf328647c07de85ba0104e8e2854。数据证据文件为 d54def73c9654930b905851d5fbb3b82-probe-02.html。



JSONL 字段包括：event\_id，事件业务身份标识，用于去重和幂等处理；event\_time，事件发生的 UTC 时间，用于按天聚合和晚到事件分析；service，取值为 assistant、search 或 batch，用于服务用量汇总；model，取值为 small 或 medium，用于模型维度统计；input\_tokens 和 output\_tokens，输入与输出 Token 数，用于用量计算与质量检查；latency\_ms，单次请求耗时，单位毫秒，用于性能统计；status，取值为 ok 或 error，用于成功与失败比例统计。



四、原理预测



核心预测在运行前写出：将相同字节上传到两个不同对象键，内容 SHA-256 不变。理由是 SHA-256 计算的是对象字节内容，对象键只是对象在桶中的名称和位置，不参与字节序列。实测结果与预测一致：改键后 source\_sha256 和 downloaded\_sha256 均保持 23c0b20c65b4476c7e541044c09c88cd970bb02191681f804daef0b7ba49f0e8 不变，equal 仍为 true。



可选对照两项。将 JSON 中一个数字改为另一个数字，哈希改变，因为字节改变；实测把 requests 从 3 改为 4 后，哈希变为 5657a58e4a955dcf741239aed81c7a600ca434c4b6c738557b8683d12b09d3f6。只改变 JSON 缩进但数据值不变时，哈希同样会改变，因为字节改变而语义不变；这一项本实验未单独运行，按原理推断成立。



云 API 模拟的能力边界是：Moto 在本机以 HTTP 模拟 S3 接口，验证的是客户端请求与字节往返，不验证真实云耐久性、跨机容灾、真实 IAM 和账单。



五、运行结果



桶为 bdp-teaching。对象键为 bronze/events/0d93bcd25e28047f804238c220bdb5b1d2dadf328647c07de85ba0104e8e2854.jsonl。endpoint 为 http://127.0.0.1:60830，这是本次运行自动分配的端口，每次运行端口不同。原始 SHA-256 为 0d93bcd25e28047f804238c220bdb5b1d2dadf328647c07de85ba0104e8e2854。下载 SHA-256 与原始相同，也是 0d93bcd25e28047f804238c220bdb5b1d2dadf328647c07de85ba0104e8e2854。相等判定为 hash\_matches 等于 true，version\_ids\_differ 等于 true。证据文件为 da49f71c732f4437aa36e1f20ed39612-verify.html，最终确认运行 run\_id 为 92312d649f3747f4b85ebc9104d1116c。



六、独立修改



make\_key 函数的前后差异只有一行。改前返回 shared/raw/demo.json，改后返回 students/2410311306/raw/demo.json。这样修改的原因是：共享前缀 shared 下所有同学写同一个键会互相覆盖；改为含本人学号的前缀后，每个提交者的对象键唯一，互不覆盖。



payload 改动为：将 requests 从 3 改为 4，tokens 保持 120 不变。



三次真实结果如下。第一次为已登记本人学号的附件基线，对象键 shared/raw/demo.json，内容 SHA-256 为 23c0b20c65b4476c7e541044c09c88cd970bb02191681f804daef0b7ba49f0e8，下载一致为 true，解释为共享前缀基线、上传下载同字节。第二次为改对象键，对象键 students/2410311306/raw/demo.json，内容 SHA-256 仍为 23c0b20c65b4476c7e541044c09c88cd970bb02191681f804daef0b7ba49f0e8，下载一致为 true，解释为键不参与字节所以哈希不变。第三次为改 payload，对象键保持 students/2410311306/raw/demo.json，内容 SHA-256 变为 5657a58e4a955dcf741239aed81c7a600ca434c4b6c738557b8683d12b09d3f6，下载一致仍为 true，解释为字节改变哈希必变、上传下载仍是同一份新字节所以相等。



diff 已存档在 submissions/lab01/student\_object\_roundtrip.diff。



七、失败恢复



无效 endpoint 为 http://127.0.0.1:1。真实错误为 EndpointConnectionError，Could not connect to the endpoint URL，http://127.0.0.1:1/。退出码为 1，外层 run\_id 为 4baabaf162ce436982efaed9ee2d44ae，内层失败 run\_id 为 68a11ea3643747248b9afd41c4015f9c。



失败原因是：127.0.0.1:1 上没有服务监听，TCP 连接被拒绝。这不是凭据问题，也不是数据质量问题，而是客户端没有连上指定地址的 S3 服务。



修复与重新执行的方法是：去掉 endpoint 参数，返回默认临时服务路径运行 coursekit lab01。恢复成功，新 run\_id 为 9e3a53487b77439c97bd5239feb8b5a2，hash\_matches 等于 true，version\_ids\_differ 等于 true。



八、AI 使用披露



本报告未使用 AI 辅助。



九、总结与交接



实验 02 将读取 data/raw/events.jsonl，共 20102 行。尚未处理的数据质量问题包括 100 条完全重复事件和 2 条不合法事件，需要在实验 02 的质量报告中治理。本实验只观察异常，没有手工删改原始文件，保留原始数据以便下一章证明质量治理过程真实发生。



十、能力边界



本地 HTTP 请求，本实验验证，证据是 Moto 服务日志显示 PUT、GET、HEAD 请求返回 200。S3 接口，本实验验证，证据是 boto3 对 Moto 执行 create\_bucket、put\_object、download\_file。字节一致，本实验验证，证据是原始与下载 SHA-256 相等且 hash\_matches 为 true。业务合法，本实验未验证，上传成功不代表事件业务合法，由实验 02 处理。云耐久性，本实验未验证，Moto 为内存易失，服务停止后对象消失。跨机容灾，本实验未验证，仅本机回环通信。真实访问控制，本实验未验证，使用的是本地测试凭据，没有真实 IAM。以上未验证项不是失败，而是本模拟实验不覆盖的范围；也不把模拟成功写成生产保证。



十一、分析题



第一题，将相同数据保存为两个不同对象键，为什么不一定改变 SHA-256？

答：SHA-256 只对对象字节内容计算，对象键是名称和位置，不参与字节序列，字节相同则摘要相同。



第二题，JSON 仅调整空格时，字节一致性与业务含义一致性是否相同？

答：不同。调整空格会改变字节，SHA-256 会变；但 JSON 语义值不变，业务含义一致。字节一致不等于语义一致。



第三题，上传一个 input\_tokens 为负一的事件文件成功，能证明什么，不能证明什么？

答：能证明 HTTP 和 S3 接口成功、字节可以往返；不能证明事件业务合法，负 Token 仍需质量校验，由实验 02 处理。



第四题，为什么临时 Moto 服务停止后，本课程仍能开展实验 02？

答：实验 02 读取本地 data/raw/events.jsonl，不依赖 Moto 服务继续运行；本地原始文件是跨实验保留的教学源。



第五题，如果改用真实云对象存储，还需要验证哪些身份、权限、持久性与成本条件？

答：需要验证真实 IAM 身份与最小权限、桶策略与访问控制、版本与生命周期、跨区持久性与容灾、加密，以及计费和成本告警等条件。

