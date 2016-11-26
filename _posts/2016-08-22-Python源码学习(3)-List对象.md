---
category: Python
---

## List对象

### 定义

```

typedef struct {
	PyObject_VAR_HEAD
	PyObject **ob_item;  	//ob_item为指向元素列表的指针，list[0]即是ob_item[0]
	int allocated;  //当前为List申请的内存元素个数
};

```

### 创建对象

```c

PyObject * PyList_New(int size){
	PyListObject *op;
	size_t nbytes;

	//1.内存数量计算，溢出检查
	nbytes = size * sizeof(PyObject *);
	if (nbytes / siezof(PyObject *) != (size_t)size)
		return PyErr_NoMemory();

	//2.为PyListObject对象申请空间
	if (num_free_lists){
		//缓冲池可用
		num_free_lists--;
		op = free_lists[num_free_lists];
		_Py_NewReference((PyObject *)op);
	}
	else {
		//缓冲池不可用
		op = PyObject_GC_NEW(PyListObject, &PyList_Type);
	}

	//3.为PyListObject对象中维护的元素列表申请空间
	if (size<0)
		op->ob_item = NULL;
	else {
		op->ob_item = (PyObject **) PyMem_MALLOC(nbytes);
		memset(op->ob_item, 0, nbytes);
	}
	op->ob_size = size;
	op->allocated = size;
	return (PyObject *)op;
}

```

### 对象缓冲池

使用`free_lists`数组，容量为80个，每一个都是PyListObject对象指针。

`num_free_lists`作为缓冲池数组的指针，初始为0。虚拟机创建第一个PyListObject时并不会使用。当删除一个PyListObject对象时检查`free_lists`数组有没有填满PyListObject指针，如果没有达到最大值，就将当前的对象赋值给`free_lists[num_free_lists]`，并将num_free_lists加1。

```c

#define MAXFREELISTS 80
static PyListObject *free_lists[MAXFREELISTS];
static int num_free_lists=0;

```


### 调整list列表容量

```
static int list_resize(PyListObject * self, int newsize){
	PyObject **item;
	int allocated = self->allocated;

	//不需要重新申请内存（满足allocated/2 <= newsize <= allocated）
	if (allocated >= newsize && newsize >= (allocated >> 1)){
		self->ob_size = newsize;
		return 0;
	}

	//计算重新申请的内存大小
	new_allocated = (newsize >> 3) + (newsize < 9 ? 3 : 6) + newsize;
	if (newsize == 0)
		new_allocated = 0;
	items = self->ob_item;
	PyMem_RESIZE(items, PyObject *, new_allocated);
	self->ob_item = items;
	self->ob_size = newsize;
	self->ob_allocated = new_allocated;
	return 0;
}
```
