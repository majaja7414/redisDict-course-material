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

###兩種Entry(作為節點使用)
用於形成chaining，解決collision，分成有放key和沒放key兩種
```
struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;     /* Next entry in the same hash bucket. */
};

typedef struct {
    void *key;
    dictEntry *next;
} dictEntryNoValue;
```

```
```

##自動擴容rehashing
```
int _dictExpand(dict *d, unsigned long size, int* malloc_failed)
{
    if (malloc_failed) *malloc_failed = 0;

    /* the size is invalid if it is smaller than the number of
     * elements already inside the hash table */
    if (dictIsRehashing(d) || d->ht_used[0] > size)
        return DICT_ERR;

    /* the new hash table */
    dictEntry **new_ht_table;
    unsigned long new_ht_used;
    signed char new_ht_size_exp = _dictNextExp(size);

    /* Detect overflows */
    size_t newsize = DICTHT_SIZE(new_ht_size_exp);
    if (newsize < size || newsize * sizeof(dictEntry*) < newsize)
        return DICT_ERR;

    /* Rehashing to the same table size is not useful. */
    if (new_ht_size_exp == d->ht_size_exp[0]) return DICT_ERR;    //如果rehash後table的size還是一樣大

    /* Allocate the new hash table and initialize all pointers to NULL */
    if (malloc_failed) {
        new_ht_table = ztrycalloc(newsize*sizeof(dictEntry*));
        *malloc_failed = new_ht_table == NULL;
        if (*malloc_failed)
            return DICT_ERR;
    } else
        new_ht_table = zcalloc(newsize*sizeof(dictEntry*));

    new_ht_used = 0;

    /* Prepare a second hash table for incremental rehashing.
     * We do this even for the first initialization, so that we can trigger the
     * rehashingStarted more conveniently, we will clean it up right after. */
    d->ht_size_exp[1] = new_ht_size_exp;
    d->ht_used[1] = new_ht_used;
    d->ht_table[1] = new_ht_table;
    d->rehashidx = 0;
    if (d->type->rehashingStarted) d->type->rehashingStarted(d);

    /* Is this the first initialization or is the first hash table empty? If so
     * it's not really a rehashing, we can just set the first hash table so that
     * it can accept keys. */
    if (d->ht_table[0] == NULL || d->ht_used[0] == 0) {
        if (d->type->rehashingCompleted) d->type->rehashingCompleted(d);
        if (d->ht_table[0]) zfree(d->ht_table[0]);
        d->ht_size_exp[0] = new_ht_size_exp;
        d->ht_used[0] = new_ht_used;
        d->ht_table[0] = new_ht_table;
        _dictReset(d, 1);
        d->rehashidx = -1;
        return DICT_OK;
    }

    return DICT_OK;
}
```
參數n*10決定最高同時訪問的空桶量，考慮極端情況下，只有5和99999這兩個bucket有資料，當轉移完5後，若沒限制訪問空桶量的限制，會從5一路訪問99999，造成其他事務必須等待，造成卡頓。<br>

取s0、s1分別為ht_table[0]和[1]的大小，此處exp指冪次，redis讓table的大小總是2^n，需要得到實際大小時，再使用DICTHT_SIZE這個類似macro的設計：當傳入一個數，假設是x，代表大小為table大小為2^x，首先檢查是否為-1，是的話回傳0，不是的話回傳x的二進位左移一位(等同10進位中n乘二的操作)，假設x=4，二進位表示為1000，左移一位得10000，即16。<br>
```
#define DICTHT_SIZE(exp) ((exp) == -1 ? 0 : (unsigned long)1<<(exp))
```
```
int dictRehash(dict *d, int n) {
    int empty_visits = n*10; /* Max number of empty buckets to visit. */
    unsigned long s0 = DICTHT_SIZE(d->ht_size_exp[0]);
    unsigned long s1 = DICTHT_SIZE(d->ht_size_exp[1]);
```

以下有兩種條件會先跳過rehashing：
    1.dict的大小改變暫時被禁止(DICT_RESIZE_FORBID)，!dictIsRehashing(d)指dict
    2.dict的大小改變暫時被忽略、跳過(DICT_RESIZE_AVOID)，且未達到強迫resize的條件(dict_force_resize_ratio)，即目前的dict大小還"能用"，不著急rehashing。
```
    if (dict_can_resize == DICT_RESIZE_FORBID || !dictIsRehashing(d)) return 0;
    if (dict_can_resize == DICT_RESIZE_AVOID && 
        ((s1 > s0 && s1 / s0 < dict_force_resize_ratio) ||
         (s1 < s0 && s0 / s1 < dict_force_resize_ratio)))
    {
        return 0;
    }
```
assert檢查rehashidx有無跑掉
```
    while(n-- && d->ht_used[0] != 0) {
        dictEntry *de, *nextde;

        /* Note that rehashidx can't overflow as we are sure there are more
         * elements because ht[0].used != 0 */
        assert(DICTHT_SIZE(d->ht_size_exp[0]) > (unsigned long)d->rehashidx);
```

如果遇到空桶，直接跳過，直到找到非空桶，若達到empty_visits上限，結束這一次的rehashing。
```
        while(d->ht_table[0][d->rehashidx] == NULL) {
            d->rehashidx++;
            if (--empty_visits == 0) return 1;
        }
```

以下是搬遷bucket的過程， 如果新table比舊table大，透過key取hash-value和新table的大小MASK來確定鍵在新table中的位置；如果新表比舊表小，直接使用rehashidx 和新表的大小MASK確定位置。
```
        de = d->ht_table[0][d->rehashidx];
        /* Move all the keys in this bucket from the old to the new hash HT */
        while(de) {
            uint64_t h;

            nextde = dictGetNext(de);
            void *key = dictGetKey(de);
            /* Get the index in the new hash table */
            if (d->ht_size_exp[1] > d->ht_size_exp[0]) {
                h = dictHashKey(d, key) & DICTHT_SIZE_MASK(d->ht_size_exp[1]);
            } else {
                /* We're shrinking the table. The tables sizes are powers of
                 * two, so we simply mask the bucket index in the larger table
                 * to get the bucket index in the smaller table. */
                h = d->rehashidx & DICTHT_SIZE_MASK(d->ht_size_exp[1]);
            }
```

如果這個dict只儲存key而不存value，採取特殊優化處理
```
            if (d->type->no_value) {
                if (d->type->keys_are_odd && !d->ht_table[1][h]) {
                    /* Destination bucket is empty and we can store the key
                     * directly without an allocated entry. Free the old entry
                     * if it's an allocated entry.
                     *
                     * TODO: Add a flag 'keys_are_even' and if set, we can use
                     * this optimization for these dicts too. We can set the LSB
                     * bit when stored as a dict entry and clear it again when
                     * we need the key back. */
                    assert(entryIsKey(key));
                    if (!entryIsKey(de)) zfree(decodeMaskedPtr(de));
                    de = key;
                } else if (entryIsKey(de)) {
                    /* We don't have an allocated entry but we need one. */
                    de = createEntryNoValue(key, d->ht_table[1][h]);
                } else {
                    /* Just move the existing entry to the destination table and
                     * update the 'next' field. */
                    assert(entryIsNoValue(de));
                    dictSetNext(de, d->ht_table[1][h]);
                }
            }
```
正常移動
```
            else {
                dictSetNext(de, d->ht_table[1][h]);
            }
            d->ht_table[1][h] = de;
            d->ht_used[0]--;
            d->ht_used[1]++;
            de = nextde;
        }
        d->ht_table[0][d->rehashidx] = NULL;
        d->rehashidx++;
    }    //while迴圈結束
```
檢查是否全數rehash完畢，如果ht_table[0]已經沒有元素了，就釋放ht_table[0]，並且將ht_table[0]改為ht_table[1]，完成rehashing
```
    /* Check if we already rehashed the whole table... */
    if (d->ht_used[0] == 0) {
        if (d->type->rehashingCompleted) d->type->rehashingCompleted(d);
        zfree(d->ht_table[0]);
        /* Copy the new ht onto the old one */
        d->ht_table[0] = d->ht_table[1];
        d->ht_used[0] = d->ht_used[1];
        d->ht_size_exp[0] = d->ht_size_exp[1];
        _dictReset(d, 1);
        d->rehashidx = -1;
        return 0;
    }

    /* More to rehash... */
    return 1;
}
```

