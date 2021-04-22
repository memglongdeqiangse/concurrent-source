# hotspot 中对象的表示机制 ：OOP-Klass二分模型

hotspot由C++实现，为什么Java的一个类不对应C++的一个类，Java类的实例对应C++类的实例？（以下内容为《hotspot实战》和几篇博文的读后笔记）

## OOP-Klass二分模型

**OOP/OOPS：**ordinary object pointer，普通对象指针，**用来描述对象的实例信息**。

**Klass：**Java类的C++对等体，**用来描述Java类**。（1.实现语言层面的Java类 2.实现Java对象的分发功能）

oop的职能是表示对象的实例数据，没必要每个对象都有vtable指针。Klass（用来描述Java类）的对象中有vtable的话，就可以根据Java对象的实际类型进行C++的分发，这样OOP对象就可以通过Klass找到所有的虚函数了。

![1618843008885](./images\1618843008885.png)

oop.hpp中的一段注释

```c++
// oopDesc is the top baseclass for objects classes. The {name}Desc classes describe
// the format of Java objects so the fields can be accessed from C++.
// oopDesc is abstract.
// (see oopHierarchy for complete oop class hierarchy)
//
// no virtual functions allowed
//----------翻译-----------
//oopDesc是所有OOP对象的基类。{name}Desc类描述了Java对象的格式，因此可以从c++访问字段。
//oopDesc是抽象的。参见oopHierarchy获取完整的oop类层次结构。不允许使用虚拟函数
```

oop的继承关系来自[一位大佬的文章](<https://blog.lovezhy.cc/2019/02/16/HotSpot%E5%8E%9F%E7%90%86%E6%8C%87%E5%8D%97-oop-klass%E6%A8%A1%E5%9E%8B/>)

> 17中oop继承链
>
> ```c++
> typedef class oopDesc*                            oop;
> typedef class   instanceOopDesc*            instanceOop;
> typedef class   methodOopDesc*                    methodOop;
> typedef class   constMethodOopDesc*            constMethodOop;
> typedef class   methodDataOopDesc*            methodDataOop;
> typedef class   arrayOopDesc*                    arrayOop;
> typedef class     objArrayOopDesc*            objArrayOop;
> typedef class     typeArrayOopDesc*            typeArrayOop;
> typedef class   constantPoolOopDesc*            constantPoolOop;
> typedef class   constantPoolCacheOopDesc*   constantPoolCacheOop;
> typedef class   klassOopDesc*                    klassOop;
> typedef class   markOopDesc*                    markOop;
> typedef class   compiledICHolderOopDesc*    compiledICHolderOop;
> ```
>
> 1.7中klass继承链
>
> ```java
> class Klass;
> class   instanceKlass;
> class     instanceMirrorKlass;
> class     instanceRefKlass;
> class   methodKlass;
> class   constMethodKlass;
> class   methodDataKlass;
> class   klassKlass;
> class     instanceKlassKlass;
> class     arrayKlassKlass;
> class       objArrayKlassKlass;
> class       typeArrayKlassKlass;
> class   arrayKlass;
> class     objArrayKlass;
> class     typeArrayKlass;
> class   constantPoolKlass;
> class   constantPoolCacheKlass;
> class   compiledICHolderKlass;
> ```
>
> 可以说是非常多，但是到了1.8中，就变的很少

> 1.8 oop继承链
>
> ```c++
> typedef class oopDesc*                            oop;
> typedef class   instanceOopDesc*            instanceOop;
> typedef class   arrayOopDesc*                    arrayOop;
> typedef class     objArrayOopDesc*            objArrayOop;
> typedef class     typeArrayOopDesc*            typeArrayOop;
> ```
>
> 1.8中klass继承链
>
> ```c++
> class Klass;
> class   InstanceKlass;
> class     InstanceMirrorKlass;
> class     InstanceClassLoaderKlass;
> class     InstanceRefKlass;
> class   ArrayKlass;
> class     ObjArrayKlass;
> class     TypeArrayKlass;
> ```
>
> 而少掉的那部分，其实只是换了个名字，叫做Metadata
>
> ```c++
> // The metadata hierarchy is separate from the oop hierarchy
> //      class MetaspaceObj
> class   ConstMethod;
> class   ConstantPoolCache;
> class   MethodData;
> //      class Metadata
> class   Method;
> class   ConstantPool;
> //      class CHeapObj
> class   CompiledICHolder;
> ```

### oopDesc之 _mark(也就是mark word)

以1.8中为例，oopDesc表示一个Java对象，其中有两个变量：

```c++
class oopDesc {
  friend class VMStructs;
  friend class JVMCIVMStructs;
 private:
  volatile markOop _mark;//线程状态 、并发锁 、GC 分代信息等内部标识，这些标识全都打在_mark变量上
  union _metadata {
    Klass*      _klass;
    narrowKlass _compressed_klass;
  } _metadata;

```

**_mark :存储对象运行时记录信息，如hashCode、GC分代年龄、锁状态标志，线程持有的锁、偏向线程id，偏向时间戳等。**markOop是什么呢，看一下oopHierarchy.hpp中的定义

```c++
// OBJECT hierarchy
// This hierarchy is a representation hierarchy, i.e. if A is a superclass
// of B, A's representation is a prefix of B's representation.

typedef juint narrowOop; // Offset instead of address for an oop within a java object

// If compressed klass pointers then use narrowKlass.
typedef juint  narrowKlass;

typedef void* OopOrNarrowOopStar;
typedef class   markOopDesc*                markOop; // 这里

#ifndef CHECK_UNHANDLED_OOPS

typedef class oopDesc*                            oop;
typedef class   instanceOopDesc*            instanceOop;
typedef class   arrayOopDesc*                    arrayOop;
typedef class     objArrayOopDesc*            objArrayOop;
typedef class     typeArrayOopDesc*            typeArrayOop;
```

看下 markOopDesc的注释（更直观一点）。。。

```java
// markOop描述一个对象的头。
//注意，这个mark不是一个真正的oop，而只是一个单词。
//由于历史原因，它被放在oop层次结构中。
//对象头的位格式(最重要的第一个，下面是大端布局):
//
// 32位:
//  --------------
//             hash:25 ------------>| age:4    biased_lock:1 lock:2 (normal object)
//             JavaThread*:23 epoch:2 age:4    biased_lock:1 lock:2 (biased object)
//             size:32 ------------------------------------------>| (CMS free block)
//             PromotedObject*:29 ---------->| promo_bits:3 ----->| (CMS promoted object)
//----------------
//
// 64位:
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
//---------------

```

![1619101646024](./images\1619101646024.png)

###  oopDesc 之 _metadata（元数据指针）

其中_klass指向描述类型的Klass对象的指针，Klass包含了实例对象所属类型的元数据，继承了Metadata，Metadata继承了MetaspaceObj，Klass 可以理解为一个 Java 类在hotspot内部的表示，再次贴出oopHierarchy.hpp中 metadata 和 Klass 的继承关系，可以看出 metadata、klass 层次结构与oop层次结构是分开的。：

```c++
// The metadata hierarchy is separate from the oop hierarchy

//      class MetaspaceObj
class   ConstMethod;
class   ConstantPoolCache;
class   MethodData;
//      class Metadata
class   Method;
class   ConstantPool;
//      class CHeapObj
class   CompiledICHolder;


// The klass hierarchy is separate from the oop hierarchy.

class Klass;
class   InstanceKlass;
class     InstanceMirrorKlass;
class     InstanceClassLoaderKlass;
class     InstanceRefKlass;
class   ArrayKlass;
class     ObjArrayKlass;
class     TypeArrayKlass;
```

贴上Klass的源码：

```c++
class Klass : public Metadata {
  friend class VMStructs;
  friend class JVMCIVMStructs;
 protected:

  enum { _primary_super_limit = 8 };
  jint        _layout_helper;

  // Klass identifier used to implement devirtualized oop closure dispatching.
  const KlassID _id;
  juint       _super_check_offset;

  // 类名.类名.  实例类:java/lang/String等数组类:[I， [Ljava/lang/String;，等等。对于所有其他类型的类设置为0。 
  // Class name.  Instance classes: java/lang/String, etc.  Array classes: [I,
  // [Ljava/lang/String;, etc.  Set to zero for all other kinds of classes.
  Symbol*     _name;

  // Cache of last observed secondary supertype
  Klass*      _secondary_super_cache;
  // Array of all secondary supertypes
  Array<Klass*>* _secondary_supers;
  // Ordered list of all primary supertypes
  Klass*      _primary_supers[_primary_super_limit];
    
  // java/lang/Class实例类镜像
  // java/lang/Class instance mirroring this class
  OopHandle _java_mirror;
  // 父类
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
  // Used when biased locking is both enabled and disabled for this type
  markOop  _prototype_header;
  jint     _biased_lock_revocation_count;

  // vtable length
  int _vtable_len;
```

再次引用[R大的一篇回答](https://www.zhihu.com/question/265677213/answer/297691281)，解释了其中的两个属性：

> 这个Klass 内部对象是HotSpot VM内部用来记录类的元数据的内部对象，并不对Java程序直接暴露出来。在这个上下文里，它里面有两个有趣的东西：一个是 _java_mirror 字段，指向该类对应的 java.lang.Class 对象；另一个是 prototype_header ，记录着该类的所有实例都会拥有的某些初始状态。例如说biased locking就会利用prototype header机制来控制某个Java类是否要参与biased locking——把prototype header设置为不带有biased pattern的值的情况下，这个类就不参与biased locking。
>
> 由 Klass::_java_mirror 字段指向的 java.lang.Class 对象就是暴露给Java程序的对应物。HotSpot VM并不通过它来记录类的元数据，只是通过它暴露一些Java / JVM规范说要提供给Java程序的反射信息。**Java的静态方法锁的就是这个对象。**

那 _metadata 中的 narrowKlass _compressed_klass表示什么呢？

```c++
// If compressed klass pointers then use narrowKlass. 
typedef juint  narrowKlass;
```

如果开启了压缩指针，则使用 narrowKlass。

### oopDesc 1.7和1.8的区别

OpenJDK 7：

```cpp
class oopDesc {
 private:
  volatile markOop  _mark;
  union _metadata {
    wideKlassOop    _klass;
    narrowOop       _compressed_klass;
  } _metadata;
};
```

关于OpenJDK 8的oopDesc跟OpenJDK 7的不同，[参考R大的回答](<https://www.zhihu.com/question/37697865/answer/73563964>)，总结一下R大的回答：

> oopDesc中的_mark字段其实在OpenJDK 7与8之间并没有显著的变化,
>
> ```c++
> class markOopDesc: public oopDesc {
>  private:
>   // Conversion
>   uintptr_t value() const { return (uintptr_t) this; } //其中 uintptr_t 是 unsigned int 类型，就不贴上了= = 
> ```
>
> 引用[R大的原话](https://www.zhihu.com/question/37697865/answer/73563964)：
>
> > 只要正确理解了它，就能明白我上面那段话是什么意思。举例来说，
> >
> > 看markOopDesc::has_bias_pattern()：
> >
> > ```cpp
> >   bool has_bias_pattern() const {
> >     return (mask_bits(value(), biased_lock_mask_in_place) == biased_lock_pattern);
> >   }
> > ```
> >
> > 这是一个成员函数，但它并没有使用this去访问任何“markOopDesc对象”的字段，而是调用value()函数将this转换成一个指针宽度的整数（uintptr_t），然后对这个整数做位操作，仅此而已。
>
>
>
> 主要的不同并不是在mark word上，而是在metadata部分：以前如klassOopDesc、methodOopDesc等元数据，虽然并非Java对象，但也由GC直接管理；到OpenJDK 8时，这些metadata都被移到GC掌管之外的native memory，于是oopDesc这个对象头里metadata的指针就不是一个oop，而是native直接指针（或者压缩的native直接指针了）。
>
> ```cpp
> typedef class klassOopDesc* wideKlassOop;
> typedef class klassOopDesc* klassOop;
> ```

### Klass 

Klass定义了所有Klass类型共享的结构和行为：描述自身的布局和与其他类间的关系（父类、子类、兄弟等等），再次贴出Klass的源码：

```c++
class Klass : public Metadata {
  friend class VMStructs;
  friend class JVMCIVMStructs;
 protected:

  enum { _primary_super_limit = 8 };
  jint        _layout_helper;

  // Klass identifier used to implement devirtualized oop closure dispatching.
  const KlassID _id;
  juint       _super_check_offset;

  // 类名.类名.java/lang/String   [Ljava/lang/String 等
  // Class name.  Instance classes: java/lang/String, etc.  Array classes: [I,
  // [Ljava/lang/String;, etc.  Set to zero for all other kinds of classes.
  Symbol*     _name;

  // Cache of last observed secondary supertype
  Klass*      _secondary_super_cache;
  // Array of all secondary supertypes
  Array<Klass*>* _secondary_supers;
  // Ordered list of all primary supertypes
  Klass*      _primary_supers[_primary_super_limit];
  // java/lang/Class实例类镜像
  OopHandle _java_mirror;
  // 父类
  Klass*      _super;
  // 第一个子类 (NULL if none); _subklass->next_sibling() 可以找到下一个
  Klass*      _subklass;
  // 下一个兄弟节点（拥有共同父类的Klass）
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
  // Used when biased locking is both enabled and disabled for this type
  markOop  _prototype_header;
  jint     _biased_lock_revocation_count;

  // vtable length
  int _vtable_len;
```

> Java的类继承层次结构是多叉树——每个Java类有且只有一个父类（根节点的java.lang.Object除外，它没有父类），并且可以有零到任意多个子类。为方便存储这种多叉树，HotSpot将其转换为二叉树来表示，其左子树由_subklass记录，代表继承树上更深一层的类；右子树由_next_sibiling记录，代表继承树上同一层、父类相同的类。
>
> [---来自R大的读书笔记](https://book.douban.com/annotation/31468146/)

JDK8之后，GC堆里的对象的对象头的_klass直接就是Klass*（或者压缩指针版），直接指向在Metaspace内的Klass（不再经过klassOopDesc这个包装层），然后Klass与Class仍然相互持有对方的指针。

![1619103901859](./images\1619103901859.png)

虽然上面的图不是1.8的，但是思想是差不多的，_klass直接指向元空间的Klass。

### 参考文章（我都是跪着读完的  >_<）

<https://time.geekbang.org/column/article/101244>

<https://www.cnblogs.com/webor2006/p/11442551.html>

<https://blog.csdn.net/bohu83/article/details/51141836>

<https://www.cnblogs.com/tiancai/p/9382542.html>

<https://www.zhihu.com/question/265677213/answer/297691281>

<https://www.javadoop.com/post/metaspace>

https://www.zhihu.com/question/265677213/answer/297691281

<https://book.douban.com/people/RednaxelaFX/annotation/25847620/>

<https://book.douban.com/annotation/31468146/>