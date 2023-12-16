# redisDictAnalying
Trying to analyze dict.c and dict.h in redis.

##基礎資料結構介紹
```
struct dict {
    dictType *type;  //存放各式函數指標

    dictEntry **ht_table[2];  //兩個hashtable的入口
    unsigned long ht_used[2];  //紀錄hashtable的已使用bucket數

    long rehashidx; //-1表示未目前沒有進行rehash(擴容)， 0~n表示目前rehash進行到第n個bucket，且第n個以前的buckets皆已完成

    /* Keep small vars at end for optimal (minimal) struct padding */
    int16_t pauserehash; /* If >0 rehashing is paused (<0 indicates coding error) */
    signed char ht_size_exp[2]; /* exponent of size. (size = 1<<exp) */
    void *metadata[];
};
```

```
```
```
```
```
```
```
```
```
```
```
```
