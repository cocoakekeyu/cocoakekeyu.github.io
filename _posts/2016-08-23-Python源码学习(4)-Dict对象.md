---
category: Python
---

## Dict对象

PyDictObject对象使用散列表对键进行搜索

### 关联容器entry

```
typedef struct {
	Py_ssize_t me_hash;  //缓存键的hash值
	PyObject *me_key;		//键的指针
	PyObject *me_value;		//值的指针
} PyDictEntry;
```

通过设置`me_key`和`me_value`的不同值，代表一个entry的不同状态，共有三种：

- Unused态：`me_key`和`me_value`都为NULL。表明该entry没有存储过键和相应的值。
- Active态： `me_key`和`me_value`都不为NULL。
- Dummy态： `me_key`指向一个dummy对象（其实是一个值为‘dummy’的PyStringObject）。当entry存储过对象并被删除后就处于Dummy态，保证探测链不会中断，否则其后面的有键值的entry就不能通过探测链找到。

### PyDictObject

```
#define PyDict_MINSIZE 8
typedef struct _dictobject PyDictObject;

struct _dictobject {
	PyObject_HEAD
	Py_ssize_t ma_fill;		//元素个数（包含了Active和Dummy态的Entry）
	Py_ssize_t ma_used;		//元素个数（Active）
	Py_ssize_t ma_mask;		//所拥有的entry数量
	PyDictEntry *ma_table;  //指向关联的Entry，若少于8个，指向内置的ma_samlltable。不然将指向新申请的内存空间存储Entry。
	PyDictEntry *(*ma_loopup)(PyDictObject *mp, PyObject *key, long hash);
	PyDictEntry ma_samlltable[PyDict_MINSIZE];  //新建一个PyDictObject对象时，内带PyDict_MINSIZE（8）个Entry，提高效率。
}
```

### PyDictObject对象创建

通过函数`PyDict_New`创建。

```
PyObject *
PyDict_New(void)
{
    register PyDictObject *mp;
    if (dummy == NULL) { /* Auto-initialize dummy */
        dummy = PyString_FromString("<dummy key>");
        if (dummy == NULL)
            return NULL;
#ifdef SHOW_CONVERSION_COUNTS
        Py_AtExit(show_counts);
#endif
#ifdef SHOW_ALLOC_COUNT
        Py_AtExit(show_alloc);
#endif
#ifdef SHOW_TRACK_COUNT
        Py_AtExit(show_track);
#endif
    }


		/* numfree为缓冲池指针 */
    if (numfree) {
        mp = free_list[--numfree];
        assert (mp != NULL);
        assert (Py_TYPE(mp) == &PyDict_Type);
        _Py_NewReference((PyObject *)mp);
        if (mp->ma_fill) {
            EMPTY_TO_MINSIZE(mp);
        } else {
            /* At least set ma_table and ma_mask; these are wrong
               if an empty but presized dict is added to freelist */
            INIT_NONZERO_DICT_SLOTS(mp);
        }
        assert (mp->ma_used == 0);
        assert (mp->ma_table == mp->ma_smalltable);
        assert (mp->ma_mask == PyDict_MINSIZE - 1);
#ifdef SHOW_ALLOC_COUNT
        count_reuse++;
#endif
    } else {
        mp = PyObject_GC_New(PyDictObject, &PyDict_Type);
        if (mp == NULL)
            return NULL;
        EMPTY_TO_MINSIZE(mp);  //初始化工作：将ma_samlltable清零，同时设置ma_used和ma_fill为0。将ma_table指向ma_samlltable,设置ma_mask为7。
#ifdef SHOW_ALLOC_COUNT
        count_alloc++;
#endif
    }
    mp->ma_lookup = lookdict_string;
#ifdef SHOW_TRACK_COUNT
    count_untracked++;
#endif
#ifdef SHOW_CONVERSION_COUNTS
    ++created;
#endif
    return (PyObject *)mp;
}
```

### PyDictObject元素搜索

两种搜索策略，lookdict和lookdict_string。

```
static PyDictEntry *
lookdict(PyDictObject *mp, PyObject *key, register long hash)
{
    register size_t i;
    register size_t perturb;
    register PyDictEntry *freeslot;   //保存探测链上第一个Dummy态的PyDictEntry。
    register size_t mask = (size_t)mp->ma_mask;
    PyDictEntry *ep0 = mp->ma_table;
    register PyDictEntry *ep;
    register int cmp;
    PyObject *startkey;

		/* 定位探测链上第一个entry */
    i = (size_t)hash & mask;
    ep = &ep0[i];

		/* entry为Unused态(me_key==NULL)或者entry的me_key与待搜索的key匹配 */
    if (ep->me_key == NULL || ep->me_key == key)
        return ep;

		/* entry为Dummy态 */
    if (ep->me_key == dummy)
        freeslot = ep;
    else {

				/* 检查Active态的entry */
        if (ep->me_hash == hash) {
            startkey = ep->me_key;
            Py_INCREF(startkey);
            cmp = PyObject_RichCompareBool(startkey, key, Py_EQ);
            Py_DECREF(startkey);
						/* python2.5新增，比较产生异常时返回NULL */
            if (cmp < 0)
                return NULL;
            if (ep0 == mp->ma_table && ep->me_key == startkey) {
                if (cmp > 0)
                    return ep;
            }
            else {
                /* The compare did major nasty stuff to the
                 * dict:  start over.
                 * XXX A clever adversary could prevent this
                 * XXX from terminating.
                 */
                return lookdict(mp, key, hash);
            }
        }
				/* 到这里说明cmp=0，Active态的entry已经被占用了，设置freeslot */
        freeslot = NULL;
    }

    /* In the loop, me_key == dummy is by far (factor of 100s) the
       least likely outcome, so test for that last. */
    for (perturb = hash; ; perturb >>= PERTURB_SHIFT) {
        i = (i << 2) + i + perturb + 1;
        ep = &ep0[i & mask];
        if (ep->me_key == NULL)
            return freeslot == NULL ? ep : freeslot;
        if (ep->me_key == key)
            return ep;
        if (ep->me_hash == hash && ep->me_key != dummy) {
            startkey = ep->me_key;
            Py_INCREF(startkey);
            cmp = PyObject_RichCompareBool(startkey, key, Py_EQ);
            Py_DECREF(startkey);
            if (cmp < 0)
                return NULL;
            if (ep0 == mp->ma_table && ep->me_key == startkey) {
                if (cmp > 0)
                    return ep;
            }
            else {
                /* The compare did major nasty stuff to the
                 * dict:  start over.
                 * XXX A clever adversary could prevent this
                 * XXX from terminating.
                 */
                return lookdict(mp, key, hash);
            }
        }
        else if (ep->me_key == dummy && freeslot == NULL)
            freeslot = ep;
    }
    assert(0);          /* NOT REACHED */
    return 0;
}
```
