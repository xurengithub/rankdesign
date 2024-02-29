# rankdesign


1.通过题目得知，分数0-10000是小于2^14-1的，redis的zset的score是一个double类型，double除了第一个符号位还有63位可以用来记录数据，63位中前14位记录分数，后49位可以用来记录时间，我们把排行榜系统上线时间当做t1，当前时间是t2,t2减t1是过度的时间毫秒数escapeTime，2^49-1减escapeTime的值作为时间记录下来，最终数据结构如下
符号位             0-10000的分数                                                    最大时间-已经度过的时间
    0                 11111111110111                         000000000000000000000000000000000000000000000000

根据double的组成结构来看，double的最大值表示是 0 11111111110 1111111111111111111111111111111111111111111111111111，所以分数的最大表达方式只能是111111111101111，2^14-9

伪代码无关任何语言：
```
// 更新分数方法
updateScore(string redisKey, string rid, int score) { 
    beforeRedisScore = zscore(redisKey, rid);
  afterRedisScore= generateRedisScore(score);
    if (beforeRedisScore不存在) {
        zadd(redisKey,afterRedisScore, rid);
    } else {
    beforeScore = calScoreFromRedisScore(beforeRedisScore);
    if (score > beforeScore) {
        zadd(redisKey, afterRedisScore, rid);
    }
    }
}

// 通过redis中的score计算分数
calScoreFromRedisScore(double redisScore) {
    byteArray = doubleToByteArray(redisScore)
        return 取出1-14位字节转化为int分数;
}

// 通过分数生成redis中的score
generateRedisScore(int score) {
    timeGap = (2^49-1) - now() - 1678183841000;
    part1 = score&(2^14-9);
       part2 = timeGap & (2^49 - 1);
        longScore = (part1 << 49) & part2;
        return longScore转化为字节数组再通过字节数组转化为double；
}

// 查询某个玩家及前后10名玩家分数及名次
queryRidAndAroundRankInfo(string redisKey, string rid) {
    myIndex = zreverank(redisKey, rid); 
    startIndex = myIndex - 10;
    list = zrevrangewithscores(redisKey,startIndex < 0 ? 0 : startIndex ,myIndex+10);
    return list;
}
```
queryRidAndAroundRankInfo如果出现了两个查询数据不一致，那可以采用执行lua脚本方式将两个查询弄成一个原子操作；如何有其他需求也可以用redis做，那可以将实现抽象出来，比如多个数字字段排序，并且大小也不是很大，还可以对字段顺序和进行定义

2.如果还要依据名称来排序那第一种实现方式就不行了，就需要自己去实现排序了，可以在内存中去实现排序，在redis通过hash去存储排行榜的数据，启动服务时通过加载redis的数据到内存中，每条角色数据都是一个对象，用一个有序list去存储对象，对象实现排序比较的方法定义对象顺序, 新增rid的排行数据时，先判断内存中是否有，有则取出来判断分数是否有变化，有变化先写redis写成功了，更新到内存中，没变化则不做任何处理。如果内存中没有数据也是重复操作

Java语言实现对象比较方法、equals方法及hashCode方法：
```
public int compareTo(RankInfo o) {
    var scoreComparison = Integer.compare(o.getScore(), score);
    if (scoreComparison != 0) {
        return scoreComparison;
    }
    var timeComparison = Long.compare(time, o.getTime());
    if (timeComparison != 0) {
        return timeComparison;
    }
    var levelComparison = Integer.compare(o.getLevel(), level);
    if (levelComparison != 0) {
        return levelComparison;
    }
    return name.compareTo(o.getName());
}

public boolean equals(RankInfo o) {
    if (this == 0) return true;
    if (o == null) return false;
    return score == o.getScore() && time == o.getTime() && level == o.getLevel() && name.equals(o.getName());
}

public int hashCode() {
    return Objects.hash(score, time, level, name);
}
```
