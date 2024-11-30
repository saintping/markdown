### 前言
OOP（Ordinary Object Pointer，普通对象指针）。Java类加载到虚拟机里对应的是Klass，Java对象在虚拟机里对应的是OOP，OOP就是Java对象在虚拟机中的根。

### Klass
Klass在方法区，内容大部分是从class文件解析而来。包括类的常量池、字段、方法等。
`src\hotspot\share\oops\klass.hpp`
```cpp
class Klass : public Metadata {
 protected:
  // If you add a new field that points to any metaspace object, you
  // must add this field to Klass::metaspace_pointers_do().

  // note: put frequently-used fields together at start of klass structure
  // for better cache behavior (may not make much of a difference but sure won't hurt)
  enum { _primary_super_limit = 8 };

  // The "layout helper" is a combined descriptor of object layout.
  // For klasses which are neither instance nor array, the value is zero.
  //
  // For instances, layout helper is a positive number, the instance size.
  // This size is already passed through align_object_size and scaled to bytes.
  // The low order bit is set if instances of this class cannot be
  // allocated using the fastpath.
  //
  // For arrays, layout helper is a negative number, containing four
  // distinct bytes, as follows:
  //    MSB:[tag, hsz, ebt, log2(esz)]:LSB
  // where:
  //    tag is 0x80 if the elements are oops, 0xC0 if non-oops
  //    hsz is array header size in bytes (i.e., offset of first element)
  //    ebt is the BasicType of the elements
  //    esz is the element size in bytes
  // This packed word is arranged so as to be quickly unpacked by the
  // various fast paths that use the various subfields.
  //
  // The esz bits can be used directly by a SLL instruction, without masking.
  //
  // Note that the array-kind tag looks like 0x00 for instance klasses,
  // since their length in bytes is always less than 24Mb.
  //
  // Final note:  This comes first, immediately after C++ vtable,
  // because it is frequently queried.
  jint        _layout_helper;

  // Klass identifier used to implement devirtualized oop closure dispatching.
  const KlassID _id;

  // The fields _super_check_offset, _secondary_super_cache, _secondary_supers
  // and _primary_supers all help make fast subtype checks.  See big discussion
  // in doc/server_compiler/checktype.txt
  //
  // Where to look to observe a supertype (it is &_secondary_super_cache for
  // secondary supers, else is &_primary_supers[depth()].
  juint       _super_check_offset;

  // Class name.  Instance classes: java/lang/String, etc.  Array classes: [I,
  // [Ljava/lang/String;, etc.  Set to zero for all other kinds of classes.
  Symbol*     _name;

  // Cache of last observed secondary supertype
  Klass*      _secondary_super_cache;
  // Array of all secondary supertypes
  Array<Klass*>* _secondary_supers;
  // Ordered list of all primary supertypes
  Klass*      _primary_supers[_primary_super_limit];
  // java/lang/Class instance mirroring this class
  OopHandle _java_mirror;
  // Superclass
  Klass*      _super;
  // First subclass (NULL if none); _subklass->next_sibling() is next one
  Klass*      _subklass;
  // Sibling link (or NULL); links all subklasses of a klass
  Klass*      _next_sibling;

  // All klasses loaded by a class loader are chained through these links
  Klass*      _next_link;

  // The VM's representation of the ClassLoader used to load this class.
  // Provide access the corresponding instance java.lang.ClassLoader.
  ClassLoaderData* _class_loader_data;

  jint        _modifier_flags;  // Processed access flags, for use by Class.getModifiers.
  AccessFlags _access_flags;    // Access flags. The class/interface distinction is stored here.

  JFR_ONLY(DEFINE_TRACE_ID_FIELD;)

  // Biased locking implementation and statistics
  // (the 64-bit chunk goes first, to avoid some fragmentation)
  jlong    _last_biased_lock_bulk_revocation_time;
  markOop  _prototype_header;   // Used when biased locking is both enabled and disabled for this type
  jint     _biased_lock_revocation_count;

  // vtable length
  int _vtable_len;

private:
  // This is an index into FileMapHeader::_shared_path_table[], to
  // associate this class with the JAR file where it's loaded from during
  // dump time. If a class is not loaded from the shared archive, this field is
  // -1.
  jshort _shared_class_path_index;

#if INCLUDE_CDS
  // Flags of the current shared class.
  u2     _shared_class_flags;
  enum {
    _has_raw_archived_mirror = 1,
    _has_signer_and_not_archived = 1 << 2
  };
#endif
  // The _archived_mirror is set at CDS dump time pointing to the cached mirror
  // in the open archive heap region when archiving java object is supported.
  CDS_JAVA_HEAP_ONLY(narrowOop _archived_mirror;)
  ...
}
```

### Oop和markOop
```cpp
class oopDesc {
 private:
  volatile markOop _mark;
  union _metadata {
    Klass*      _klass;
    narrowKlass _compressed_klass;
  } _metadata;
}

typedef class   markOopDesc*                markOop;

class markOopDesc: public oopDesc {
}

#ifndef CHECK_UNHANDLED_OOPS
  typedef class oopDesc*                            oop;
  typedef class   instanceOopDesc*            instanceOop;
  typedef class   arrayOopDesc*                    arrayOop;
  typedef class     objArrayOopDesc*            objArrayOop;
  typedef class     typeArrayOopDesc*            typeArrayOop;    
#else
  class oop {
    oopDesc* _o;
 }
#endif
```

markOopDesc是oopDesc的子类，oop是oopDesc的指针，markOop是markOopDesc的指针。
markOop有两个字段_mark和_metadata。后面是oop关联的类信息Klass。_mark标记位的信息包括：hash值、垃圾回收标记、bias锁、monitor等。
32位和64位机器下布局见注释

```cpp
// The markOop describes the header of an object.
//
// Note that the mark is not a real oop but just a word.
// It is placed in the oop hierarchy for historical reasons.
//
// Bit-format of an object header (most significant first, big endian layout below):
//
//  32 bits:
//  --------
//             hash:25 ------------>| age:4    biased_lock:1 lock:2 (normal object)
//             JavaThread*:23 epoch:2 age:4    biased_lock:1 lock:2 (biased object)
//             size:32 ------------------------------------------>| (CMS free block)
//             PromotedObject*:29 ---------->| promo_bits:3 ----->| (CMS promoted object)
//
//  64 bits:
//  --------
//  unused:25 hash:31 -->| unused:1   age:4    biased_lock:1 lock:2 (normal object)
//  JavaThread*:54 epoch:2 unused:1   age:4    biased_lock:1 lock:2 (biased object)
//  PromotedObject*:61 --------------------->| promo_bits:3 ----->| (CMS promoted object)
//  size:64 ----------------------------------------------------->| (CMS free block)
//
//  unused:25 hash:31 -->| cms_free:1 age:4    biased_lock:1 lock:2 (COOPs && normal object)
//  JavaThread*:54 epoch:2 cms_free:1 age:4    biased_lock:1 lock:2 (COOPs && biased object)
//  narrowOop:32 unused:24 cms_free:1 unused:4 promo_bits:3 ----->| (COOPs && CMS promoted object)
//  unused:21 size:35 -->| cms_free:1 unused:7 ------------------>| (COOPs && CMS free block)
```

垃圾回收的一个重要工作就是找到堆栈中的所有OOP，并且分析可达性。

### 对象内存布局
比如类B继承A
`Class B ： public A`

C++语言里对象的内存布局是：B类的虚函数表 + A类对象属性 + B类对象属性。如果有更多继承层次，就+C、+D一直排下去。
Java对象的内存布局是：A类对象属性 + B类对象属性。没有虚函数表，这种布局在垃圾回收时有优势。

>// OBJECT hierarchy
// This hierarchy is a representation hierarchy, i.e. if A is a superclass
// of B, A's representation is a prefix of B's representation.
