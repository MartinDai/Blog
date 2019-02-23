# 前言

> 最近在研究布隆过滤器（如果不了解什么是布隆过滤器的，推荐看这篇[如何判断一个元素在亿级数据中是否存在？][1]了解），发现Guava提供了封装好的类，但是只能单机使用，一般现在的应用都是部署在分布式系统的，所以想找个可以在分布式系统下使用的布隆过滤器，找了半天只找到一个基于redis开发的模块项目[ReBloom][2]，但是这个是需要额外安装的，而且文档里只说了怎么在docker下运行，没研究过docker所以放弃了。后来找到一篇博客讲怎么利用布隆过滤器统计消息未读数的（博客地址不记得了，是一位淘宝同学写的），博客最后放了一份整合redis和bloomFilter的代码demo，详见[BloomFilter.java][3]，看了下实现比较简单，但是使用方式不是我想要的，所以参考着自己整理了一份。

# BloomFilterHelper

```
package com.doodl6.springmvc.service.cache.redis;

import com.google.common.base.Preconditions;
import com.google.common.hash.Funnel;
import com.google.common.hash.Hashing;

public class BloomFilterHelper<T> {

    private int numHashFunctions;

    private int bitSize;

    private Funnel<T> funnel;

    public BloomFilterHelper(Funnel<T> funnel, int expectedInsertions, double fpp) {
        Preconditions.checkArgument(funnel != null, "funnel不能为空");
        this.funnel = funnel;
        bitSize = optimalNumOfBits(expectedInsertions, fpp);
        numHashFunctions = optimalNumOfHashFunctions(expectedInsertions, bitSize);
    }

    int[] murmurHashOffset(T value) {
        int[] offset = new int[numHashFunctions];

        long hash64 = Hashing.murmur3_128().hashObject(value, funnel).asLong();
        int hash1 = (int) hash64;
        int hash2 = (int) (hash64 >>> 32);
        for (int i = 1; i <= numHashFunctions; i++) {
            int nextHash = hash1 + i * hash2;
            if (nextHash < 0) {
                nextHash = ~nextHash;
            }
            offset[i - 1] = nextHash % bitSize;
        }

        return offset;
    }

    /**
     * 计算bit数组长度
     */
    private int optimalNumOfBits(long n, double p) {
        if (p == 0) {
            p = Double.MIN_VALUE;
        }
        return (int) (-n * Math.log(p) / (Math.log(2) * Math.log(2)));
    }

    /**
     * 计算hash方法执行次数
     */
    private int optimalNumOfHashFunctions(long n, long m) {
        return Math.max(1, (int) Math.round((double) m / n * Math.log(2)));
    }
}
```

**BloomFilterHelper**是实现功能的关键，包含了计算bitmap的核心算法，其实大部分代码都是来源于Guava库里面的BloomFilterStrategies类，但是因为这个类是专门为Guava的BloomFilter类使用的，所以没有对外暴露一些重要的算法逻辑。

再来看怎么结合redis一起使用BloomFilterHelper

# RedisService

```
package com.doodl6.springmvc.service.cache.redis;

import com.google.common.base.Preconditions;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import javax.annotation.Resource;
import java.util.Collection;
import java.util.Map;
import java.util.concurrent.TimeUnit;

@Service
public class RedisService {

    @Resource
    private RedisTemplate<String, Object> redisTemplate;

    /**
     * 根据给定的布隆过滤器添加值
     */
    public <T> void addByBloomFilter(BloomFilterHelper<T> bloomFilterHelper, String key, T value) {
        Preconditions.checkArgument(bloomFilterHelper != null, "bloomFilterHelper不能为空");
        int[] offset = bloomFilterHelper.murmurHashOffset(value);
        for (int i : offset) {
            redisTemplate.opsForValue().setBit(key, i, true);
        }
    }

    /**
     * 根据给定的布隆过滤器判断值是否存在
     */
    public <T> boolean includeByBloomFilter(BloomFilterHelper<T> bloomFilterHelper, String key, T value) {
        Preconditions.checkArgument(bloomFilterHelper != null, "bloomFilterHelper不能为空");
        int[] offset = bloomFilterHelper.murmurHashOffset(value);
        for (int i : offset) {
            if (!redisTemplate.opsForValue().getBit(key, i)) {
                return false;
            }
        }

        return true;
    }
}

```


RedisService很简单，只有两个方法

**addByBloomFilter**，往redis里面添加元素

**includeByBloomFilter**，检查元素是否在redis bloomFilter里面

这里redis的客户端使用的是spring-data-redis封装的，可以在我的项目[SpringMVC-Project][4]中查看完整的使用代码。


  [1]: https://crossoverjie.top/2018/11/26/guava/guava-bloom-filter/
  [2]: https://github.com/RedisLabsModules/rebloom
  [3]: https://github.com/olylakers/RedisBloomFilter/blob/master/src/main/java/org/olylakers/bloomfilter/BloomFilter.java
  [4]: https://github.com/MartinDai/SpringMVC-Project