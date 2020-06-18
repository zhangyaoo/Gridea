---
title: 'O(1)时间复杂度下，双向链表实现LRU'
date: 2020-06-15 19:17:45
tags: [Java基础]
published: true
hideInList: false
feature: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1592231255802&di=4e1d20b36c0f79e7837c4a3128fb75bb&imgtype=0&src=http%3A%2F%2Fk.zol-img.com.cn%2Fsoftbbs%2F6737%2Fa6736285_s.jpg
isTop: false
---

链表实现思路：
 * 1、数据是直接利用 HashMap 来存放的。
 * 2、内部使用了一个双向链表来存放数据，所以有一个头结点 header，以及尾结点 tailer。
 * 3、每次写入头结点，删除尾结点时都是依赖于 header tailer
注：可以参考linkedHashMap底层结构

![](https://1124826889.github.io/zhangyao.github.io//post-images/1592220507945.png)
```
import com.google.common.collect.Maps;
import java.util.Map;

/**
 * 线程不安全，同步机制自行控制。
 */
public class LRUCacheV2 {
    /**
     * 缓存map
     */
    private final Map<String, Node> cacheMap;

    /**
     * 头指针
     */
    private Node head;

    /**
     * 尾指针
     */
    private Node tail;

    /**
     * 容量
     */
    private final int cacheSize;

    /**
     * 当前容量
     */
    private int currentCacheSize;

    LRUCacheV2(int capacity){
        cacheMap = Maps.newHashMapWithExpectedSize(capacity);
        cacheSize = capacity;
        currentCacheSize = 0;
    }

    public Object get(String key){
        Node node = cacheMap.get(key);
        if(node != null){
            // 移动到头指针
            move2head(node);
            return node.getData();
        }
        return null;
    }

    public void remove(String key){
        Node node = cacheMap.get(key);
        if(node != null){
            Node pre = node.getPre();
            Node next = node.getNext();
            if(pre != null){
                pre.setNext(next);
            }
            if(next != null){
                next.setPre(pre);
            }

            // 如果删除刚好是头节点或者尾节点，也要移动指针
            if(node.getKey().equals(head.getKey())){
                head = pre;
            }
            if(node.getKey().equals(tail.getKey())){
                tail = next;
            }

            cacheMap.remove(key);
        }
    }

    public void put(String key, Object value){
        Node node = cacheMap.get(key);
        if(node != null){
            // 存在节点的话，就覆盖，并且放到头
            node.setData(value);
            move2head(node);
            cacheMap.put(key, node);
        }else {
            // 不存在节点，构造并且放到头
            if(currentCacheSize == cacheSize){
                // 删除尾node
                String delKey = tail.getKey();
                cacheMap.remove(delKey);

                // 尾指针移动
                Node next = tail.getNext();
                if(next != null){
                    next.setPre(null);
                }
                tail.setNext(null);
                tail = next;

            }else{
                currentCacheSize++;
            }
            node = new Node();
            node.setData(value);
            node.setKey(key);
            // 头指针移动
            move2head(node);
        }
        cacheMap.put(key, node);
    }

    /**
     * 节点移到头
     */
    private void move2head(Node node){
        if(head == null){
            // 初始化head 和 tail
            head = node;
            head.setNext(null);
            head.setPre(null);
            tail = node;
        }else {
            // 如果是相同的Key，啥都不用动，node就是最新的头
            if(node.getKey().equals(head.getKey())){
                return;
            }

            // 截取node
            Node pre = node.getPre();
            Node next = node.getNext();
            if(pre != null){
                pre.setNext(next);
            }
            if(next != null){
                next.setPre(pre);
            }

            // 如果要截取的节点是尾节点，那么尾节点指针也要向前移动
            if(node.getKey().equals(tail.getKey())){
                tail = next;
            }

            // 放在头前面
            head.setNext(node);
            node.setPre(head);
            // node下个指针指向null
            node.setNext(null);
            head = node;
        }
    }

    @Override
    public String toString() {
        StringBuilder sb = new StringBuilder() ;
        Node node = head;
        while (node != null){
            sb.append(node.getKey()).append(":")
                    .append(node.getData())
                    .append("-->") ;
            node = node.getPre();
        }
        return sb.toString();
    }

    public static void main(String[] args) {
        LRUCacheV2 lruCacheV2 = new LRUCacheV2(4);
        lruCacheV2.put("1","1");
        lruCacheV2.put("2","2");
        lruCacheV2.put("3","3");
        lruCacheV2.put("4","4");
        lruCacheV2.put("5","5");
        //lruCacheV2.get("2");
        //lruCacheV2.put("2","22");
        lruCacheV2.remove("5");
        System.out.println(lruCacheV2.toString());
    }
}
```
参考：
1、https://www.iteye.com/blog/gogole-692103
2、https://crossoverjie.top/2018/04/07/algorithm/LRU-cache/
