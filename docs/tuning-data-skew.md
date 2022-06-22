# spark 数据倾斜
- 提高shuffle并行度：增大shuffle read task参数值，让每个task处理比原来更少的数据；适应所有场景，简单有效可作为首选方案，缺点是缓解的效果有限；对于group,join shuffle类语句，可以通过设置spark.sql.shuffle.partitions来调整并发；
- 两阶段聚合：适用于groupbykey分组聚合，reducebykey等shuffle算子，思路：首先通过map给每个key打上n以内的随机数前缀进行局部聚合，并进行reducebykey的局部聚合，然后再次map将key的前缀随机数去掉再次进行全局聚合；可以让整个过程中原本一个task处理的数据分摊到多个task做局部聚合，规避单task数据过量，缺点是仅适用于聚合类的Shuffle操作，无法适用于join类的shuffle操作
- 广播broadcast: 对RDD或Spark SQL使用join类操作或语句，且join操作的RDD比较小（百兆或1,2G）；使用broadcast和map类算子实现join的功能替代原本的join，彻底规避shuffle。对较小RDD直接collect到内存，并创建broadcast变量；并对另外一个RDD执行map类算子，在该算子的函数中，从broadcast变量（collect出的较小RDD）与当前RDD中的每条数据依次比对key，相同的key执行你需要方式的join；
- 采样倾斜key拆分join：适用于两个表都很大，但大key占比很小；对join导致的倾斜是因为某几个key，可将原本RDD中的倾斜key从原RDD拆分出来得到新RDD，并以加随机前缀的方式打散n份做join，将倾斜key对应的大量数据分摊到更多task上来规避倾斜；不适用于大量倾斜key;
- 随机前缀加扩容RDD进行join：适用场景：RDD中有大量key导致倾斜
    - 首先查看RDD/Hive表中数据分布并找到造成倾斜的RDD/表；
    - 对倾斜RDD中的每条数据打上n以内的随机数前缀；
    - 对另外一个正常RDD的每条数据扩容n倍，扩容出的每条数据依次打上0到n的前缀；
    - 对处理后的两个RDD进行join。与采样倾斜key方案不同在于这里对不倾斜RDD中所有数据进行扩大n倍，而不是找出倾斜key进行扩容, 效果非常显著；缺点是扩容需要大内存
实际中需要结合业务全盘考虑，可先提升Shuffle的并行度，最后针对数据分布选择后面方案中的一种或多种灵活应用。
