---
layout: post
title:  Kestrel and Storm
---

本文为Storm官方文档[KestrelAndStorm](http://storm.incubator.apache.org/documentation/Kestrel-and-Storm.html)的读书笔记

本页面解释如何使用Storm消费Kestrel集群的数据。

## 背景

本教程来自[storm-kestrel](https://github.com/nathanmarz/storm-kestrel) 和storm-starter。

## Kestrel Server and Queue

单个kestrel server有一个队列集合。Kestrel 队列是一个非常简单的运行在JVM上的消息队列，它使用memcache协议和客户端通信。如果想了解更详细，可以看[storm-kestrel](https://github.com/nathanmarz/storm-kestrel)上的[KestrelThriftClient](https://github.com/nathanmarz/storm-kestrel/blob/master/src/jvm/backtype/storm/spout/KestrelThriftClient.java)的实现。

每个队列严格按照FIFO顺序。为了保证性能，消息缓存在内存中。不过只有128M在内存中。停止服务时，队列的状态存储在日志文件中。

详细的，可以看[这里](https://github.com/nathanmarz/kestrel/blob/master/docs/guide.md)。

Kestrel ： fast，small，durable，reliable

举例说，Twitter使用Kestrel做消息架构的主心骨，[这里](http://bhavin.directi.com/notes-on-kestrel-the-open-source-twitter-queue/)有描述。

## 向Kestrel中添加消息。

首先，我们需要有一个程序向Kestrel队列发送消息。下面的方法使用了storm-kestrel中的KestrelClient。

    private static void queueSentenceItems(KestrelClient kestrelClient, String queueName) throws ParseError, IOException {

        String[] sentences = new String[] {
                "the cow jumped over the moon",
                "an apple a day keeps the doctor away",
                "four score and seven years ago",
                "snow white and the seven dwarfs",
                "i am at two with nature"}; 

        Random _rand = new Random();    

        for(int i=1; i<=10; i++){   

            String sentence = sentences[_rand.nextInt(sentences.length)];   

            String val = "ID " + i + " " + sentence;    

            boolean queueSucess = kestrelClient.queue(queueName, val);  

            System.out.println("queueSucess=" +queueSucess+ " [" + val +"]");
        }
    }

## 从Kestrel从移除

这个方法从队列中取出消息但不删除。

    private static void dequeueItems(KestrelClient kestrelClient, String queueName) throws IOException, ParseError { 
        for(int i=1; i<=12; i++){

            Item item = kestrelClient.dequeue(queueName);

            if(item==null){
                System.out.println("The queue (" + queueName + ") contains no items.");
            }
            else
            {
                byte[] data = item._data;
                String receivedVal = new String(data);
                System.out.println("receivedItem=" + receivedVal);
            }
        }
    }

这个方法从队列中取消息并删除它们：

     private static void dequeueAndRemoveItems(KestrelClient kestrelClient, String queueName) throws IOException, ParseError { 
        for(int i=1; i<=12; i++){

            Item item = kestrelClient.dequeue(queueName);


            if(item==null){
                System.out.println("The queue (" + queueName + ") contains no items.");
            }
            else
            {
                int itemID = item._id;


                byte[] data = item._data;

                String receivedVal = new String(data);

                kestrelClient.ack(queueName, itemID);

                System.out.println("receivedItem=" + receivedVal);
            }
        }
    }


## 持续的向Kestrel中添加消息

这是我们的最终程序，用以向本地Kestrel server的'sentence_queue'中持续地发送消息。

要停止程序，在console中敲‘]’和回车。

    import java.io.IOException; 
    import java.io.InputStream; 
    import java.util.Random;
    import backtype.storm.spout.KestrelClient;
    import backtype.storm.spout.KestrelClient.Item;
    import backtype.storm.spout.KestrelClient.ParseError;   

    public class AddSentenceItemsToKestrel {    

        /**
         * @param args
         */
        public static void main(String[] args) {    

            InputStream is = System.in; 

            char closing_bracket = ']'; 

            int val = closing_bracket;  

            boolean aux = true; 

            try {   

                KestrelClient kestrelClient = null;
                String queueName = "sentence_queue";    

                while(aux){ 

                    kestrelClient = new KestrelClient("localhost",22133);   

                    queueSentenceItems(kestrelClient, queueName);   

                    kestrelClient.close();  

                    Thread.sleep(1000); 

                    if(is.available()>0){
                     if(val==is.read())
                         aux=false;
                    }
                }
            } catch (IOException e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
            }
            catch (ParseError e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
            } catch (InterruptedException e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
            }   

            System.out.println("end");  

        }
    } 

拓扑使用KestrelSpout从Kestrel 队列中读取sentence，分割之（Bolt:SplitSentence）,提交每个word出现次数(Bolt:WordCount)。数据处理在[Guaranteeing message processing](http://storm.incubator.apache.org/documentation/Guaranteeing-message-processing.html)中描述。

    TopologyBuilder builder = new TopologyBuilder(); 
    builder.setSpout("sentences", new KestrelSpout("localhost",22133,"sentence_queue",new StringScheme())); 
    builder.setBolt("split", new SplitSentence(), 10) 
        .shuffleGrouping("sentences"); 
    builder.setBolt("count", new WordCount(), 20) 
        .fieldsGrouping("split", new Fields("word"));

## 执行

首先，启动本地kestrel server。

然后，等待5秒以避免ConnectException

现在执行程序向Kestrel 队列中添加数据。启动Storm topology。这两个顺序没关系。

如果以TOPOLOGY_DEBUG形式运行，你可以看到元组在拓扑中被提交。

## 思考

可以一个Spout对应KestrelSpout中的多个队列么？

换句话说，Spout 有m个task，KestrelSpout中有n个队列。？

我看了下[KestrelThriftSpout](https://github.com/nathanmarz/storm-kestrel/blob/master/src/jvm/backtype/storm/spout/KestrelThriftSpout.java)的代码，其支持的模型就是 hosts+queue_name。估计Kestrel被使员工的时候就是一个队列对外开放。