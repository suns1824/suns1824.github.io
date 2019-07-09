## producer开发

#### 构造Producer对象
```text
public Producer(String topic, Boolean isAsync) {
        Properties props = new Properties();
        //必须指定，该参数指定了一组host:port对，创建向kafka服务器的连接，比如k1:9092,k2:9092,k3:9092
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, KafkaProperties.KAFKA_SERVER_URL + ":" + KafkaProperties.KAFKA_SERVER_PORT);
        props.put(ProducerConfig.CLIENT_ID_CONFIG, "DemoProducer");
        //下面两个参数都必须指定，这两个参数指定了实现了org.apache.kafka.common.serialization.server接口的类的全限定名，Kafka为大多数的初始类型默认提供了现成的序列化器。
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, IntegerSerializer.class.getName());
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        producer = new KafkaProducer<>(props);
        this.topic = topic;
        this.isAsync = isAsync;
}
//调用，两种方式，同步和异步，Kafka底层完全实现了异步化发送，当时通过Java的Future接口实现了API层级的同步和异步发送方式
if (isAsync) { // Send asynchronously，DemoCallBack实现了CallBack接口
                producer.send(new ProducerRecord<>(topic,
                    messageNo,
                    messageStr), new DemoCallBack(startTime, messageNo, messageStr));
            } else { // Send synchronously，通过Future.get无限等待结果返回，实现同步发送的效果
                try {
                    producer.send(new ProducerRecord<>(topic,
                        messageNo,
                        messageStr)).get();
                    System.out.println("Sent message: (" + messageNo + ", " + messageStr + ")");
                } catch (InterruptedException | ExecutionException e) {
                    e.printStackTrace();
                }
            }
            
//producer一定要关闭
producer.close();
```
producer主要参数(除了上文提及)：
>* **acks**: 用于控制producer生产消息的持久性。当一条消息被发送到kafka集群时，这条消息会被发送到指定topic分区leader所在的broker上，producer等待从该leader broker返回消息的写入结果确保消息被成功提交。这之后producer可以
继续发送下一条。leader broker何时发送写入结果返还给producer直接影响消息的持久性甚至是producer端的吞吐量：producer端越快接收到leader broker的相应就能越快发送下一条消息，吞吐量就越大。  
acks指定了在给producer发送响应前，leader broker必须要确保已成功写入该消息的副本数。acks有三个取值：0，1，all：0表示producer无需理睬leader broker的处理结果；all/-1表示leader broker只有在将消息写入本地日志还会等待ISR中
其他所有副本都写入各自的本地日志后，才会发送响应结果给producer；1表示XXX。
>* **buffer.memory**: 该参数指定了producer端用于缓存消息的缓冲区大小，单位是字节(默认为32MB)。producer启动时会创建一个内存缓冲区用于保存待发送的消息，由另一个专属线程负责从缓冲区读消息执行真正的发送（如果写入缓冲区的速度超过
了读取消息并发送的速度，就会XXX）。该参数的大小可以近乎认定为producer程序使用的内存大小，过小的内存缓冲区会降低producer程序的吞吐量。
>* **compression.type**： 指定producer端消息的压缩类型，可以降低网络I/O传输开销，但也会增加prodducer端的CPU开销。如果broker端的压缩参数设置得和producer不同，broker端在写入消息时也会额外使用CPU资源对消息进行解压缩-重压缩操作。
目前Kafka支持3种压缩算法：GZIP、Snappy、LZ4(实际使用经验来看LZ4性能最好)。
>* **retries**: 该参数指定了producer端发送消息的重试次数，超过这个次数后，broker端才会将这些错误封装进回调函数的异常中。在考虑retries的参数设置时，需要考虑两点：     
    1) 如果瞬时的网络抖动使得broker已成功写入消息但没有成功发送消息给producer，因此producer会认为消息发送失败，从而重试。为此，consumer端需要执行去重处理，不过kafka从0.11.0.0已经支持“精确一次”处理语义，从设计上避免了该类问题。    
    2) 重试可能造成乱序，Kafka默认将5个消息发送请求缓存在内存中，消息重试可能会导致消息流乱序，可以将**max.in.flight.requests.per.connection**参数设置为1，确保同一时刻只能发送一个请求。    
    另外，producer两次重试之间会停顿一段时间，以防止频繁重试对系统带来冲击，该时间是可配置的，由参数**retry.backoff.ms**指定，默认为100ms，推荐通过测试平均的leader选举时间(最常见的错误是leader选举导致)并根据该时间来设置该参数。   
>* **batch.size**: 该参数对于调优producer吞吐量和延时性能指标都有着重要的作用，producer会将发往同一分区的多条消息封装进一个batch中。当batch满的时候，producer返回发送batch中所有的消息。当然很可能batch还有很多空间时producer就发送该batch。
batch过小，一次发送请求能够写入的消息很少，producer吞吐量就会很低；如果很大，那么内存压力就大，延迟也会高。
>* **linger.ms**: 该参数控制消息发送延时行为，默认为0，表示消息立即被发送，无需关心batch是否填满。但是这样会拉低producer的吞吐量，因为producer发送的每次请求中包含的消息越多，producer就越能将发送请求的开销分摊到更多的消息上，
从而提高吞吐量。   
>* **max.request.size**:该参数用于控制producer端发送请求的大小，即能够发送的最大消息大小。
>* **request.timeout.ms**: producer发送请求给broker后，broker需要在规定的时间范围内将处理结果返还给producer，该参数控制了该时间，默认为30s。

#### 消息分区机制
Kafka Producer发送消息时需要确认将消息发送到topic的那个partition中去，producer提供了分区策略和对应的分区器(partitioner)提供给用户。默认partitioner会尽可能地保证相同key的消息会被发送到相同的分区中。如果没有为消息指定key，那么partitioner
会采用轮询的方式来确保消息在topic的所有分区上均匀分配。   
ps: 对于有key的消息，Java版本的producer采用了murmur2算法计算key的hash值，然后对总分区取模得到消息将要被发送的分区号。用户也可以自己实现分区策略：
```text
@Override
    public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
        ......
        return 0;
    }

    @Override
    public void close() {
    //资源关闭
    }

    @Override
    public void configure(Map<String, ?> configs) {
       //configure方法实现了Configurable接口，这里进行资源的初始化工作
    }

然后在KafkaProducer的Properties对象中指定partitioner.class参数：
properties.put("partitioner.class", "com.raysurf.xxx.RaysurfPartitioner");
```
#### 消息序列化
Kafka为基本的数据类型提供了序列化实现，复杂类型一般需要自定义序列化：
```text
@Data
public class User {
  private String name;
  private int age;
  private int sex;
}
public class UserSerializer implements Serializer {
    private ObjectMapper mapper;
    @Override
    public void configure(Map configs, boolean isKey) {
        mapper = new ObjectMapper();
    }

    @Override
    public byte[] serialize(String topic, Object data) {
        byte[] ret = null;
        try {
            ret = mapper.writeValueAsString(data).getBytes("utf-8");
        } catch (UnsupportedEncodingException e) {
            e.printStackTrace();
        } catch (JsonProcessingException e) {
            e.printStackTrace();
        }
        return ret;
    }

    @Override
    public void close() {

    }
}
```

#### producer拦截器
对于producer而言，interceptor使得用户在消息发送之前以及producer回调逻辑前有机会对消息做一些定制化的需求，比如修改消息等。同时producer允许用户指定多个interceptor按序作用于同一条消息从而形成一个拦截链。
举例说明：  
实现一个双interceptor组成的拦截链。第一个interceptor在消息发送前将时间戳信息加到消息的value的最前部；第二个interceptor会在消息发送后更新成功消息数或失败发送消息数。   
interceptor1:   
```text
public class TimeStampPrependerInterceptor implements ProducerInterceptor<String, String> {
    /*
    这个方法在KafkaProducer.send中被使用到，表明其运行在主线程中。producer确保在消息被序列化之前调用该方法，用户可以在这里对消息做任何操作，但最好不要
    修改消息所属的topic和分区有关的信息，否则会影响目标分区的计算。
     */
    @Override
    public ProducerRecord<String, String> onSend(ProducerRecord<String, String> record) {
        //将时间戳信息加到消息value的最前部
        return new ProducerRecord(record.topic(), record.partition(), record.timestamp(), record.key(), System.currentTimeMillis() + "," + record.value().toString());
    }
    /*
    该方法运行在producer的I/O线程，在消息被应答之前或者消息发送失败时调用。
     */
    @Override
    public void onAcknowledgement(RecordMetadata metadata, Exception exception) {...}
    //关闭interceptor,做资源清理相关工作
    @Override
    public void close() {...}
    //实现Configurable接口
    @Override
    public void configure(Map<String, ?> configs) {...}
}
```
interceptor2:  
```text
public class CounterInterceptor implements ProducerInterceptor<String, String> {
    private int successCount = 0;
    private int failCount = 0;
    @Override
    public ProducerRecord<String, String> onSend(ProducerRecord<String, String> record) {
        return record;
    }
    @Override
    public void onAcknowledgement(RecordMetadata metadata, Exception exception) {
        if (exception == null) {
            successCount ++;
        } else {
            failCount ++;
        }
    }
    @Override
    public void close() {
        System.out.println("success count: " + successCount);
        System.out.println("fail count: " + failCount);
    }
    @Override
    public void configure(Map<String, ?> configs) {...}
}
```

在构造KafkaProducer的代码中添加如下：
```text
        List<String> interceptors = new ArrayList<>();
        interceptors.add("com.raysurf.test.kafka.example.interceptor.TimeStampPrependInterceptro");
        interceptors.add("com.raysurf.test.kafka.example.interceptor.CounterInterceptor");
        props.put(ProducerConfig.INTERCEPTOR_CLASSES_CONFIG, interceptors);
```

#### 无消息丢失配置
**Kafka的producer采用异步发送机制，KafkaProducer.send方法仅仅将消息放入缓冲区内，然后有一个I/O线程从缓冲区提取消息并封装进消息batch，然后
发送出去**。如果I/O线程发送之前producer崩溃，那么存储缓冲区中的消息就会全部丢失。另外，如果由于瞬时的网络抖动，Kafka进行消息发送的重试，则可能会出现乱序。   
对于这个问题，最粗暴的方法是采用同步的方法：producer.send(record).get()，但性能可能会受到较大影响。可以通过如下配置，实现producer端的无消息丢失：
```text
//producer端配置
block.on.buffer.full=true  //内存缓冲区满时producer阻塞并停止接收消息而不是抛出异常，0.10.0.0后不用理会该参数，设置max.block.ms(超出这个时间再报异常)即可
acks=all or -1
retries=Integer.MAX_VALUE
max.in.flight.requests.per.connection=1

使用KafkaProducer.send(record, callback)
callback中显示地立即关闭producer，使用close(0)

//broker端配置
unclean.leader.election.enable=false  //关闭unclean leader选举，不允许非ISR中的副本被选举为leader，从而避免broker端因日志水位截断而造成的消息丢失
replication.factor=3   //>=3个备份
min.insync.replicas=2  //至少被写入到ISR中的多少个副本才算成功  
确保replication.factor>min.insync.replicas
enable.auto.commit=false
```
#### 多线程处理
>* 多线程单KafkaProducer实例（因为KafkaProducer是线程安全的）
>* 多线程多KafkaProducer实例     
对比：  

|         | 优势   |  劣势  |
| --------   | -----:  | :----:  |
| 多线程单KafkaProducer实例     | 实现简单，性能好 |   所有线程共享一个内存缓冲区，可能需要较多内存；一个线程崩溃，整个实例被破坏     |
| 多线程多KafkaProducer实例        |   可以细粒度调优(每个线程拥有KafkaProducer实例)，单个KafkaProducer崩溃不会影响其他   |   需要较大的内存开销   |
