# TreeMap

TreeMap基于红黑树实现。



## 源码



### 查找

`TreeMap`基于红黑树实现，而红黑树是一种自平衡二叉查找树，所以 TreeMap 的查找操作流程和二叉查找树一致。二叉查找树的查找流程是这样的：

先将目标值和根节点的值进行比较，如果目标值小于根节点的值，则再和根节点的左孩子进行比较。如果目标值大于根节点的值，则继续和根节点的右孩子比较。在查找过程中，如果目标值和二叉树中的某个节点值相等，则返回 true，否则返回 false。TreeMap 查找和此类似，只不过在 TreeMap 中，节点（Entry）存储的是键值对。在查找过程中，比较的是键的大小，返回的是值，如果没找到，则返回`null`。



### 实现Compare接口

其key对象为什么必须要实现Compare接口?  因为红黑树的插入、删除的过程，必须要找到key的位置，因此需要进行比较key的大小，因此需要实现Compare。





## 实现一致性Hash算法

一致性Hash环要求能**快速通过key查找到桶的位置**，并且**桶在环上有序**，所以使用TreeMap来实现环。

```java
/**
 * 带虚拟节点的一致性Hash算法
 */
public class ConsistentHashingWithVirtualNode {

    /**
     * 待添加入Hash环的服务器列表
     */
    private static String[] servers =
            { "192.168.0.0:111", "192.168.0.1:111", "192.168.0.2:111", "192.168.0.3:111",
            "192.168.0.4:111" };

    /**
     * 真实结点列表,考虑到服务器上线、下线的场景，即添加、删除的场景会比较频繁，这里使用LinkedList会更好
     */
    private static List<String> realNodes = new LinkedList<String>();

    /**
     * 虚拟节点，key表示虚拟节点的hash值，value表示虚拟节点的名称
     */
    private static SortedMap<Integer, String> virtualNodes = new TreeMap<Integer, String>();

    /**
     * 虚拟节点的数目，这里写死，为了演示需要，一个真实结点对应5个虚拟节点
     */
    private static final int VIRTUAL_NODES = 5;

    static {
        // 先把原始的服务器添加到真实结点列表中
        for (int i = 0; i < servers.length; i++) {
            realNodes.add(servers[i]);
        }


        // 再添加虚拟节点，遍历LinkedList使用foreach循环效率会比较高
        for (String str : realNodes) {
            for (int i = 0; i < VIRTUAL_NODES; i++) {
                String virtualNodeName = str + "&&VN" + String.valueOf(i);
                int hash = getHash(virtualNodeName);
                System.out.println("虚拟节点[" + virtualNodeName + "]被添加, hash值为" + hash);
                virtualNodes.put(hash, virtualNodeName);
            }
        }
        System.out.println();
    }

    /**
     * 使用FNV1_32_HASH算法计算服务器的Hash值,这里不使用重写hashCode的方法，最终效果没区别
     */
    private static int getHash(String str) {
        final int p = 16777619;
        int hash = (int) 2166136261L;
        for (int i = 0; i < str.length(); i++)
            hash = (hash ^ str.charAt(i)) * p;
        hash += hash << 13;
        hash ^= hash >> 7;
        hash += hash << 3;
        hash ^= hash >> 17;
        hash += hash << 5;

        // 如果算出来的值为负数则取其绝对值
        if (hash < 0)
            hash = Math.abs(hash);
        return hash;
    }

    /**
     * 得到应当路由到的结点
     */
    private static String getServer(String node) {
        
        // 得到带路由的结点的Hash值
        int hash = getHash(node);

        // 得到大于该Hash值的所有Map
        SortedMap<Integer, String> subMap = virtualNodes.tailMap(hash);

        //自己加的,考虑到subMap中没有数据，就取virtualNodes中的第一个值
        if (null == subMap || subMap.size() <= 0) {
            subMap = virtualNodes;
        }

        // 第一个Key就是顺时针过去离node最近的那个结点
        Integer i = subMap.firstKey();

        // 返回对应的虚拟节点名称，这里字符串稍微截取一下
        String virtualNode = subMap.get(i);

        return virtualNode.substring(0, virtualNode.indexOf("&&"));
    }

    public static void main(String[] args) {

        String[] nodes = { "127.0.0.1:1111", "221.226.0.1:2222", "10.211.0.1:3333", "223.213.34.67:2341" };

        for (int i = 0; i < nodes.length; i++)
            System.out
                    .println("[" + nodes[i] + "]的hash值为" + getHash(nodes[i]) +
                            ", 被路由到结点[" + getServer(nodes[i]) + "]");
    }
}
```

