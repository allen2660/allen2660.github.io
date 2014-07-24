---
layout: post
title:  protobuf中的反射解析
---


本篇讲述了Reflection以及GeneratedMessageReflection的实现。

reflection的那些GetXXX，方法，都是怎么实现的，可以从Message中根据Descriptor取出东西？


## GeneratedMessageReflection 实现

### 构造函数&fields

    GeneratedMessageReflection(const Descriptor* descriptor,
                             const Message* default_instance,
                             const int offsets[],
                             int has_bits_offset,
                             int unknown_fields_offset,
                             int extensions_offset,
                             const DescriptorPool* pool,
                             MessageFactory* factory,
                             int object_size);
主要fields：

+ descriptor_:Descriptor*
+ default_instance_:Message* 
+ offsets_ int*
+ has_bits_offset_:int
+ unknown_fields_offset_:int
+ object_size_:int
+ descriptor_pool_:DescriptorPool*
+ message_factory_:MessageFactory*

看一下一个典型的函数GetString：

    string GeneratedMessageReflection::GetString(
        const Message& message, const FieldDescriptor* field) const {
      USAGE_CHECK_ALL(GetString, SINGULAR, STRING);
      if (field->is_extension()) {
        return GetExtensionSet(message).GetString(field->number(),
                                                  field->default_value_string());
      } else {
        switch (field->options().ctype()) {
          default:  // TODO(kenton):  Support other string reps.
          case FieldOptions::STRING:
            //字符串是以string*的方式存储的。
            return *GetField<const string*>(message, field);
        }   

        GOOGLE_LOG(FATAL) << "Can't get here.";
        return kEmptyString;  // Make compiler happy.
      }
    }

这个类有一些核心的辅助函数

    // These simple template accessors obtain pointers (or references) to
    // the given field.
    template <typename Type>
    inline const Type& GeneratedMessageReflection::GetRaw(
        const Message& message, const FieldDescriptor* field) const {
      const void* ptr = reinterpret_cast<const uint8*>(&message) +
                        offsets_[field->index()];
      return *reinterpret_cast<const Type*>(ptr);
    }   

    template <typename Type>
    inline Type* GeneratedMessageReflection::MutableRaw(
        Message* message, const FieldDescriptor* field) const {
      void* ptr = reinterpret_cast<uint8*>(message) + offsets_[field->index()];
      return reinterpret_cast<Type*>(ptr);
    }   

    template <typename Type>
    inline const Type& GeneratedMessageReflection::DefaultRaw(
        const FieldDescriptor* field) const {
      const void* ptr = reinterpret_cast<const uint8*>(default_instance_) +
                        offsets_[field->index()];
      return *reinterpret_cast<const Type*>(ptr);
    }   

    inline const uint32* GeneratedMessageReflection::GetHasBits(
        const Message& message) const {
      const void* ptr = reinterpret_cast<const uint8*>(&message) + has_bits_offset_;
      return reinterpret_cast<const uint32*>(ptr);
    }
    inline uint32* GeneratedMessageReflection::MutableHasBits(
        Message* message) const {
      void* ptr = reinterpret_cast<uint8*>(message) + has_bits_offset_;
      return reinterpret_cast<uint32*>(ptr);
    }   

    inline const ExtensionSet& GeneratedMessageReflection::GetExtensionSet(
        const Message& message) const {
      GOOGLE_DCHECK_NE(extensions_offset_, -1);
      const void* ptr = reinterpret_cast<const uint8*>(&message) +
                        extensions_offset_;
      return *reinterpret_cast<const ExtensionSet*>(ptr);
    }
    inline ExtensionSet* GeneratedMessageReflection::MutableExtensionSet(
        Message* message) const {
      GOOGLE_DCHECK_NE(extensions_offset_, -1);
      void* ptr = reinterpret_cast<uint8*>(message) + extensions_offset_;
      return reinterpret_cast<ExtensionSet*>(ptr);
    }   

    // Simple accessors for manipulating has_bits_.
    inline bool GeneratedMessageReflection::HasBit(
        const Message& message, const FieldDescriptor* field) const {
      return GetHasBits(message)[field->index() / 32] &
        (1 << (field->index() % 32));
    }   

    inline void GeneratedMessageReflection::SetBit(
        Message* message, const FieldDescriptor* field) const {
      MutableHasBits(message)[field->index() / 32] |= (1 << (field->index() % 32));
    }   

    inline void GeneratedMessageReflection::ClearBit(
        Message* message, const FieldDescriptor* field) const {
      MutableHasBits(message)[field->index() / 32] &= ~(1 << (field->index() % 32));
    }   

    // Template implementations of basic accessors.  Inline because each
    // template instance is only called from one location.  These are
    // used for all types except messages.
    template <typename Type>
    inline const Type& GeneratedMessageReflection::GetField(
        const Message& message, const FieldDescriptor* field) const {
      return GetRaw<Type>(message, field);
    }   

    template <typename Type>
    inline void GeneratedMessageReflection::SetField(
        Message* message, const FieldDescriptor* field, const Type& value) const {
      *MutableRaw<Type>(message, field) = value;
      SetBit(message, field);
    }   

    template <typename Type>
    inline Type* GeneratedMessageReflection::MutableField(
        Message* message, const FieldDescriptor* field) const {
      SetBit(message, field);
      return MutableRaw<Type>(message, field);
    }   

    template <typename Type>
    inline const Type& GeneratedMessageReflection::GetRepeatedField(
        const Message& message, const FieldDescriptor* field, int index) const {
      return GetRaw<RepeatedField<Type> >(message, field).Get(index);
    }   

    template <typename Type>
    inline const Type& GeneratedMessageReflection::GetRepeatedPtrField(
        const Message& message, const FieldDescriptor* field, int index) const {
      return GetRaw<RepeatedPtrField<Type> >(message, field).Get(index);
    }   

    template <typename Type>
    inline void GeneratedMessageReflection::SetRepeatedField(
        Message* message, const FieldDescriptor* field,
        int index, Type value) const {
      MutableRaw<RepeatedField<Type> >(message, field)->Set(index, value);
    }   

    template <typename Type>
    inline Type* GeneratedMessageReflection::MutableRepeatedField(
        Message* message, const FieldDescriptor* field, int index) const {
      RepeatedPtrField<Type>* repeated =
        MutableRaw<RepeatedPtrField<Type> >(message, field);
      return repeated->Mutable(index);
    }   

    template <typename Type>
    inline void GeneratedMessageReflection::AddField(
        Message* message, const FieldDescriptor* field, const Type& value) const {
      MutableRaw<RepeatedField<Type> >(message, field)->Add(value);
    }   

    template <typename Type>
    inline Type* GeneratedMessageReflection::AddField(
        Message* message, const FieldDescriptor* field) const {
      RepeatedPtrField<Type>* repeated =
        MutableRaw<RepeatedPtrField<Type> >(message, field);
      return repeated->Add();
    }

## GMR和动态生成的Message的关系

反射的Reflection离不开下面几个类：

+ Reflection
+ DynamicMessageFactory
+ Message
+ Descriptor

常用的使用场景是：

1. 创建一个DynamicMessageFactory d_factory
2. const Message * msg = d_factory->GetPrototype(descriptor);
3. return msg->New();

### DynamicMessageFactory::GetPrototype

1. 首先构建TypeInfo对象
2. 使用TypeInfo对象构建DynamicMessage对象 prototype
3. 构建prototype的reflection对象（其中default_intance 是prototype）
4. return prototype。



## GMR和静态Message的关系

我们来看一段addressbook.pb.h和addressbook.pb.cc的代码。

addressbook.pb.cc中有下面三个函数：

    // Internal implementation detail -- do not call these.
    void  protobuf_AddDesc_addressbook_2eproto();
    void protobuf_AssignDesc_addressbook_2eproto();
    void protobuf_ShutdownFile_addressbook_2eproto();

### AddDesc

    // Force AddDescriptors() to be called at static initialization time.
    struct StaticDescriptorInitializer_addressbook_2eproto {
      StaticDescriptorInitializer_addressbook_2eproto() {
        protobuf_AddDesc_addressbook_2eproto();
      }
    } static_descriptor_initializer_addressbook_2eproto_;

    void protobuf_AddDesc_addressbook_2eproto() {
      static bool already_here = false;
      if (already_here) return;
      already_here = true;
      GOOGLE_PROTOBUF_VERIFY_VERSION;   

      ::google::protobuf::DescriptorPool::InternalAddGeneratedFile(
        "\n\021addressbook.proto\022\010tutorial\"\'\n\010FullNam"
        "e\022\r\n\005first\030\001 \002(\t\022\014\n\004last\030\002 \001(\t\"\200\002\n\006Perso"
        "n\022\014\n\004name\030\001 \002(\t\022\n\n\002id\030\002 \002(\005\022\r\n\005email\030\003 \001"
        "(\t\022+\n\005phone\030\004 \003(\0132\034.tutorial.Person.Phon"
        "eNumber\022$\n\010fullname\030\005 \002(\0132\022.tutorial.Ful"
        "lName\032M\n\013PhoneNumber\022\016\n\006number\030\001 \002(\t\022.\n\004"
        "type\030\002 \001(\0162\032.tutorial.Person.PhoneType:\004"
        "HOME\"+\n\tPhoneType\022\n\n\006MOBILE\020\000\022\010\n\004HOME\020\001\022"
        "\010\n\004WORK\020\002\"/\n\013AddressBook\022 \n\006person\030\001 \003(\013"
        "2\020.tutorial.PersonB)\n\024com.example.tutori"
        "alB\021AddressBookProtos", 421);
      ::google::protobuf::MessageFactory::InternalRegisterGeneratedFile(
        "addressbook.proto", &protobuf_RegisterTypes);
      FullName::default_instance_ = new FullName();
      Person::default_instance_ = new Person();
      Person_PhoneNumber::default_instance_ = new Person_PhoneNumber();
      AddressBook::default_instance_ = new AddressBook();
      FullName::default_instance_->InitAsDefaultInstance();
      Person::default_instance_->InitAsDefaultInstance();
      Person_PhoneNumber::default_instance_->InitAsDefaultInstance();
      AddressBook::default_instance_->InitAsDefaultInstance();
      ::google::protobuf::internal::OnShutdown(&protobuf_ShutdownFile_addressbook_2eproto);
    }

### AssignDesc
这个操作只执行一遍。构造default_instance,offsets等用于构造反射对象。

    GOOGLE_PROTOBUF_DECLARE_ONCE(protobuf_AssignDescriptors_once_);
    inline void protobuf_AssignDescriptorsOnce() {
      ::google::protobuf::GoogleOnceInit(&protobuf_AssignDescriptors_once_,
                     &protobuf_AssignDesc_addressbook_2eproto);
    }

    void protobuf_AssignDesc_addressbook_2eproto() {
      protobuf_AddDesc_addressbook_2eproto();
      const ::google::protobuf::FileDescriptor* file =
        ::google::protobuf::DescriptorPool::generated_pool()->FindFileByName(
          "addressbook.proto");
      GOOGLE_CHECK(file != NULL);
      FullName_descriptor_ = file->message_type(0);
      static const int FullName_offsets_[2] = {
        GOOGLE_PROTOBUF_GENERATED_MESSAGE_FIELD_OFFSET(FullName, first_),
        GOOGLE_PROTOBUF_GENERATED_MESSAGE_FIELD_OFFSET(FullName, last_),
      };
      FullName_reflection_ =
        new ::google::protobuf::internal::GeneratedMessageReflection(
          FullName_descriptor_,
          FullName::default_instance_,
          FullName_offsets_,
          GOOGLE_PROTOBUF_GENERATED_MESSAGE_FIELD_OFFSET(FullName, _has_bits_[0]),
          GOOGLE_PROTOBUF_GENERATED_MESSAGE_FIELD_OFFSET(FullName, _unknown_fields_),
          -1,
          ::google::protobuf::DescriptorPool::generated_pool(),
          ::google::protobuf::MessageFactory::generated_factory(),
          sizeof(FullName));
      Person_descriptor_ = file->message_type(1);
      static const int Person_offsets_[5] = {
        GOOGLE_PROTOBUF_GENERATED_MESSAGE_FIELD_OFFSET(Person, name_),
        GOOGLE_PROTOBUF_GENERATED_MESSAGE_FIELD_OFFSET(Person, id_),
        GOOGLE_PROTOBUF_GENERATED_MESSAGE_FIELD_OFFSET(Person, email_),
        GOOGLE_PROTOBUF_GENERATED_MESSAGE_FIELD_OFFSET(Person, phone_),
        GOOGLE_PROTOBUF_GENERATED_MESSAGE_FIELD_OFFSET(Person, fullname_),
      };
      Person_reflection_ =
        new ::google::protobuf::internal::GeneratedMessageReflection(
          Person_descriptor_,
          Person::default_instance_,
          Person_offsets_,
          GOOGLE_PROTOBUF_GENERATED_MESSAGE_FIELD_OFFSET(Person, _has_bits_[0]),
          GOOGLE_PROTOBUF_GENERATED_MESSAGE_FIELD_OFFSET(Person, _unknown_fields_),
          -1,
          ::google::protobuf::DescriptorPool::generated_pool(),
          ::google::protobuf::MessageFactory::generated_factory(),
          sizeof(Person));
      Person_PhoneNumber_descriptor_ = Person_descriptor_->nested_type(0);
      static const int Person_PhoneNumber_offsets_[2] = {
        GOOGLE_PROTOBUF_GENERATED_MESSAGE_FIELD_OFFSET(Person_PhoneNumber, number_),
        GOOGLE_PROTOBUF_GENERATED_MESSAGE_FIELD_OFFSET(Person_PhoneNumber, type_),
      };
      Person_PhoneNumber_reflection_ =
        new ::google::protobuf::internal::GeneratedMessageReflection(
          Person_PhoneNumber_descriptor_,
          Person_PhoneNumber::default_instance_,
          Person_PhoneNumber_offsets_,
          GOOGLE_PROTOBUF_GENERATED_MESSAGE_FIELD_OFFSET(Person_PhoneNumber, _has_bits_[0]),
          GOOGLE_PROTOBUF_GENERATED_MESSAGE_FIELD_OFFSET(Person_PhoneNumber, _unknown_fields_),
          -1,
          ::google::protobuf::DescriptorPool::generated_pool(),
          ::google::protobuf::MessageFactory::generated_factory(),
          sizeof(Person_PhoneNumber));
      Person_PhoneType_descriptor_ = Person_descriptor_->enum_type(0);
      AddressBook_descriptor_ = file->message_type(2);
      static const int AddressBook_offsets_[1] = {
        GOOGLE_PROTOBUF_GENERATED_MESSAGE_FIELD_OFFSET(AddressBook, person_),
      };
        AddressBook_reflection_ =
        new ::google::protobuf::internal::GeneratedMessageReflection(
          AddressBook_descriptor_,
          AddressBook::default_instance_,
          AddressBook_offsets_,
          GOOGLE_PROTOBUF_GENERATED_MESSAGE_FIELD_OFFSET(AddressBook, _has_bits_[0]),
          GOOGLE_PROTOBUF_GENERATED_MESSAGE_FIELD_OFFSET(AddressBook, _unknown_fields_),
          -1,
          ::google::protobuf::DescriptorPool::generated_pool(),
          ::google::protobuf::MessageFactory::generated_factory(),
          sizeof(AddressBook));
    }

### ShutdownFile

这个函数被注册到google::protobuf::internal::OnShutdown(); 在AddDesc中注册的。

    void protobuf_ShutdownFile_addressbook_2eproto() {
      delete FullName::default_instance_;
      delete FullName_reflection_;
      delete Person::default_instance_;
      delete Person_reflection_;
      delete Person_PhoneNumber::default_instance_;
      delete Person_PhoneNumber_reflection_;
      delete AddressBook::default_instance_;
      delete AddressBook_reflection_;
    }

## 静态代码的序列化/反序列化

静态代码继承自Message，自然有一系列来自父类的Parse/Serialize方法。

不过静态类有且仅有一个自己的Serailize方法：

    void Person::SerializeWithCachedSizes(::google::protobuf::io::CodedOutputStream* output) const;
    
    ::google::protobuf::uint8* Person::SerializeWithCachedSizesToArray(::google::protobuf::uint8* target) const;

而其他的方法，都是在父类中，使用反射实现的，参见[pb的codec](/protobuf-codec/)