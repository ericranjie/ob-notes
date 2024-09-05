![](https://s2.51cto.com/images/100/blog/share_default.jpeg?x-oss-process=image/ignore-error,1)

# 异源数据同步 → DataX 为什么要支持 kafka？

 精选 原创

[我是青石路](https://blog.51cto.com/u_13423706)2024-09-04 16:06:21

**_文章标签_[kafka](https://blog.51cto.com/topic/kafka.html)[ide](https://blog.51cto.com/search/result?q=ide)[数据源](https://blog.51cto.com/search/result?q=%E6%95%B0%E6%8D%AE%E6%BA%90)****_文章分类_[Python](https://blog.51cto.com/nav/python)[后端开发](https://blog.51cto.com/nav/program)****_阅读数_**173****

## 开心一刻

昨天发了一条朋友圈：酒吧有什么好去的，上个月在酒吧当服务员兼职，一位大姐看上了我，说一个月给我 10 万，要我陪她去上海，我没同意

朋友评论道：你没同意，为什么在上海？

我回复到：上个月没同意

![异源数据同步 → DataX 为什么要支持 kafka？_kafka](https://s2.51cto.com/images/blog/202408/31033453_66d21edd8cd2d81511.gif "嘴真硬")

## 前情回顾

关于  [DataX](https://gitee.com/mirrors/DataX)，官网有很详细的介绍，鄙人不才，也写过几篇文章

> 异构数据源同步之数据同步 → datax 改造，有点意思
> 
> 异构数据源同步之数据同步 → datax 再改造，开始触及源码
> 
> 异构数据源同步之数据同步 → DataX 使用细节
> 
> 异构数据源数据同步 → 从源码分析 DataX 敏感信息的加解密

不了解的小伙伴可以按需去查看，所以了，`DataX` 就不做过多介绍了；官方提供了非常多的插件，囊括了绝大部分的数据源，基本可以满足我们日常需要，但数据源种类太多，DataX 插件不可能包含全部，比如 `kafka`，DataX 官方是没有提供读写插件的，大家知道为什么吗？你们如果对数据同步了解的比较多的话，一看到 kafka，第一反应往往想到的是 `实时同步`，而 DataX 针对的是 `离线同步`，所以 DataX 官方没提供 kafka 插件是不是也就能理解了？因为不合适嘛！

但如果客户非要离线同步也支持 kafka

![异源数据同步 → DataX 为什么要支持 kafka？_ide_02](https://s2.51cto.com/images/blog/202408/31033453_66d21eddb0a6e10965.gif "人家要嘛")

你能怎么办？直接怼过去：实现不了？

![异源数据同步 → DataX 为什么要支持 kafka？_数据源_03](https://s2.51cto.com/images/blog/202408/31033453_66d21eddc5cb165067.jpg?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/resize,m_fixed,w_1184 "实现不了")

所以没得选，那就只能给 DataX 开发一套 kafka 插件了；基于  [DataX插件开发宝典](https://gitee.com/mirrors/DataX/blob/master/dataxPluginDev.md)，插件开发起来还是非常简单的

## kafkawriter

1. 编程接口  
    自定义 `Kafkawriter` 继承 DataX 的 `Writer`，实现 job、task 对应的接口即可

```plain
/**
 * @author 青石路
 */
public class KafkaWriter extends Writer {

    public static class Job extends Writer.Job {

        private Configuration conf = null;

        @Override
        public List<Configuration> split(int mandatoryNumber) {
            List<Configuration> configurations = new ArrayList<Configuration>(mandatoryNumber);
            for (int i = 0; i < mandatoryNumber; i++) {
                configurations.add(this.conf.clone());
            }
            return configurations;
        }

        private void validateParameter() {
            this.conf.getNecessaryValue(Key.BOOTSTRAP_SERVERS, KafkaWriterErrorCode.REQUIRED_VALUE);
            this.conf.getNecessaryValue(Key.TOPIC, KafkaWriterErrorCode.REQUIRED_VALUE);
        }

        @Override
        public void init() {
            this.conf = super.getPluginJobConf();
            this.validateParameter();
        }


        @Override
        public void destroy() {

        }
    }

    public static class Task extends Writer.Task {
        private static final Logger logger = LoggerFactory.getLogger(Task.class);
        private static final String NEWLINE_FLAG = System.getProperty("line.separator", "\n");

        private Producer<String, String> producer;
        private Configuration conf;
        private Properties props;
        private String fieldDelimiter;
        private List<String> columns;
        private String writeType;

        @Override
        public void init() {
            this.conf = super.getPluginJobConf();
            fieldDelimiter = conf.getUnnecessaryValue(Key.FIELD_DELIMITER, "\t", null);
            columns = conf.getList(Key.COLUMN, String.class);
            writeType = conf.getUnnecessaryValue(Key.WRITE_TYPE, WriteType.TEXT.name(), null);
            if (CollUtil.isEmpty(columns)) {
                throw DataXException.asDataXException(KafkaWriterErrorCode.REQUIRED_VALUE,
                        String.format("您提供配置文件有误，[%s]是必填参数，不允许为空或者留白 .", Key.COLUMN));
            }

            props = new Properties();
            props.put(CommonClientConfigs.BOOTSTRAP_SERVERS_CONFIG, conf.getString(Key.BOOTSTRAP_SERVERS));
            //这意味着leader需要等待所有备份都成功写入日志，这种策略会保证只要有一个备份存活就不会丢失数据。这是最强的保证。
            props.put(ProducerConfig.ACKS_CONFIG, conf.getUnnecessaryValue(Key.ACK, "0", null));
            props.put(CommonClientConfigs.RETRIES_CONFIG, conf.getUnnecessaryValue(Key.RETRIES, "0", null));
            props.put(ProducerConfig.BATCH_SIZE_CONFIG, conf.getUnnecessaryValue(Key.BATCH_SIZE, "16384", null));
            props.put(ProducerConfig.LINGER_MS_CONFIG, 1);
            props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, conf.getUnnecessaryValue(Key.KEY_SERIALIZER, "org.apache.kafka.common.serialization.StringSerializer", null));
            props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, conf.getUnnecessaryValue(Key.VALUE_SERIALIZER, "org.apache.kafka.common.serialization.StringSerializer", null));

            Configuration saslConf = conf.getConfiguration(Key.SASL);
            if (ObjUtil.isNotNull(saslConf)) {
                logger.info("配置启用了SASL认证");
                props.put(CommonClientConfigs.SECURITY_PROTOCOL_CONFIG, saslConf.getNecessaryValue(Key.SASL_SECURITY_PROTOCOL, KafkaWriterErrorCode.REQUIRED_VALUE));
                props.put(SaslConfigs.SASL_MECHANISM, saslConf.getNecessaryValue(Key.SASL_MECHANISM, KafkaWriterErrorCode.REQUIRED_VALUE));
                String userName = saslConf.getNecessaryValue(Key.SASL_USERNAME, KafkaWriterErrorCode.REQUIRED_VALUE);
                String password = saslConf.getNecessaryValue(Key.SASL_PASSWORD, KafkaWriterErrorCode.REQUIRED_VALUE);
                props.put(SaslConfigs.SASL_JAAS_CONFIG, String.format("org.apache.kafka.common.security.plain.PlainLoginModule required username=\"%s\" password=\"%s\";", userName, password));
            }

            producer = new KafkaProducer<String, String>(props);
        }

        @Override
        public void prepare() {
            if (Boolean.parseBoolean(conf.getUnnecessaryValue(Key.NO_TOPIC_CREATE, "false", null))) {

                ListTopicsResult topicsResult = AdminClient.create(props).listTopics();
                String topic = conf.getNecessaryValue(Key.TOPIC, KafkaWriterErrorCode.REQUIRED_VALUE);

                try {
                    if (!topicsResult.names().get().contains(topic)) {
                        new NewTopic(
                                topic,
                                Integer.parseInt(conf.getUnnecessaryValue(Key.TOPIC_NUM_PARTITION, "1", null)),
                                Short.parseShort(conf.getUnnecessaryValue(Key.TOPIC_REPLICATION_FACTOR, "1", null))
                        );
                        List<NewTopic> newTopics = new ArrayList<NewTopic>();
                        AdminClient.create(props).createTopics(newTopics);
                    }
                } catch (Exception e) {
                    throw new DataXException(KafkaWriterErrorCode.CREATE_TOPIC, KafkaWriterErrorCode.REQUIRED_VALUE.getDescription());
                }
            }
        }

        @Override
        public void startWrite(RecordReceiver lineReceiver) {
            logger.info("start to writer kafka");
            Record record = null;
            while ((record = lineReceiver.getFromReader()) != null) {//说明还在读取数据,或者读取的数据没处理完
                //获取一行数据，按照指定分隔符 拼成字符串 发送出去
                if (writeType.equalsIgnoreCase(WriteType.TEXT.name())) {
                    producer.send(new ProducerRecord<String, String>(this.conf.getString(Key.TOPIC),
                            recordToString(record),
                            recordToString(record))
                    );
                } else if (writeType.equalsIgnoreCase(WriteType.JSON.name())) {
                    producer.send(new ProducerRecord<String, String>(this.conf.getString(Key.TOPIC),
                            recordToString(record),
                            recordToKafkaJson(record))
                    );
                }
                producer.flush();
            }
        }

        @Override
        public void destroy() {
            logger.info("producer close");
            if (producer != null) {
                producer.close();
            }
        }

        /**
         * 数据格式化
         *
         * @param record
         * @return
         */
        private String recordToString(Record record) {
            int recordLength = record.getColumnNumber();
            if (0 == recordLength) {
                return NEWLINE_FLAG;
            }
            Column column;
            StringBuilder sb = new StringBuilder();
            for (int i = 0; i < recordLength; i++) {
                column = record.getColumn(i);
                sb.append(column.asString()).append(fieldDelimiter);
            }

            sb.setLength(sb.length() - 1);
            sb.append(NEWLINE_FLAG);

            return sb.toString();
        }

        private String recordToKafkaJson(Record record) {
            int recordLength = record.getColumnNumber();
            if (recordLength != columns.size()) {
                throw DataXException.asDataXException(KafkaWriterErrorCode.ILLEGAL_PARAM,
                        String.format("您提供配置文件有误，列数不匹配[record columns=%d, writer columns=%d]", recordLength, columns.size()));
            }
            List<KafkaColumn> kafkaColumns = new ArrayList<>();
            for (int i = 0; i < recordLength; i++) {
                KafkaColumn column = new KafkaColumn(record.getColumn(i), columns.get(i));
                kafkaColumns.add(column);
            }
            return JSONUtil.toJsonStr(kafkaColumns);
        }
    }
}
```

DataX 框架按照如下的顺序执行 Job 和 Task 的接口

![异源数据同步 → DataX 为什么要支持 kafka？_kafka_04](https://s2.51cto.com/images/blog/202408/31033453_66d21edddb1ad65824.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/resize,m_fixed,w_1184 "job_task 接口执行顺序")

重点看 Task 的接口实现

- init：读取配置项，然后创建 Producer 实例
- prepare：判断 Topic 是否存在，不存在则创建
- startWrite：通过 RecordReceiver 从 Channel 获取 Record，然后写入 Topic  
    支持两种写入格式：`text`、`json`，细节请看下文中的 `kafkawriter.md`
- destroy：关闭 Producer 实例

实现不难，相信大家都能看懂

1. 插件定义  
    在 `resources` 下新增 `plugin.json`

```plain
{
    "name": "kafkawriter",
    "class": "com.qsl.datax.plugin.writer.kafkawriter.KafkaWriter",
    "description": "write data to kafka",
    "developer": "qsl"
}
```

强调下 `class`，是 `KafkaWriter` 的全限定类名，如果你们没有完全拷贝我的，那么要改成你们自己的

1. 配置文件  
    在 `resources` 下新增 `plugin_job_template.json`

```plain
{
    "name": "kafkawriter",
    "parameter": {
        "bootstrapServers": "",
        "topic": "",
        "ack": "all",
        "batchSize": 1000,
        "retries": 0,
        "fieldDelimiter": ",",
        "writeType": "json",
        "column": [
            "const_id",
            "const_field",
            "const_field_value"
        ],
        "sasl": {
            "securityProtocol": "SASL_PLAINTEXT",
            "mechanism": "PLAIN",
            "username": "",
            "password": ""
        }
    }
}
```

配置项说明： [kafkawriter.md](https://gitee.com/youzhibing/qsl-datax/blob/master/qsl-datax-plugin/kafkawriter/doc/kafkawriter.md)

1. 打包发布  
    可以参考官方的 `assembly` 配置，利用 assembly 来打包

至此，`kafkawriter` 就算完成了

## kafkareader

1. 编程接口  
    自定义 `Kafkareader` 继承 DataX 的 `Reader`，实现 job、task 对应的接口即可

```plain
/**
 * @author 青石路
 */
public class KafkaReader extends Reader {

    public static class Job extends Reader.Job {

        private Configuration originalConfig = null;

        @Override
        public void init() {
            this.originalConfig = super.getPluginJobConf();
            this.validateParameter();
        }

        @Override
        public void destroy() {

        }

        @Override
        public List<Configuration> split(int adviceNumber) {
            List<Configuration> configurations = new ArrayList<>(adviceNumber);
            for (int i=0; i<adviceNumber; i++) {
                configurations.add(this.originalConfig.clone());
            }
            return configurations;
        }

        private void validateParameter() {
            this.originalConfig.getNecessaryValue(Key.BOOTSTRAP_SERVERS, KafkaReaderErrorCode.REQUIRED_VALUE);
            this.originalConfig.getNecessaryValue(Key.TOPIC, KafkaReaderErrorCode.REQUIRED_VALUE);
        }
    }

    public static class Task extends Reader.Task {

        private static final Logger logger = LoggerFactory.getLogger(Task.class);

        private Consumer<String, String> consumer;
        private String topic;
        private Configuration conf;
        private int maxPollRecords;
        private String fieldDelimiter;
        private String readType;
        private List<Column.Type> columnTypes;

        @Override
        public void destroy() {
            logger.info("consumer close");
            if (Objects.nonNull(consumer)) {
                consumer.close();
            }
        }

        @Override
        public void init() {
            this.conf = super.getPluginJobConf();
            this.topic = conf.getString(Key.TOPIC);
            this.maxPollRecords = conf.getInt(Key.MAX_POLL_RECORDS, 500);
            fieldDelimiter = conf.getUnnecessaryValue(Key.FIELD_DELIMITER, "\t", null);
            readType = conf.getUnnecessaryValue(Key.READ_TYPE, ReadType.JSON.name(), null);
            if (!ReadType.JSON.name().equalsIgnoreCase(readType)
                    && !ReadType.TEXT.name().equalsIgnoreCase(readType)) {
                throw DataXException.asDataXException(KafkaReaderErrorCode.REQUIRED_VALUE,
                        String.format("您提供配置文件有误，不支持的readType[%s]", readType));
            }
            if (ReadType.JSON.name().equalsIgnoreCase(readType)) {
                List<String> columnTypeList = conf.getList(Key.COLUMN_TYPE, String.class);
                if (CollUtil.isEmpty(columnTypeList)) {
                    throw DataXException.asDataXException(KafkaReaderErrorCode.REQUIRED_VALUE,
                            String.format("您提供配置文件有误，readType是JSON时[%s]是必填参数，不允许为空或者留白 .", Key.COLUMN_TYPE));
                }
                convertColumnType(columnTypeList);
            }
            Properties props = new Properties();
            props.put(CommonClientConfigs.BOOTSTRAP_SERVERS_CONFIG, conf.getString(Key.BOOTSTRAP_SERVERS));
            props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, conf.getUnnecessaryValue(Key.KEY_DESERIALIZER, "org.apache.kafka.common.serialization.StringDeserializer", null));
            props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, conf.getUnnecessaryValue(Key.VALUE_DESERIALIZER, "org.apache.kafka.common.serialization.StringDeserializer", null));
            props.put(ConsumerConfig.GROUP_ID_CONFIG, conf.getNecessaryValue(Key.GROUP_ID, KafkaReaderErrorCode.REQUIRED_VALUE));
            props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");
            props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
            props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, maxPollRecords);
            Configuration saslConf = conf.getConfiguration(Key.SASL);
            if (ObjUtil.isNotNull(saslConf)) {
                logger.info("配置启用了SASL认证");
                props.put(CommonClientConfigs.SECURITY_PROTOCOL_CONFIG, saslConf.getNecessaryValue(Key.SASL_SECURITY_PROTOCOL, KafkaReaderErrorCode.REQUIRED_VALUE));
                props.put(SaslConfigs.SASL_MECHANISM, saslConf.getNecessaryValue(Key.SASL_MECHANISM, KafkaReaderErrorCode.REQUIRED_VALUE));
                String userName = saslConf.getNecessaryValue(Key.SASL_USERNAME, KafkaReaderErrorCode.REQUIRED_VALUE);
                String password = saslConf.getNecessaryValue(Key.SASL_PASSWORD, KafkaReaderErrorCode.REQUIRED_VALUE);
                props.put(SaslConfigs.SASL_JAAS_CONFIG, String.format("org.apache.kafka.common.security.plain.PlainLoginModule required username=\"%s\" password=\"%s\";", userName, password));
            }
            consumer = new KafkaConsumer<>(props);
        }

        @Override
        public void startRead(RecordSender recordSender) {
            consumer.subscribe(CollUtil.newArrayList(topic));
            int pollTimeoutMs = conf.getInt(Key.POLL_TIMEOUT_MS, 1000);
            int retries = conf.getInt(Key.RETRIES, 5);
            if (retries < 0) {
                logger.info("joinGroupSuccessRetries 配置有误[{}], 重置成默认值[5]", retries);
                retries = 5;
            }
            /**
             * consumer 每次都是新创建，第一次poll时会重新加入消费者组，加入过程会进行Rebalance，而 Rebalance 会导致同一 Group 内的所有消费者都不能工作
             * 所以 poll 拉取的过程中，即使topic中有数据也不一定能拉到，因为 consumer 正在加入消费者组中
             * kafka-clients 没有对应的API、事件机制来知道 consumer 成功加入消费者组的确切时间
             * 故增加重试
             */
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(pollTimeoutMs));
            int i = 0;
            if (CollUtil.isEmpty(records)) {
                for (; i < retries; i++) {
                    records = consumer.poll(Duration.ofMillis(pollTimeoutMs));
                    logger.info("第 {} 次重试，获取消息记录数[{}]", i + 1, records.count());
                    if (!CollUtil.isEmpty(records)) {
                        break;
                    }
                }
            }
            if (i >= retries) {
                logger.info("重试 {} 次后，仍未获取到消息，请确认是否有数据、配置是否正确", retries);
                return;
            }
            transferRecord(recordSender, records);
            do {
                records = consumer.poll(Duration.ofMillis(pollTimeoutMs));
                transferRecord(recordSender, records);
            } while (!CollUtil.isEmpty(records) && records.count() >= maxPollRecords);
        }

        private void transferRecord(RecordSender recordSender, ConsumerRecords<String, String> records) {
            if (CollUtil.isEmpty(records)) {
                return;
            }
            for (ConsumerRecord<String, String> record : records) {
                Record sendRecord = recordSender.createRecord();
                String msgValue = record.value();
                if (ReadType.JSON.name().equalsIgnoreCase(readType)) {
                    transportJsonToRecord(sendRecord, msgValue);
                } else if (ReadType.TEXT.name().equalsIgnoreCase(readType)) {
                    // readType = text，全当字符串类型处理
                    String[] columnValues = msgValue.split(fieldDelimiter);
                    for (String columnValue : columnValues) {
                        sendRecord.addColumn(new StringColumn(columnValue));
                    }
                }
                recordSender.sendToWriter(sendRecord);
            }
            consumer.commitAsync();
        }

        private void convertColumnType(List<String> columnTypeList) {
            columnTypes = new ArrayList<>();
            for (String columnType : columnTypeList) {
                switch (columnType.toUpperCase()) {
                    case "STRING":
                        columnTypes.add(Column.Type.STRING);
                        break;
                    case "LONG":
                        columnTypes.add(Column.Type.LONG);
                        break;
                    case "DOUBLE":
                        columnTypes.add(Column.Type.DOUBLE);
                    case "DATE":
                        columnTypes.add(Column.Type.DATE);
                        break;
                    case "BOOLEAN":
                        columnTypes.add(Column.Type.BOOL);
                        break;
                    case "BYTES":
                        columnTypes.add(Column.Type.BYTES);
                        break;
                    default:
                        throw DataXException.asDataXException(KafkaReaderErrorCode.ILLEGAL_PARAM,
                                String.format("您提供的配置文件有误，datax不支持数据类型[%s]", columnType));
                }
            }
        }

        private void transportJsonToRecord(Record sendRecord, String msgValue) {
            List<KafkaColumn> kafkaColumns = JSONUtil.toList(msgValue, KafkaColumn.class);
            if (columnTypes.size() != kafkaColumns.size()) {
                throw DataXException.asDataXException(KafkaReaderErrorCode.ILLEGAL_PARAM,
                        String.format("您提供的配置文件有误，readType是JSON时[%s列数=%d]与[json列数=%d]的数量不匹配", Key.COLUMN_TYPE, columnTypes.size(), kafkaColumns.size()));
            }
            for (int i=0; i<columnTypes.size(); i++) {
                KafkaColumn kafkaColumn = kafkaColumns.get(i);
                switch (columnTypes.get(i)) {
                    case STRING:
                        sendRecord.setColumn(i, new StringColumn(kafkaColumn.getColumnValue()));
                        break;
                    case LONG:
                        sendRecord.setColumn(i, new LongColumn(kafkaColumn.getColumnValue()));
                        break;
                    case DOUBLE:
                        sendRecord.setColumn(i, new DoubleColumn(kafkaColumn.getColumnValue()));
                        break;
                    case DATE:
                        // 暂只支持时间戳
                        sendRecord.setColumn(i, new DateColumn(Long.parseLong(kafkaColumn.getColumnValue())));
                        break;
                    case BOOL:
                        sendRecord.setColumn(i, new BoolColumn(kafkaColumn.getColumnValue()));
                        break;
                    case BYTES:
                        sendRecord.setColumn(i, new BytesColumn(kafkaColumn.getColumnValue().getBytes(StandardCharsets.UTF_8)));
                        break;
                    default:
                        throw DataXException.asDataXException(KafkaReaderErrorCode.ILLEGAL_PARAM,
                                String.format("您提供的配置文件有误，datax不支持数据类型[%s]", columnTypes.get(i)));
                }
            }
        }
    }
}
```

重点看 Task 的接口实现

- init：读取配置项，然后创建 Consumer 实例
- startWrite：从 Topic 拉取数据，通过 RecordSender 写入到 Channel 中  
    这里有几个细节需要注意下

1. Consumer 每次都是新创建的，拉取数据的时候，如果消费者还未加入到指定的消费者组中，那么它会先加入到消费者组中，加入过程会进行 Rebalance，而 Rebalance 会导致同一消费者组内的所有消费者都不能工作，此时即使 Topic 中有可拉取的消息，也拉取不到消息，所以引入了重试机制来尽量保证那一次同步任务拉取的时候，消费者能正常拉取消息
2. 一旦 Consumer 拉取到消息，则会循环拉取消息，如果某一次的拉取数据量小于最大拉取量（maxPollRecords），说明 Topic 中的消息已经被拉取完了，那么循环终止；这与常规使用（Consumer 会一直主动拉取或被动接收）是有差别的
3. 支持两种读取格式：`text`、`json`，细节请看下文的配置文件说明
4. 为了保证写入 Channel 数据的完整，需要配置列的数据类型（DataX 的数据类型）

- destroy：  
    关闭 Consumer 实例

1. 插件定义  
    在 `resources` 下新增 `plugin.json`

```plain
{
    "name": "kafkareader",
    "class": "com.qsl.datax.plugin.reader.kafkareader.KafkaReader",
    "description": "read data from kafka",
    "developer": "qsl"
}
```

`class` 是 `KafkaReader` 的全限定类名

1. 配置文件  
    在 `resources` 下新增 `plugin_job_template.json`

```plain
{
    "name": "kafkareader",
    "parameter": {
        "bootstrapServers": "",
        "topic": "test-kafka",
        "groupId": "test1",
        "writeType": "json",
        "pollTimeoutMs": 2000,
        "columnType": [
            "LONG",
            "STRING",
            "STRING"
        ],
        "sasl": {
            "securityProtocol": "SASL_PLAINTEXT",
            "mechanism": "PLAIN",
            "username": "",
            "password": "2"
        }
    }
}
```

配置项说明： [kafkareader.md](https://gitee.com/youzhibing/qsl-datax/blob/master/qsl-datax-plugin/kafkareader/doc/kafkareader.md)

1. 打包发布  
    可以参考官方的 `assembly` 配置，利用 assembly 来打包

至此，`kafkareader` 也完成了

## 总结

1. 完整代码： [qsl-datax](https://gitee.com/youzhibing/qsl-datax)
2. kafkareader 重试机制只能降低拉取不到数据的概率，并不能杜绝；另外，如果上游一直往 Topic 中发消息，kafkareader 每次拉取的数据量都等于最大拉取量，那么同步任务会一直进行而不会停止，这还是离线同步吗？
3. 离线同步，不推荐走 kafka，因为用 kafka 走实时同步更香

  

  

- **赞**
 - **收藏**
 - **评论**
 - **分享**
 - **举报**

上一篇：[Kafka Topic 中明明有可拉取的消息，为什么 poll 不到](https://blog.51cto.com/u_13423706/11918688)

[![](https://ucenter.51cto.com/images/noavatar_middle.gif)](https://blog.51cto.com/)

提问和评论都可以，用心的回复会被更多人看到**评论**

**相关文章**

- [
    
    关于【Python转技术管理】的项目实战学习资料（附讲解～～）
    
    详细讲解了有关【技术管理】在项目实战中的处理方式以及价值点，并且附带了相相应的内容讲解，以便大家可以更好的处理技术管理相关的问题以及学习处理这类问题的最优解
    
    ](https://d.51cto.com/eDOcp1)
    
    技术管理
    
- [
    
    Azure Files – 它是什么？我为什么要它？
    
    什么是 Azure 文件存储？Azure 文件是位于 Azure 存储帐户上的云中的完全托管文件共享。Azure 文件共享，可通过 SMB、NFS 和 FileREST 协议进行访问。Azure 文件共享可由 Azure VM 中的客户端或运行 Windows、macOS 和 Linux 的本地工作站同时装载。此外，Azure 文件同步可用于缓存和同步 Windows 服务器上的 Azure 文
    
    ](https://blog.51cto.com/fjcloud/10268436)
    
    Azure 文件共享 文件存储 
    
- [
    
    服务器为什么要定期备份
    
    服务器为什么要定期备份1. 数据保护和恢复：服务器备份是保护数据免受意外数据丢失、硬件故障、人为错误、恶意攻击等因素影响的关键措施。通过定期备份，可以将服务器上的数据复制到另一个位置或媒体中，以便在发生数据丢失或损坏时能够进行快速恢复。备份可以帮助恢复关键数据和配置，减少业务中断时间，保证业务的连续性和可用性。2. 灾难恢复：备份是灾难恢复计划的核心组成部分。当服务器遭受灾难性事件（如自然灾害
    
    ](https://blog.51cto.com/u_15748830/10480744)
    
    数据 服务器 开发环境 
    
- [
    
    什么是Token？为什么大模型要计算Token数
    
    大模型不是直接做的“字符”的计算，而是将字符变成一个数字，也就是变成了 token 来处理。
    
    ](https://blog.51cto.com/u_15214399/10947027)
    
    API Token 大模型 LLM 
    
- [
    
    datax支持mysqlbinlog增量数据同步
    
    # DataX 实现 MySQL Binlog 增量数据同步指南在当今的数据处理环境中，增量数据同步成为了实时数据处理的一项重要需求。DataX 是一个通用数据同步工具，可以有效地帮助我们实现 MySQL Binlog 的增量数据同步。本文将详细介绍如何使用 DataX 支持 MySQL Binlog 的增量数据同步，内容将包括整体流程、每一步的代码示例及解释。## 整体流程下面是实现
    
    ](https://blog.51cto.com/u_16213442/11905451)
    
    MySQL 数据同步 mysql 
    
- [
    
    datax支持hive数据同步吗
    
    ### 数据同步工具DataX对Hive的支持在大数据领域中，数据同步工具是必不可缺的工具之一。而DataX作为阿里巴巴开源的一款高性能数据同步工具，备受关注。那么，对于Hive这样的大数据存储系统，DataX是否支持数据同步呢？本文将为您介绍DataX对Hive数据同步的支持情况。### DataX支持Hive数据同步首先，我们需要明确的是，DataX是支持对Hive数据的同步的。D
    
    ](https://blog.51cto.com/u_16213324/9772593)
    
    Hive 数据同步 数据 
    
- [
    
    为什么要进行数据同步
    
    整个类。这类变量可以被多个线程共享。因此，我们需要对这类变量进行数据同步。   ...
    
    ](https://blog.51cto.com/u_16193056/6777869)
    
    thread 工作 Java 类变量 脏数据 
    
- [
    
    异构数据源同步之数据同步 → DataX 使用细节
    
    开心一刻 中午我妈微信给我消息 妈：儿子啊，妈电话欠费了，能帮妈充个话费吗 我：妈，我知道了，我帮你充 当我帮我
    
    ](https://blog.51cto.com/u_13423706/11101893)
    
    mysql bc 数据 
    
- [
    
    datax数据同步导kafka datax大数据同步
    
    概述DataX 是阿里巴巴集团内被广泛使用的离线数据同步工具/平台，实现包括 MySQL、Oracle、SqlServer、Postgre、HDFS、Hive、ADS、HBase、TableStore(OTS)、MaxCompute(ODPS)、DRDS 等各种异构数据源之间高效的数据同步功能。DataX本身作为数据同步框架，将不同数据源的同步抽象为从源头数据源读取数据的Reader插件，以及向目
    
    ](https://blog.51cto.com/u_16213641/11190978)
    
    datax数据同步导kafka json python 数据库 数据源 
    
- [
    
    datax 可以同步到kafka吗 datax数据同步原理
    
    1.datax介绍DataX 是阿里云 DataWorks数据集成 的开源版本，在阿里巴巴集团内被广泛使用的离线数据同步工具/平台。DataX 实现了包括 MySQL、Oracle、OceanBase、SqlServer、Postgre、HDFS、Hive、ADS、HBase、TableStore(OTS)、MaxCompute(ODPS)、Hologres、DRDS 等各种异构数据源之间高效的数
    
    ](https://blog.51cto.com/u_16099212/11723696)
    
    datax 可以同步到kafka吗 数据库 大数据 hadoop 数据 
    
- [
    
    datax数据源hive datax支持的数据源
    
    目录一、DataX的简介二、DataX支持的数据源三、架构介绍四、安装与使用同步MySQL数据到HDFS案例同步HDFS数据到MySQL案例一、DataX的简介        DataX 是阿里巴巴开源的一个异构数据源离线同步工具，致力于实现包括关系型数据库(MySQL、Oracle等)、HDFS、Hive、ODPS、
    
    ](https://blog.51cto.com/u_16213725/8536709)
    
    datax数据源hive big data HDFS 数据源 数据 
    
- [
    
    dataX 支持kafka datax配置
    
    DataX 是阿里开源的一个异构数据源离线同步工具,致力于实现包括关系型数据库(MySQL、Oracle等)、HDFS、Hive、ODPS、HBase、FTP等各种异构数据源之间稳定高效的数据同步功能。DataX工具是用json文件作为配置文件的，根据官方提供文档我们构建Json文件如下：{ "job": { "content": [ {
    
    ](https://blog.51cto.com/u_16213607/10288720)
    
    dataX 支持kafka 字符串 数据库 数组 
    
- [
    
    kafka为什么要offset kafka为什么要zeekeeper
    
    前言：在我们了解kafka为什么依赖zookeeper之前，首先要先知道zookeeper自身的一个基础架构和作用“所有一切的努力都是为了自己的名字”Zookeeper概念扫盲基本概述ZooKeeper是一个分布式协调服务，它的主要作用是为分布式系统提供一致性服务数据结构 ZooKeeper的数据存储也同样是基于节点，这种节点叫做Znode。每一个Znode里包含了数据、子节点引用、访问
    
    ](https://blog.51cto.com/u_16213597/9675737)
    
    kafka为什么要offset zookeeper 客户端 数据 
    
- [
    
    datax 支持hive吗 datax支持的数据源
    
    一.datax介绍DataX 是阿里云 DataWorks数据集成 的开源版本，在阿里巴巴集团内被广泛使用的离线数据同步工具/平台。DataX 实现了包括 MySQL、Oracle、OceanBase、SqlServer、Postgre、HDFS、Hive、ADS、HBase、TableStore(OTS)、MaxCompute(ODPS)、Hologres、DRDS, databend 等各种异
    
    ](https://blog.51cto.com/u_14230/8804291)
    
    datax 支持hive吗 大数据 Powered by 金山文档 数据源 html 
    
- [
    
    datax支持kafka datax支持excel吗
    
    DataX是阿里巴巴开源的一个异构数据源离线同步工具，主要用于实现各种异构数据源之间稳定高效的数据同步功能。以下是关于DataX的详细阐述：设计理念和架构：DataX的设计理念是将复杂的网状的同步链路变成星型数据链路，它作为中间传输载体负责连接各种数据源。当需要接入一个新的数据源时，只需要将此数据源对接到DataX，就能与已有的数据源实现无缝数据同步。DataX本身作为离线数据同步框架，采用Fra
    
    ](https://blog.51cto.com/u_16213715/11803577)
    
    datax支持kafka database 数据源 数据 数据同步 
    
- [
    
    datax kafka同步至hive kafka同步数据库
    
    1.前言MirrorMaker 是 Kafka官方提供的跨数据中心的流数据同步方案。原理是通过从 原始kafka集群消费消息，然后把消息发送到 目标kafka集群。操作简单，只要通过简单的 consumer配置和 producer配置，然后启动 Mirror，就可以实现准实时的数据同步。2.独立 Kafka集群使用 MirrorMaker2.1 开启远程连接这里需要确保 目标Kafka集群（接收数
    
    ](https://blog.51cto.com/u_12929/8911866)
    
    datax kafka同步至hive kafka java 分布式 大数据 
    
- [
    
    datax 是否支持kafka datax支持的数据库
    
    DataX的使用在接触datax之前，一直用的是Apache Sqoop这个工具，它是用来在Apache Hadoop 和诸如关系型数据库等结构化数据传输大量数据的工具。但是在实际工作中，不同的公司可能会用到不同的nosql数据库和关系型数据库，不一定是基于hadoop的hive，hbase等这些，所以sqoop也有一定的局限性。在工作处理业务中，公司大佬给我推介了阿里巴巴的datax，用完的感受
    
    ](https://blog.51cto.com/u_16099324/10886233)
    
    datax 是否支持kafka 关系型数据库 数据库 github 
    
- [
    
    datax 同步hive到kafka oracle kafka同步大数据
    
    简介： 在大数据时代，存在大量基于数据的业务。数据需要在不同的系统之间流动、整合。通常，核心业务系统的数据存在OLTP数据库系统中，其它业务系统需要获取OLTP系统中的数据。传统的数仓通过批量数据同步的方式，定期从OLTP系统中抽取数据。背景在大数据时代，存在大量基于数据的业务。数据需要在不同的系统之间流动、整合。通常，核心业务系统的数据存在OLTP数据库系统中，其它业务系统需要获取OL
    
    ](https://blog.51cto.com/u_16213640/11138202)
    
    datax 同步hive到kafka 数据 kafka SQL 
    
- [
    
    datax 支持 hive writer datax同步数据到hive
    
    文章目录4. DataX使用4.1 DataX使用概述4.1.1 DataX任务提交命令4.1.2 DataX配置文件格式4.2 同步MySQL数据到HDFS案例4.2.1 MySQLReader之TableMode4.2.1.1 编写配置文件4.2.1.1.1 创建配置文件base_province.json4.2.1.1.2 配置文件内容如下4.2.1.2 配置文件说明4.2.1.2.1 R
    
    ](https://blog.51cto.com/u_16213709/8919631)
    
    数据仓库 flume 大数据 数据库 配置文件 
    
- [
    
    datax是否支持kafka datax canal
    
    DataX 是一个异构数据源离线同步工具，致力于实现包括关系型数据库(MySQL、Oracle等)、HDFS、Hive、ODPS、HBase、FTP等各种异构数据源之间稳定高效的数据同步功能。设计理念为了解决异构数据源同步问题，DataX将复杂的网状的同步链路变成了星型数据链路，DataX作为中间传输载体负责连接各种数据源。当需要接入一个新的数据源的时候，只需要将此数据源对接到DataX，便能跟已
    
    ](https://blog.51cto.com/u_16213580/11502494)
    
    datax是否支持kafka 数据源 数据 数据同步 
    
- [
    
    datax添加hive数据源 datax同步数据到hive
    
    使用DataX和sqoop将数据从MySQL导入Hive一、DataX简述二、sqoop简述三、需求背景四、实现方式3.1 使用DataX将数据从MySQL导入Hive3.2 通过sqoop将数据从MySQL导入Hive四、总结4.1 Datax主要特点4.2 Sqoop主要特点4.3 Sqoop 和 Datax的区别 一、DataX简述DataX 是阿里云 DataWorks数据集成 的开源版
    
    ](https://blog.51cto.com/u_14126/8573638)
    
    datax添加hive数据源 hive sqoop mysql 大数据 
    
- [
    
    hadoop yarn queue 的好处
    
    接着昨天的继续看hadoop-yarn-api，昨天看了api package下的4个协议，今天来看下con package下的代码 conf目录下的内容比较少，就4个文件分别是ConfigurationProvider, ConfigurationProviderFactory,HAUtil以及YarnConfiguration &nbs
    
    ](https://blog.51cto.com/u_16099232/11915544)
    
    ide bootstrap hadoop 
    
- [
    
    flink jobmanage 设置多少
    
    写在前面1. Flink RPC详解Flink使用Akka+Netty框架实现RPC通信，之前在spark框架源码剖析过程中已经对Akka实现RPC通信过程有所介绍，这里不做过多描述。相关概念说明如下：ActorSystem是管理Actor生命周期的组件，Actor是负责进行通信的组件。每一个Actor都有一个MailBox，别的Actor发送给它的消息都首先存储在MailBox中，通过这种方式可
    
    ](https://blog.51cto.com/u_16099325/11915628)
    
    flink 大数据 初始化 RPC 
    
- [
    
    ios 设备连接工具
    
    　　近视手术方式分为两大类，激光和晶体植入。无论哪一种，都是为了近视手术的唯一目的：裸眼视物。通过改变光线折射进入眼睛的角度，从而达到摘镜的目的。　　那近视手术的原理到底是什么?成都爱尔眼科医院屈光专家周进院长带来讲解。　　激光类近视手术　　激光类近视手术，无论用的飞秒激光、准分子激光，其实都是利用了：光传输原理和光爆破原理。　　● 光传输原理　　医生将设定数据(聚焦深度、激光能量等)输入飞秒激光
    
    ](https://blog.51cto.com/u_16213717/11916877)
    
    ios 设备连接工具 其他 数据 
    
- [
    
    目标检测前景概率和背景概率的区别
    
    上期我们一起学了CNN中四种常用的卷积操作 从这期开始，我们开始步入目标检测领域的大门，开始逐步一层一层的揭开目标检测的面纱。路要一步一步的走，字得一个一个的码。步子不能跨太大，太大容易那个啥，字也不能码太多，太多也不好消化。欢迎关注微信公众号：智能算法目标检测是计算机视觉和数字图像处理的一个热门方向，广泛应用于机器人导航、智能视频监控、工业检测、航空航天等诸多领域，通过计
    
    ](https://blog.51cto.com/u_16213653/11917601)
    
    目标检测前景概率和背景概率的区别 3目标检测的准确率 目标检测 滑动窗口 特征提取 
    
- [
    
    chrome 控制台导入jQuery
    
    要开发一个谷歌插件，我们首先需要了解一下插件的基本架构和工作原理。下面是编写谷歌插件简要的步骤：1.确定插件类型：谷歌提供了多种不同类型的插件，您需要根据插件的具体需求来选择合适的类型。2.编写manifest.json文件：这是每个插件都需要的清单文件，其中定义了插件的名称、描述、版本号等信息。3.编写background.js文件：这个文件包含了插件后台运行的代码，可以用它来注册监听器、处理请
    
    ](https://blog.51cto.com/u_16099265/11918556)
    
    chrome 控制台导入jQuery javascript chrome 开发语言 html 
    

[![](https://s2.51cto.com/oss/202210/28/078be310a471f8818a8dd5da9194a04a.jpg?x-oss-process=image/ignore-error,1)](https://blog.51cto.com/u_13423706)

[我是青石路](https://blog.51cto.com/u_13423706) 

- [181](https://blog.51cto.com/u_13423706/original)
    
    [原创](https://blog.51cto.com/u_13423706/original)
    
- 7.9万
    
    人气
    
- [6](https://blog.51cto.com/u_13423706/followers)
    
    [粉丝](https://blog.51cto.com/u_13423706/followers)
    
- 3
    
    评论
    

- [0](https://blog.51cto.com/u_13423706/translate)
    
    [翻译](https://blog.51cto.com/u_13423706/translate)
    
- [1](https://blog.51cto.com/u_13423706/reproduce)
    
    [转载](https://blog.51cto.com/u_13423706/reproduce)
    
- [0](https://blog.51cto.com/u_13423706/following)
    
    [关注](https://blog.51cto.com/u_13423706/following)
    
- 4
    
    收藏
    

![](https://s2.51cto.com/images/202203/31b28da6607cbc93cd900122f89b19420ac8c5.png?x-oss-process=image/resize,h_96,w_107)

![](https://s2.51cto.com/images/202202/a8ef82362559226c848000f191ce5a08f03207.png?x-oss-process=image/resize,h_96,w_107)

![](https://s2.51cto.com/images/202202/b67592a476d24a33731174926f050c7209f5df.png?x-oss-process=image/resize,h_96,w_107)

![](https://s2.51cto.com/images/202202/03ea5c8040c00c2cbaf04751aae5540380427a.png?x-oss-process=image/resize,h_96,w_107)

![](https://s2.51cto.com/images/202202/e6f51ba16aff6795ae8301531aecaf9001d3b3.png?x-oss-process=image/resize,h_96,w_107)

![](https://s2.51cto.com/images/202202/527253e72c5bb0e15610863965f07e8d6be7aa.png?x-oss-process=image/resize,h_96,w_107)

关注私信

**分类列表**[更多](https://blog.51cto.com/u_13423706/classify)

- [# cassandra5篇](https://blog.51cto.com/u_13423706/category1.html "#  cassandra")
- [# druid1篇](https://blog.51cto.com/u_13423706/category2.html "#  druid")
- [# java web2篇](https://blog.51cto.com/u_13423706/category3.html "#  java web")
- [# java并发编程2篇](https://blog.51cto.com/u_13423706/category4.html "#  java并发编程")
- [# java分布式1篇](https://blog.51cto.com/u_13423706/category5.html "#  java分布式")

**近期文章**

- [1.DAMA认证 首席数据官(CCDO)深圳高研第四期招生](https://blog.51cto.com/hbcx/11926587 "DAMA认证 首席数据官(CCDO)深圳高研第四期招生")
- [2.开源通用验证码识别OCR —— DdddOcr 源码赏析（一）](https://blog.51cto.com/zhaochenwei/11926175 "开源通用验证码识别OCR —— DdddOcr 源码赏析（一）")
- [3.Linux下GO语言连接南大通用GBase 8s数据库](https://blog.51cto.com/u_16565911/11926482 "Linux下GO语言连接南大通用GBase 8s数据库")
- [4.windows环境下使用virtualenv对python进行多版本隔离](https://blog.51cto.com/u_16077267/11925807 "windows环境下使用virtualenv对python进行多版本隔离")
- [5.Python Resize图片](https://blog.51cto.com/catCode2024/11926031 "Python Resize图片")

[![新人福利](https://s2.51cto.com/blog/activity/bride/DetailsBride.gif?x-oss-process=image/ignore-error,1)](https://blog.51cto.com/activity-first-publish#xiang)

**文章目录**

- 开心一刻 
    
- 前情回顾 
    
- kafkawriter 
    
- kafkareader 
    
- 总结 
    

- **目录**
 - **赞**
 - **收藏**
 - **评论**
 - **分享**

[51CTO首页](https://www.51cto.com/) 

[AI.x社区 ![](https://s9.51cto.com/oss/202404/07/2331c9f60a7383b36c1333314be286f249b5b3.png)](https://www.51cto.com/aigc/) 

[博客](https://blog.51cto.com/) 

[学堂 ![](https://s7.51cto.com/oss/202409/05/214d11239058442aee69215591dfb3b38cf091.png)](https://edu.51cto.com/activity/95.html?utm_platform=pc&utm_medium=51cto&utm_source=edu&utm_content=dh) 

[精品班](https://e.51cto.com/?utm_platform=pc&utm_medi-um=51cto&utm_source=zhuzhan&utm_content=sy_topbar) 

[软考社区 ![](https://s4.51cto.com/oss/202409/02/c7122ff03b6a3b0377e0375c0c0385ef190f87.png)](https://rk.51cto.com/?utm_platform=pc&utm_medium=51cto&utm_source=zhuzhan&utm_content=sy_topbar) 

[免费课](https://edu.51cto.com/surl=o0bwJ2) 

[企业培训](https://b.51cto.com/index?utm_source=hometop) 

[鸿蒙开发者社区](https://ost.51cto.com/?utm_source=hometop) 

[WOT技术大会](https://51cto.com/wot/?utm_source=dhl) 

[IT证书 ![](https://s2.51cto.com/oss/202405/15/91545ec31a576825683629ce5f37d4b8a6512c.png)](https://edu.51cto.com/cert/?utm_platform=pc&utm_medium=51cto&utm_source=edu&utm_content=dh) 

[](http://so.51cto.com/?keywords=&sort=time)

公众号矩阵

移动端

[](https://edu.51cto.com/videolist/index.html?utm_platform=pc&utm_medium=51cto&utm_source=zhuzhan&utm_content=dh)[](https://edu.51cto.com/courselist/index-zh5.html?utm_source=hometop)[](https://edu.51cto.com/ranking/index.html?utm_source=hometop)[](https://e.51cto.com/ncamp/list?utm_platform=pc&utm_medium=51cto&utm_source=zhuzhan&utm_content=sy_topbar&rtm_frd=13)[](https://e.51cto.com/rk/?utm_platform=pc&utm_medi-um=51cto&utm_source=zhuzhan&utm_content=sy_topbar&rtm_frd=14)

[](https://e.51cto.com/wejob/list?utm_platform=pc&utm_medi-um=51cto&utm_source=zhuzhan&utm_content=sy_topbar)[](https://e.51cto.com/wejob/list?pid=5&utm_platform=pc&utm_medium=51cto&utm_source=zhuzhan&utm_content=sy_topbar&rtm_frd=41)[](https://e.51cto.com/wejob/list?pid=1&utm_platform=pc&utm_medium=51cto&utm_source=zhuzhan&utm_content=sy_topbar&rtm_frd=42)[](https://rk.51cto.com/?utm_platform=pc&utm_medium=51cto&utm_source=zhuzhan&rtm_frd=07&utm_content=sy_topbar&rtm_frd=43)[](https://e.51cto.com/wejob/list?pid=33&utm_platform=pc&utm_medium=51cto&utm_source=zhuzhan&utm_content=sy_topbar&rtm_frd=44)[](https://t.51cto.com/?utm_platform=pc&utm_medium=51cto&utm_source=zhuzhan&rtm_frd=07&utm_content=sy_topbar&rtm_frd=43)

[](https://b.51cto.com/index?utm_source=hometop)

[](https://ost.51cto.com/postlist)[](https://ost.51cto.com/resource)[](https://ost.51cto.com/answerlist)[](https://ost.51cto.com/study)[](https://ost.51cto.com/column)[](https://ost.51cto.com/activity)

![](https://s5.51cto.com/oss/202302/07/862966771f540df82857144db74b27ee5b4b23.jpeg)

![](https://s4.51cto.com/oss/202302/07/d53d67c771f5cc42bac359bceb138c4cb1713b.jpg)

![](https://s6.51cto.com/oss/202302/07/58786f9973e5e929ef521783e1ee40413b04de.jpeg)

![](https://s3.51cto.com/oss/202302/07/c77c03983d48589b1af789dfc284acb6a7c529.jpeg)

![](https://s4.51cto.com/oss/202302/07/544d71641d983430fc9955636e625e6bb21ff9.jpeg)

![](https://s3.51cto.com/oss/202302/07/f1bd61e720bf669483d941a8486c124f32c451.jpeg)

![](https://s9.51cto.com/oss/202302/07/4719e7b27bae3af5e33552481b6cb913288b01.jpeg)

![](https://s5.51cto.com/oss/202302/07/61a991f484307eed2fe9356cc215c4d8f2dc0f.jpg)

![](https://s5.51cto.com/oss/202408/30/a7a3092691d8f3fdb3322730c0fba80fd82f85.png)

![](https://s8.51cto.com/oss/202302/07/24febb8152cc24e264e642f8cb8bb515efea26.jpeg)

![](https://s9.51cto.com/oss/202302/07/43cca7d0489cc5d1f70060be760bde17d552e2.jpeg)

![](https://s5.51cto.com/oss/202302/07/c4d2220826890472539671d7c428f0c0ee9451.jpg)

![](https://s2.51cto.com/oss/202408/30/b5977c058d1e72d034549101bcef232c9fe32a.png)

[![51CTO博客](https://s2.51cto.com/media/2024/blog/logo.png?x-oss-process=image/ignore-error,1 "51CTO博客")

## 51CTO博客

](https://blog.51cto.com/)

- [首页](https://blog.51cto.com/)
- [关注](https://blog.51cto.com/nav/following)
- [排行榜](https://blog.51cto.com/ranking/hot/aigc)
- [软考题库](https://rk.51cto.com/?utm_platform=pc&utm_medium=51cto&utm_source=blog&utm_content=shouye)![软考题库](https://s2.51cto.com/blog/hot@2x.png?x-oss-process=image/ignore-error,1)
- [![新人福利](https://s2.51cto.com/images/100/blog/activity/first2.gif?x-oss-process=image/ignore-error,1)](https://blog.51cto.com/activity-first-publish#shouye)

- 
- 写文章
- [创作中心](https://blog.51cto.com/creative-center/index)[](https://blog.51cto.com/creative-center/task)
- [登录注册](https://home.51cto.com/index?from_service=blog&scene=login1&reback=https://blog.51cto.com/u_13423706/11918689)

[![51CTO博客](https://s2.51cto.com/images/100/blog/logo4.png?x-oss-process=image/ignore-error,1 "51CTO博客")](https://blog.51cto.com/)

Copyright © 2005-2024 [51CTO.COM](https://www.51cto.com/) 版权所有 京ICP证060544号

关于我们

|   |   |   |   |
|---|---|---|---|
|[官方博客](https://blog.51cto.com/51ctoblog)|[全部文章](https://blog.51cto.com/nav)|[热门标签](https://blog.51cto.com/topic/all)|[班级博客](https://blog.51cto.com/class-blog/index)|
|[了解我们](https://www.51cto.com/about/aboutus.html)|[网站地图](https://www.51cto.com/about/map.html)|[意见反馈](https://blog.51cto.com/feedback?utm_medium=aboutus2)|

友情链接

|   |   |
|---|---|
|[鸿蒙开发者社区](https://ost.51cto.com/?utm_source=blogsitemap)|[51CTO学堂](https://edu.51cto.com/)|
|[51CTO](https://www.51cto.com/)|[软考资讯](https://edu.51cto.com/rk/)|