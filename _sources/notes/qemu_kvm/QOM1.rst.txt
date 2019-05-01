QEMU Object Model
=================

- intro

  - OOP implemented in C library.
  - use C struct with function pointer
  - inheritance: single inheritance + interface

- 特點

  - QOM has 2 type of class member: C struct member and Property member
  - split Class and Object to 2 different C struct.
  
    - There is only one Class for each type, but each class instance has one different Object.
    - class virtual method is stored in Class.
    - per instance data are stored in Object.

  - type is defined by ``TypeInfo`` and ``type_register()``

為了更詳細的講解 QOM, 我們需要知道 QOM 由 4 個 components 所組成.

  - Type: define Class and Object
  - Class: store static data & virtual method pointers
  - Object: store dynamic data
  - Property: another type of Class/Object member, which supports getter/setter.

Basic Example
-------------

example code part1

- Type is defined by ``TypeInfo`` and ``type_register()``
- define Class init function(``.class_init``) and Object constructor(``.instance_init``) in ``TypeInfo``

::

    // Example Code of class Foo
    // Type: TypeInfo foo_info and (internal) TypeImpl
    // Class: FooClass
    // Object: FooState

    #define TYPE_FOO "foo"

    type_init(foo_register_type)
    void foo_register_type(void){
        type_register_static(&foo_info);
    }
    
    const TypeInfo foo_info = {
        .name = TYPE_FOO                    // typename
        .parent = TYPE_OBJECT               // parent type (base class)
        .class_init = foo_class_init,
        .class_size = sizeof(FooClass),     // Class type size
        .instance_init = foo_instance_init, // constructor
        .instance_size = sizeof(FooState)   // Object type size
    }
    
example code part2(not finish)

- define Class and Object
- write Class init function
- write Object constructor

::

    struct FooClass {};
    void foo_class_init(ObjectClass* klass, void* data){
    }

    struct FooState {};
    foo_instance_init(Object* obj){
    }

example code part3(not finish)

- use Class/Object.

::

    Object* obj = object_new(TYPE_FOO);
    object_delete(obj);

    // get Object by type conversion
    FooState foo_s = OBJECT_CHECK(FooState, obj, TYPE_FOO);

    // Object can reference to Class: obj->class
    FooClass foo_c = OBJECT_GET_CLASS(FooClass, obj, TYPE_FOO);

Q&A1
----

- Class v.s. Object:

  每個 Type 只有一個 Class, 但每個 class instance 就會產生一個獨立的 Object. 
  所以很有可能放在 Class member 代表所有 instance 共用的資料, 比如說 member function 或 C++ class static member.
  每個 instance 獨立的資料, 則是放在 Object 的 member 中.

- What is Property?

  目前看起來 Property 也是一種 class member, 它透過 hash table 在做出一組跟原本獨立的 class members 集合.
  Property class member 比起一般 member, 優點是有支援 setter/getter. 應該還有其他差異, 留帶觀察.

QOM Internal
------------

how to define a type?

- ``type_init(): module_init(QOM)``
- ``type_register_static(TypeInfo* info)``:
  用 TypeInfo 建立一個 TypeImpl, 並且把該 TypeImpl 放到 type_table 裏面.::
    
    - type_new(): 用 TypeInfo 建立 TypeImpl
    - type_table_add/lookup(): hash table - (key, value) = (typename(char *), TypeImpl)

- ``type_initialize(TypeImpl *ti)``: 初始化 TypeImpl 以及 Class, 並呼叫 TypeInfo 時定義的 ``class_init()``.

  - 第一次 ``object_new()`` 時才會被呼叫到 (lazy evaluation).
  - allocate memory for Class: ``ti->class = g_malloc0(ti->class_size);``
  - 初始化 Class 的 property 為 hash table: ``ti->class->properties = g_hash_table_new_full( ... );``
  - parent 相關

    - recursive ``type_initialize()`` parent
    - 疑似繼承的行為, 直接 memcpy parent 的 class: ``memcpy(ti->class, parent->class, parent->class_size);``
    - 初始化 ti 跟 parent 的所有 interfaces: type_initialize_interface()

  - initialize from TypeInfo: class_init() and class_base_init()

    - recursive initialize parente class: ``parent->class_base_init(ti->class, ti->class_data);``
    - Class initialize: ``ti->class_init()``

- ``object_new(char* typename)``: 初始化 Object::

    - type_initialize() at first (notice: only execute once at first call)
    - object_initialize_with_type(obj, type->instance_size, type): 初始化 Object

      - 初始化 property 的 hash table: obj->properties = g_hash_table_new_full( ... )
      - obj->class points to TypeImpl's class(TypeImpl type; type->class;)
      - use TypeInfo: ti->instance_init(obj), ti->instance_post_init(obj);

More

- type conversion MACRO::

    1. OBJECT_CHECK(type, obj, name): (type*)(Object*) obj;
    2. OBJECT_CLASS_CHECK(class_type, class, name): (class_type*)(ObjectClass*) class;
    3. OBJECT_GET_CLASS(class, obj, name): OBJECT_CLASS_CHECK(class, obj->class, name)

Property
~~~~~~~~

API and internals::
    
    Object* object_new_with_props(): object_new() + object_set_propv() + object_property_add_child()

        object_property_set(): set value by setter of ObjectProperty

        ObjectProperty *object_property_find(Object *obj, const char *name, Error **errp)
            find property from obj: // Class = object_get_class(obj);
               1. find parent Class property
               2. find Class property
               3. find Object property


- other features

  - inheritance: single inheritance + interface-based multiple inheritance (see comments in ``include/qom/object.h``)
  - no operator overloading
  - all methods(function pointer) are virtual method

- Visitor: 

  - include/qapi/visitor-impl.h: struct Visitor is a set of function pointers (start_struct, start_list ...)
  - qapi/qapi-visit-core.c
  - qapi/string-input-visitor.c: struct StringInputVisitor, string_input_visitor_new(), 

QDev
----


QDev Inheritance (Tree View) (not finish)
QDev Inheritance (Table View)

basic:

  ===================== ==================== ================== ===============
  Type                  Parent               Class              Instance
  ===================== ==================== ================== ===============
  TYPE_DEVICE           TYPE_OBJECT          X                  DeviceState
  TYPE_SYS_BUS_DEVICE   TYPE_DEVICE          SysBusDeviceClass  SysBusDevice
  --------------------- -------------------- ------------------ ---------------
  TYPE_BUS              TYPE_OBJECT          BusClass           BusState
  TYPE_SYSTEM_BUS       TYPE_BUS             X                  BusState
  ===================== ==================== ================== ===============

PCI device:

  ===================== ==================== ================== ===============
  Type                  Parent               Class              Instance
  ===================== ==================== ================== ===============
  TYPE_PCI_DEVICE       TYPE_DEVICE          PCIDeviceClass     PCIDevice,
  TYPE_PCI_BUS          TYPE_BUS             PCIBusClass        PCIBus,
  TYPE_PCI_HOST_BRIDGE  TYPE_SYS_BUS_DEVICE  PCIHostBridgeClass PCIHostState,
  --------------------- -------------------- ------------------ ---------------
  TYPE_PCIE_BUS         TYPE_PCI_BUS         X                  X
  TYPE_PCIE_HOST_BRIDGE TYPE_PCI_HOST_BRIDGE X                  PCIExpressHost
  ===================== ==================== ================== ===============

.. _qdev_property:

device parameters
~~~~~~~~~~~~~~~~~
(not finish)

CLI options to DeviceClass->props (Property type)

::

    // CLI options: -device ivshmem,shm=hello,size=1

    // define ivshmem device
    #define TYPE_IVSHMEM "ivshmem"
    static const TypeInfo ivshmem_info = {                                                                                       .name          = TYPE_IVSHMEM,
    }

    static Property ivshmem_properties[] = {
        DEFINE_PROP_STRING("size", IVShmemState, sizearg),
        DEFINE_PROP_STRING("shm", IVShmemState, shmobj),
        ...
        DEFINE_PROP_END_OF_LIST(),
    };

    // Interals

       // CLI options: -device ivshmem,shm=hello,size=1
       // QemuOpts: driver=ivshmem,shm=hello,size=1
       // QEMU Object Property
           qdev_device_add(QemuOpts *opts, Error **errp) 
           => qemu_opt_foreach() 
           => set_property(&dev, opt->name, opt->str, &err);
           => object_property_parse(obj, opt->str, opt->name);

       // QDev Property (DeviceClass->props)
           device_initfn()
           => qdev_property_add_static(): Add a static QOM property to @dev for qdev property @prop.

DeviceClass->props (Property type) to ``IVShmemState->{shmobj, sizearg}``
::

    static Property ivshmem_properties[] = {
        DEFINE_PROP_STRING("size", IVShmemState, sizearg),
        DEFINE_PROP_STRING("shm", IVShmemState, shmobj),
        ...
        DEFINE_PROP_END_OF_LIST(),
    };

Resources
---------

source code
~~~~~~~~~~~

- ./include/qom/object.h 
- ./qom/object.c

reference
~~~~~~~~~

- `[2012] QOM Vadis? Taking Objects To The CPU And Beyond <https://www.linux-kvm.org/images/f/f6/2012-forum-QOM_CPU.pdf>`_
- `QOM Device Creation and Activation <http://nairobi-embedded.org/035_qom_and_device_creation.html>`_
- `Qemu 中的设备注册 <http://ytliu.info/blog/2015/01/10/qemushe-bei-chu-shi-hua/>`_
- `[2013] Modern QEMU Devices A Hands-On Approach <http://www.linux-kvm.org/images/0/0b/Kvm-forum-2013-Modern-QEMU-devices.pdf>`_
- `[2014] QOM exegesis and apocalypse <https://www.linux-kvm.org/images/9/90/Kvmforum14-qom.pdf>`_
