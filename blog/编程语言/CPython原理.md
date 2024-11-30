### 语法树
[https://github.com/python/cpython/blob/main/Grammar/python.gram](https://github.com/python/cpython/blob/main/Grammar/python.gram "https://github.com/python/cpython/blob/main/Grammar/python.gram")

如果想给Python加一点语法糖，可以参考这个[https://eli.thegreenplace.net/2010/06/30/python-internals-adding-a-new-statement-to-python](https://eli.thegreenplace.net/2010/06/30/python-internals-adding-a-new-statement-to-python "https://eli.thegreenplace.net/2010/06/30/python-internals-adding-a-new-statement-to-python")

从Python代码---(pgen)--->AST---(asdl_c.py)--->CFG--->ByteCode的大致过程参见
[https://devguide.python.org/compiler/#abstract-syntax-trees-ast](https://devguide.python.org/compiler/#abstract-syntax-trees-ast "https://devguide.python.org/compiler/#abstract-syntax-trees-ast")

关于语法解析和编译前端是另外一个话题。本文主要关注Python特性部分。

### 一切皆对象
Python对象在解释器里是PyObject
```c
typedef struct _object {
    _PyObject_HEAD_EXTRA
    Py_ssize_t ob_refcnt;
    struct _typeobject *ob_type;
} PyObject;
typedef struct {
    PyObject ob_base;
    Py_ssize_t ob_size; /* Number of items in variable part */
} PyVarObject;

struct _longobject {
    PyObject_VAR_HEAD
    digit ob_digit[1];
};
typedef struct _longobject PyLongObject;

typedef struct {
    PyObject_VAR_HEAD
    /* Vector of pointers to list elements.  list[0] is ob_item[0], etc. */
    PyObject **ob_item;

    /* ob_item contains space for 'allocated' elements.  The number
     * currently in use is ob_size.
     * Invariants:
     *     0 <= ob_size <= allocated
     *     len(list) == ob_size
     *     ob_item == NULL implies ob_size == allocated == 0
     * list.sort() temporarily sets allocated to -1 to detect mutations.
     *
     * Items must normally not be NULL, except during construction when
     * the list is not yet visible outside the function that builds it.
     */
    Py_ssize_t allocated;
} PyListObject;

typedef struct {
    PyObject_HEAD
    Py_ssize_t ma_used;
    PyDictKeysObject *ma_keys;
    PyObject **ma_values;
} PyDictObject;

typedef struct _typeobject {
    PyObject_VAR_HEAD
    const char *tp_name; /* For printing, in format "<module>.<name>" */
    Py_ssize_t tp_basicsize, tp_itemsize; /* For allocation */
    ...
} PyTypeObject;
```

使用objgraph + graphviz感受一下对象模型。
```python
obj=[1, 2, "a"]
objgraph.show_backrefs(obj, max_depth=32, filename = "test.dot")
```

![cpython-obj.png](https://ping666.com/wp-content/uploads/2024/09/cpython-obj.png "cpython-obj.png")

### 字节码
每一个新的名字空间或者作用都域编译成一个PyCodeObject，比如模块，函数，类。
```c
/* Bytecode object */
typedef struct {
    PyObject_HEAD
    int co_argcount;        /* #arguments, except *args */
    int co_kwonlyargcount;    /* #keyword only arguments */
    int co_nlocals;        /* #local variables */
    int co_stacksize;        /* #entries needed for evaluation stack */
    int co_flags;        /* CO_..., see below */
    PyObject *co_code;        /* instruction opcodes */
    PyObject *co_consts;    /* list (constants used) */
    PyObject *co_names;        /* list of strings (names used) */
    PyObject *co_varnames;    /* tuple of strings (local variable names) */
    PyObject *co_freevars;    /* tuple of strings (free variable names) */
    PyObject *co_cellvars;      /* tuple of strings (cell variable names) */
    /* The rest doesn't count for hash or comparisons */
    unsigned char *co_cell2arg; /* Maps cell vars which are arguments. */
    PyObject *co_filename;    /* unicode (where it was loaded from) */
    PyObject *co_name;        /* unicode (name, for reference) */
    int co_firstlineno;        /* first source line number */
    PyObject *co_lnotab;    /* string (encoding addr<->lineno mapping) See
                   Objects/lnotab_notes.txt for details. */
    void *co_zombieframe;     /* for optimization only (see frameobject.c) */
    PyObject *co_weakreflist;   /* to support weakrefs to code objects */
} PyCodeObject;
PyCodeObject * PyAST_CompileObject(mod_ty mod, PyObject *filename, PyCompilerFlags *flags, int optimize, PyArena *arena)。
```

在Python代码里可以通过函数的__code__属性直接访问这些信息。更简单的是用dis模块。
![cpython-obj-insight.png](https://ping666.com/wp-content/uploads/2024/09/cpython-obj-insight.png "cpython-obj-insight.png")

第一列是行号（.co_firstlineno），第二列是指令偏移（.co_code里的index），第三列是指令（.co_code[index]里的PyObject值），第四列是参数(.co_consts，.co_names等)。

Python会将PyCodeObject缓存到pyc文件里以备下次使用，除了

- 主模块不缓存
  所以不要在主模块里写太多代码，虽然这一般都不是一个问题。
- 主动禁止
  启动参数-B、环境变量PYTHONDONTWRITEBYTECODE=1、sys.dont_write_bytecode=True效果是一样的。

![cpyhton-obj-bytecode.png](https://ping666.com/wp-content/uploads/2024/09/cpyhton-obj-bytecode.png "cpyhton-obj-bytecode.png")

可以用uncompyle6这个包来反编译pyc文件。
![cpython-obj-umcomp.png](https://ping666.com/wp-content/uploads/2024/09/cpython-obj-umcomp.png "cpython-obj-umcomp.png")

### 运行时
```c
typedef struct {
    PyObject_HEAD
    PyObject *func_code;    /* A code object, the __code__ attribute */
    PyObject *func_globals;    /* A dictionary (other mappings won't do) */
    PyObject *func_defaults;    /* NULL or a tuple */
    PyObject *func_kwdefaults;    /* NULL or a dict */
    PyObject *func_closure;    /* NULL or a tuple of cell objects */
    PyObject *func_doc;        /* The __doc__ attribute, can be anything */
    PyObject *func_name;    /* The __name__ attribute, a string object */
    PyObject *func_dict;    /* The __dict__ attribute, a dict or NULL */
    PyObject *func_weakreflist;    /* List of weak references */
    PyObject *func_module;    /* The __module__ attribute, can be anything */
    PyObject *func_annotations;    /* Annotations, a dict or NULL */
    PyObject *func_qualname;    /* The qualified name */

    /* Invariant:
     *     func_closure contains the bindings for func_code->co_freevars, so
     *     PyTuple_Size(func_closure) == PyCode_GetNumFree(func_code)
     *     (func_closure may be NULL if PyCode_GetNumFree(func_code) == 0).
     */
} PyFunctionObject;
```

通过PyCodeObject生成PyFunctionObject
`PyObject *PyFunction_NewWithQualName(PyObject *code, PyObject *globals, PyObject *qualname)`

函数对象的主体也是一个代码块.func_code，然后是调用时的其他信息，如.func_defaults，.func_kwdefaults，.func_closure等。
符号按LEGB原则查找，local，enclose，global，builtin。
因为普通的int值都是类对象，所以用户自定义类反而没什么特别的。

Python允许多重继承，为了避免diamond问题，类方法的查找原则：老式类（2.2之前的类）按深度优先，新式类（继承于object，3.0之后的默认行为）按广度优先，从左到右。

```c
typedef struct _frame {
    PyObject_VAR_HEAD
    struct _frame *f_back;      /* previous frame, or NULL */
    PyCodeObject *f_code;       /* code segment */
    PyObject *f_builtins;       /* builtin symbol table (PyDictObject) */
    PyObject *f_globals;        /* global symbol table (PyDictObject) */
    PyObject *f_locals;         /* local symbol table (any mapping) */
    PyObject **f_valuestack;    /* points after the last local */
    /* Next free slot in f_valuestack.  Frame creation sets to f_valuestack.
       Frame evaluation usually NULLs it, but a frame that yields sets it
       to the current stack top. */
    PyObject **f_stacktop;
    PyObject *f_trace;          /* Trace function */

        /* In a generator, we need to be able to swap between the exception
           state inside the generator and the exception state of the calling
           frame (which shouldn't be impacted when the generator "yields"
           from an except handler).
           These three fields exist exactly for that, and are unused for
           non-generator frames. See the save_exc_state and swap_exc_state
           functions in ceval.c for details of their use. */
    PyObject *f_exc_type, *f_exc_value, *f_exc_traceback;
    /* Borrowed reference to a generator, or NULL */
    PyObject *f_gen;

    int f_lasti;                /* Last instruction if called */
    /* Call PyFrame_GetLineNumber() instead of reading this field
       directly.  As of 2.3 f_lineno is only valid when tracing is
       active (i.e. when f_trace is set).  At other times we use
       PyCode_Addr2Line to calculate the line from the current
       bytecode index. */
    int f_lineno;               /* Current line number */
    int f_iblock;               /* index in f_blockstack */
    char f_executing;           /* whether the frame is still executing */
    PyTryBlock f_blockstack[CO_MAXBLOCKS]; /* for try and loop blocks */
    PyObject *f_localsplus[1];  /* locals+stack, dynamically sized */
} PyFrameObject;
```

通过PyFunctionObject生成PyFrameObject
`PyFrameObject *PyFrame_New(PyThreadState *tstate, PyCodeObject *code, PyObject *globals, PyObject *locals)`
注意.f_lineno，很多日志库里，当前文件名和行号就是通过实时获取当前帧然后拿到的。

![cpython-runtime.png](https://ping666.com/wp-content/uploads/2024/09/cpython-runtime.png "cpython-runtime.png")

### 解释器执行
这个函数在ceval.c文件里，总共2000千多行，上面test.py涉及到的几个指令以及大致执行过程如下。

```c
PyAPI_FUNC(PyObject *) PyEval_EvalFrameEx(struct _frame *f, int exc){
    PyThreadState *tstate = PyThreadState_GET();    //抢占GIL，通过pthread_mutex_t 、__asm__ volatile("":::"memory")等实现
    tstate->frame = f;
    co = f->f_code;
    first_instr = (unsigned char*) PyBytes_AS_STRING(co->co_code);    //代码指令
    next_instr = first_instr + f->f_lasti + 1;
    stack_pointer = f->f_stacktop;    //栈数据
    for(;;){
        //每隔N次或者收到gil_drop_request消息，尝试释放/重新抢占GIL
        if (eval_breaker || gil_drop_request){
            drop_gil(tstate);    //让其他线程有机会执行
            take_gil(tstate);    //重新抢占
        }
fast_next_opcode：
            opcode = NEXTOP();    //first_instr++
        switch (opcode){
            case NOP:
                goto fast_next_opcode;
              case LOAD_FAST:{
                PyObject *value = GETLOCAL(oparg);
                if (value == NULL) {
                    format_exc_check_arg(PyExc_UnboundLocalError,
                                         UNBOUNDLOCAL_ERROR_MSG,
                                         PyTuple_GetItem(co->co_varnames, oparg));
                    goto error;
                }
                Py_INCREF(value);
                PUSH(value);    //*stack_pointer++ = (value)
                goto fast_next_opcode;
            }
            case LOAD_CONST:{
                PyObject *value = GETITEM(consts, oparg);
                Py_INCREF(value);
                PUSH(value);    //*stack_pointer++ = (value)
                goto fast_next_opcode;
            }
            case BINARY_ADD: {    //压栈和出栈显示的比较清楚
                PyObject *right = POP();    //*--stack_pointer
                PyObject *left = TOP();    //stack_pointer[-1]
                PyObject *sum;
                sum = PyNumber_Add(left, right);
                Py_DECREF(left);
                Py_DECREF(right);
                SET_TOP(sum);    //stack_pointer[-1] = sum
                if (sum == NULL)
                    goto error;
                continue;
            }
            case POP_TOP: {
                PyObject *value = POP();    //*--stack_pointer
                Py_DECREF(value);
                goto fast_next_opcode;
            }
            case JUMP_FORWARD:{
                JUMPBY(oparg);    //next_instr += (oparg)
                goto fast_next_opcode;
            }
            case CALL_FUNCTION:{
                PyObject **sp, *res;
                PCALL(PCALL_ALL);    //4 statistics
                sp = stack_pointer;
    #ifdef WITH_TSC
                res = call_function(&sp, oparg, &intr0, &intr1);
    #else
                res = call_function(&sp, oparg);    //libffi方式的函数调用
    #endif
                stack_pointer = sp;
                PUSH(res);    //*stack_pointer++ = (res)
                if (res == NULL)
                    goto error;
                continue;
            }
            case RETURN_VALUE:{
                retval = POP();    //*--stack_pointer
                why = WHY_RETURN;
                goto fast_block_end;
            }
            ...
        }
    }
    /* pop frame */
exit_eval_frame:
    Py_LeaveRecursiveCall();
    f->f_executing = 0;
    tstate->frame = f->f_back;
    return retval;
}
```
将近100个case，决定了最终的Python语义。

### 垃圾回收
Python回收主要靠对象的引用计数ob_refcnt，当ob_refcnt=0时及时删除。

```c
#define Py_INCREF(op) (                         \
    _Py_INC_REFTOTAL  _Py_REF_DEBUG_COMMA       \    //总体_Py_RefTotal++
    ((PyObject *)(op))->ob_refcnt++)    //单个对象++

#define Py_DECREF(op)                                   \
    do {                                                \
        PyObject *_py_decref_tmp = (PyObject *)(op);    \
        if (_Py_DEC_REFTOTAL  _Py_REF_DEBUG_COMMA       \    //总体_Py_RefTotal--
        --(_py_decref_tmp)->ob_refcnt != 0)             \
            _Py_CHECK_REFCNT(_py_decref_tmp)            \
        else                                            \
        _Py_Dealloc(_py_decref_tmp);                    \    //真正删除
    } while (0)
```

引用计数很简单和很实时，也不会周期性的Stop all the world，但是有些情况不能处理，比如循环引用。
少量的循环引用通过真正的GC处理，同时为了尽减少GC时间，Python把对象做了类似冷热分层，新生成的对象放到0代列表，GC按代回收，旧的会依次挪到1，2代。

```c
PyObject *
_PyObject_GC_Malloc(size_t basicsize)
{
    PyObject *op;
    PyGC_Head *g;
    if (basicsize > PY_SSIZE_T_MAX - sizeof(PyGC_Head))
        return PyErr_NoMemory();
    g = (PyGC_Head *)PyObject_MALLOC(
        sizeof(PyGC_Head) + basicsize);    //任何对象在起始位置放一个GC头
    if (g == NULL)
        return PyErr_NoMemory();
    g->gc.gc_refs = 0;
    _PyGCHead_SET_REFS(g, GC_UNTRACKED);    //这里暂时还没放进跟踪列表，在调用方PyType_GenericAlloc里初始化成功后，才放到0代列表里
    generations[0].count++; /* number of allocated GC objects */
    if (generations[0].count > generations[0].threshold &&
        enabled &&
        generations[0].threshold &&
        !collecting &&
        !PyErr_Occurred()) {
        collecting = 1;
        collect_generations();    //按代GC
        collecting = 0;
    }
    op = FROM_GC(g);    //返回gc头后面的真实对象((PyObject *)(((PyGC_Head *)g)+1))
    return op;
}
```

GC某一代的逻辑在这个函数里
`static Py_ssize_t collect(int generation, Py_ssize_t *n_collected, Py_ssize_t *n_uncollectable, int nofail)`

### GIL
原因就是PyEval_EvalFrameEx函数中对状态PyInterpreterState的抢占使用，是一个问题，但是没有大家认为的那么严重。

![cpython-gil.png](https://ping666.com/wp-content/uploads/2024/09/cpython-gil.png "cpython-gil.png")

CPU密集型程序问题比较突出，解决方法一般有3个：

- 使用PyPy、JPython等解释器实现
  因为GIL是CPython这个特定实现引入的，不是Python本身的原罪。但是不同的实现有一些小区别，不建议项目中途切换。
- 使用多进程模型
  业务一般这样处理。
- 使用c语言扩展，将多核能力放在扩展包里
  大部分基础库作者喜欢这种方式，因为逻辑本身就是C实现的。
