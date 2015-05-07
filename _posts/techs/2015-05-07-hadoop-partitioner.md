---
layout: post
title:  Hadoop自定义Partitioner
---

##背景

在当前的项目中，需要使用Hive表格的Bucket功能，以及OrcFile的Index功能。一般情况下，我们会等数据产出后，再做一轮Bucket&Sort任务，将bucketKey的Hash值相等的记录写进同一个文件，并且在文件内部排序，来实现。

分两阶段来跑，是比较浪费IO的，加上我的第一轮是一个Join任务，Join的key刚好是`id\0id_type`,分桶的key是id，所以只要定制一个Partitioner，在Partitioner的时候基于id来做Hash，在Reducer端，刚好利用了key有序这个特性，保证id是有序的。

所以问题就变成了写一个基于key中的id的Partitioner。

##实现

直接贴代码

	public class ReduceNumBucketPartitioner<K2, V2> implements Partitioner<K2, V2> {
	    private JobConf jobConf;
	    private ArrayList<OutputJobInfo> jobInfoList;
	    private Map<String, Integer> tableNameList;
	    private int tableNum;
	    private List<List<String>> bucketColsList;
	    private List<List<Integer>> bucketIdxsList;
	    private List<Integer> bucketNumList;
	    private List<Integer> reducerNumIndexList;
	    private byte separator = 0;
	    private String tablePrefix;
	    private int reduceBucketNum = -1;
	    private List<String> bucketCols;
	    private static final Logger LOGGER = UdwLogger.getLogger();
	
	    /* (non-Javadoc)
	     * @see org.apache.hadoop.mapred.JobConfigurable#configure(org.apache.hadoop.mapred.JobConf)
	     */
	    @Override
	    public void configure(JobConf job) throws RuntimeException {
	        try {
	            this.jobInfoList = JobInfoUtil.getOutputJobInfos(job);
	        } catch (IOException e) {
	            e.printStackTrace();
	            throw new IllegalArgumentException("Build OutputJobInfo failed!");
	        }
	        this.jobConf = job;
	        this.tableNum = jobInfoList.size();
	        this.tableNameList = new HashMap<String, Integer>(tableNum);
	        this.bucketColsList = new ArrayList<List<String>>(tableNum);
	        this.bucketIdxsList = new ArrayList<List<Integer>>(tableNum);
	        this.bucketNumList = new ArrayList<Integer>(tableNum);
	        this.reducerNumIndexList = new ArrayList<Integer>(tableNum);
	        String separatorStr = jobConf.get(UdwConstants.KEY_STREAMING_SEPARATOR,
	                UdwConstants.DEFAULT_STREAMING_SEPARATOR);
	        this.separator = CommonUtil.StringToByte(separatorStr);
	        this.tablePrefix = jobConf.get(UdwConstants.KEY_OUTPUT_TABLE_PREFIX, "");
	
	        // get reduce num
	        this.reduceBucketNum = jobConf.getInt("mapred.reduce.tasks", -1);
	        LOGGER.info("reduceBucketNum: " + reduceBucketNum);
	        this.bucketCols = Arrays.asList(jobConf.get("udw.mapred.bucket.cols").split(","));
	        
	        int tableIdx = 0;
	        int reducerNumIndex = 0;
	
	        for (OutputJobInfo jobInfo : jobInfoList) {
	            // 1,store bucket info
	            bucketNumList.add(reduceBucketNum);
	            reducerNumIndexList.add(reducerNumIndex);
	            //bucketColsList.add(jobInfo.getBucketColumnList());
	            bucketColsList.add(this.bucketCols);
	            List<Integer> bucketIdx = new ArrayList<Integer>();
	            for (String col : this.bucketCols) {
	                bucketIdx.add(jobInfo.getSchema().getPosition(col));
	                
	            }
	            bucketIdxsList.add(bucketIdx);
	
	            // 2,store tableName info
	            tableNameList.put(jobInfo.getName(), tableIdx);
	
	            // 3,calculate the next projects' reducer number Index
	            /*
	            if (jobInfo.isBucketed()) {
	                reducerNumIndex += jobInfo.getBucketNum();
	            }*/
	            reducerNumIndex += this.reduceBucketNum;
	            tableIdx++;
	        }
	    }
	
	    /* (non-Javadoc)
	     * @see org.apache.hadoop.mapred.Partitioner#getPartition(java.lang.Object, java.lang.Object, int)
	     */
	    @Override
	    public int getPartition(K2 key, V2 value, int numReduceTasks) {
	        String realKey = new String();
	        try {
	            
	            int valueSize = 0;
	            byte[] valueBytes;
	            if (value instanceof Text) {
	                valueSize = ((Text) value).getLength();
	                valueBytes = ((Text) value).getBytes();
	            } else {
	                valueSize = ((BytesWritable) value).getLength();
	                valueBytes = ((BytesWritable) value).getBytes();
	            }
	            
	            int keySize = 0;
	            byte[] keyBytes;
	            if (key instanceof Text) {
	                keySize = ((Text) key).getLength();
	                keyBytes = ((Text) key).getBytes();
	            } else {
	                keySize = ((BytesWritable) key).getLength();
	                keyBytes = ((BytesWritable) key).getBytes();
	            }
	
	            // Key: id \0 id_type
	            // 12345\0abc
	            // beginPos:0
	            // endPos:5
	            int beginPos = 0;
	            
	            int endPos = get_first_end_pos(keyBytes, keySize, separator);
	
	            Object fieldValue = UdwRecordUtil.decode(keyBytes, beginPos, endPos, DataType.STRING);
	            realKey = new String(keyBytes, beginPos, endPos);
	
	            // 3,compute the hashFunc(bucketColumns)
	            
	            //curTblIndex : 0 因为拿不到这个TableIndex，所以不要切value
	            int bucketNum = bucketNumList.get(0);
	            /*if (bucketNum <= 0) {
	                LOGGER.error("Bucket Number of [" + tableName + "] should > 0");
	                return -1;
	            }*/
	            int reduceIndex = 0;
	            return ReduceNumBucketPartitioner.getBucketResult(bucketNum, fieldValue, reduceIndex);
	            
	        } catch (IOException e) {
	            /*
	             * MultiTableRecordWriter throw IOException when error records> 100
	             * The error record will not write to file
	             */
	            LOGGER.warn(e.toString());
	            
	            return (realKey.hashCode() & Integer.MAX_VALUE) % numReduceTasks;
	        }
	
	    }
	    
	    public static int get_first_end_pos(byte[] keyBytes, int keySize, byte separator) {
	        // TODO Auto-generated method stub
	        int endPos = 0;
	        for (int i = 0; i != keySize; i++) {
	            if (keyBytes[i] == separator) {
	                endPos = i;
	                break;
	            }
	        }
	        return endPos;
	    }
	
	    /**
	     * Get the number describe use which reducer  
	     * 
	     * @param bucketNum 
	     * @param recordValue   this is reducer key,need split to get real value
	     * @param bucketColName the name of bucket-key in schema
	     * @param curJobInfo
	     * @param reduceIndex
	     * @return
	     * @throws IOException
	     */
	    public static int getBucketResult(int bucketNum, Object fieldValue, int reduceIndex)
	            throws IOException {
	        int bucketResult = -1;
	
	        TypeInfo type = TypeInfoUtils.getTypeInfoFromTypeString("string");
	        ObjectInspector bucketObjInspector = UdwRecordUtil
	                .getObjectInspector(type);
	        Integer hashCode = -1;
	        try {
	            hashCode = ObjectInspectorUtils.hashCode(fieldValue,
	                    bucketObjInspector);
	        } catch (RuntimeException e) {
	            e.printStackTrace();
	            throw new IOException(
	                    "The BucketColum not support STRUCT and UNION type");
	        }
	        bucketResult = ((hashCode & Integer.MAX_VALUE) % bucketNum) + reduceIndex;
	        return bucketResult;
	    }

	}

##遇到的问题

遇到两个问题：

1. 第一版是基于Value来寻找bucket_key来做签名的，但是Mapper输出的value，和输出表的Schema不一致导致会抛异常，异常状态下的hash值是基于key来计算的，没有发现错误导致join不上。
2. 第二版是基于Value中的bucket_key的Schema来做签名的，同样的，Mapper输出的value和输出表的Schema不一致，异常join不上。


##写Mapreduce一定要做的事情

+ 关键信息的Counter。假设在做一个Join，那么每种情况每边输出多少，输出多少。
+ 线下验证，streaming任务的UT要做足，cat | mapper | sort | reducer，输出的expect
+ UT很关键，这次Partitioner采坑，就是因为我没有做足UT，自己找原因，找自己的原因。