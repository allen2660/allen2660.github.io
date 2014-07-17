---
layout: default
title:  protobuf的动态编译
---

# 动态编译

## 使用场景

通常情况下，使用Protobuf的人会首先写好.proto文件，再使用protoc编译器编译成想要的目标语言文件。然后将这些自动生成的代码和应用程序一起编译。

在某些场景下，人们无法或者说不想预先知道.proto文件（比如Hive的Protobuf Serde）。这就需要动态编译.proto文件。

### C++

通过前面分析protobuf源码知道，Protobuf提供了compiler包来编译.proto文件，protoc就是这么用的。

#### 创建Message

    const Descriptor * GetDescriptorFromProtoByTypeName (const string & input_file, const string &type_name) {
        DiskSourceTree *source_tree = new DiskSourceTree;
        ErrorPrinter *error_collector = new ErrorPrinter(ERROR_FORMAT_GCC, source_tree);
        Importer *importer = new Importer(source_tree, error_collector);    

        source_tree->MapPath("","./");  

        //下面这两行，至少需要执行一行，不然DescriptorPool找不到Message。DescriptorPool会首先去Tables的symbols_找，找不到会去database中找Symbol
        //这个和SourceTreeDescriptorDatabase::FindFileContainingSymbol 直接返回false有关系。
        //所以需要先FindFileByName或者Import将.proto文件 load到内存中
        //const FileDescriptor *fd = importer.Import(input_file);
        const FileDescriptor *fd = importer->pool()->FindFileByName(input_file);    

        const Descriptor *desc = importer->pool()->FindMessageTypeByName(type_name);
        cout << desc->DebugString();
        return desc;
    }

	Message *createMessage(const Descriptor *descriptor) {
        Message *message = NULL;
        //再一次踩坑，这个factory中也有一个scoped_ptr
        //一开始是使用的栈上的，总是core
        DynamicMessageFactory *factory = new DynamicMessageFactory;
        if (descriptor) {
            // 创建default message(prototype)
            const Message *prototype = factory->GetPrototype(descriptor);
            if (NULL != prototype) {
                // 创建一个可修改的message
                message = prototype->New();
            }   
        }       
        return message;
    }

#### 使用反射设置Message

    /**
    * @brief generateMessage 
     * 实际类型是tutorial.AddressBook, 但是我们没有类型的定义（动态Parse的，不是静态生成）
     * 这里使用反射接口来设置Message的Field，最后打出Message
     *
     * @param descriptor
     */
    void generateMessage(const Descriptor *descriptor, ostream *output) {
        Message *message = createMessage(descriptor);
        if(message == NULL) {
            cout<< "message is NULL" << endl;
        }
        const Reflection *reflection = NULL;
        reflection = message->GetReflection();  

        const FieldDescriptor * field = NULL;
        field = descriptor->FindFieldByName("name");
        reflection->SetString(message, field, "allen"); 

        const FieldDescriptor * field_id = NULL;
        field_id = descriptor->FindFieldByName("id");
        reflection->SetInt32(message, field_id, 12345); 

        cout << message->DebugString();
        message->SerializeToOstream(output);    

        //delete message;   

    }


#### 使用反射解析Message

当用动态接口得到FileDescriptor，以及Descriptor后，要想解析一个二进制pbMsg，可以这么做：
    
    void parseMessage(const Descriptor *descriptor, istream *in_stream) {
        Message *message = createMessage(descriptor);   

        message->ParseFromIstream(in_stream);
        const Reflection *reflection = message->GetReflection();    

        const FieldDescriptor * field = NULL;
        field = descriptor->FindFieldByName("name");
        string name = reflection->GetString(*message, field);   

        const FieldDescriptor * field_id = NULL;
        field_id = descriptor->FindFieldByName("id");
        int id = reflection->GetInt32(*message, field_id);  

        cout << message->DebugString(); 

        delete message; 

    }

### Java

Java目前没有找到直接编译.proto文件的方法，[这篇文章](http://blog.csdn.net/lufeng20/article/details/8736584)中讲解可以使用desc文件来build，但是生成desc文件还是要使用protoc二进制。这个待研究。

## Demo

上面的完整代码如下：

    /**
    * @file dynamic_demo.cc
     * @brief 
     * @author liwei
     * @version 
     * @date 2014-07-14
     */ 
    

    #include <iostream>
    #include <fstream>
    #include <string>
    #include <google/protobuf/compiler/importer.h>
    #include <google/protobuf/compiler/command_line_interface.h>
    #include <google/protobuf/descriptor.h>
    #include <google/protobuf/dynamic_message.h>    

    using namespace std;
    using namespace google::protobuf::compiler;
    using namespace google::protobuf::io;
    using namespace google::protobuf;   
    

    enum ErrorFormat {
        ERROR_FORMAT_GCC,   // GCC error output format (default).
        ERROR_FORMAT_MSVS   // Visual Studio output (--error_format=msvs).
      };    

    class ErrorPrinter : public MultiFileErrorCollector,
                                               public ErrorCollector {
     public:
      ErrorPrinter(ErrorFormat format, DiskSourceTree *tree = NULL)
        : format_(format), tree_(tree) {}
      ~ErrorPrinter() {}    

      // implements MultiFileErrorCollector ------------------------------
      void AddError(const string& filename, int line, int column,
                    const string& message) {    

        // Print full path when running under MSVS
        string dfile;
        if (format_ == ERROR_FORMAT_MSVS &&
            tree_ != NULL &&
            tree_->VirtualFileToDiskFile(filename, &dfile)) {
          cerr << dfile;
        } else {
          cerr << filename;
        }   

        // Users typically expect 1-based line/column numbers, so we add 1
        // to each here.
        if (line != -1) {
          // Allow for both GCC- and Visual-Studio-compatible output.
          switch (format_) {
            case ERROR_FORMAT_GCC:
              cerr << ":" << (line + 1) << ":" << (column + 1);
              break;
            case ERROR_FORMAT_MSVS:
              cerr << "(" << (line + 1) << ") : error in column=" << (column + 1);
              break;
          }
        }   

        cerr << ": " << message << endl;
      } 

      // implements io::ErrorCollector -----------------------------------
      void AddError(int line, int column, const string& message) {
        AddError("input", line, column, message);
      } 

     private:
      const ErrorFormat format_;
      DiskSourceTree *tree_;    

    };  

    const Descriptor * GetDescriptorFromProtoByTypeName (const string & input_file, const string &type_name) {
        DiskSourceTree *source_tree = new DiskSourceTree;
        ErrorPrinter *error_collector = new ErrorPrinter(ERROR_FORMAT_GCC, source_tree);
        Importer *importer = new Importer(source_tree, error_collector);    

        source_tree->MapPath("","./");  

        //下面这两行，至少需要执行一行，不然DescriptorPool找不到Message。DescriptorPool会首先去Tables的symbols_找，找不到会去database中找Symbol
        //这个和SourceTreeDescriptorDatabase::FindFileContainingSymbol 直接返回false有关系。
        //所以需要先FindFileByName或者Import将.proto文件 load到内存中
        //const FileDescriptor *fd = importer.Import(input_file);
        const FileDescriptor *fd = importer->pool()->FindFileByName(input_file);    

        const Descriptor *desc = importer->pool()->FindMessageTypeByName(type_name); 
        cout << desc->DebugString();
        return desc;
    }   
    

    Message *createMessage(const Descriptor *descriptor) {
        Message *message = NULL;
        //再一次踩坑，这个factory中也有一个scoped_ptr
        //一开始是使用的栈上的，总是core
        DynamicMessageFactory *factory = new DynamicMessageFactory;
        if (descriptor) {
            // 创建default message(prototype)
            const Message *prototype = factory->GetPrototype(descriptor);
            if (NULL != prototype) {
                // 创建一个可修改的message
                message = prototype->New();
            }
        }   

        return message;
    }   

    /**
     * @brief generateMessage 
     * 实际类型是tutorial.AddressBook, 但是我们没有类型的定义（动态Parse的，不是静态生成）
     * 这里使用反射接口来设置Message的Field，最后打出Message
     *
     * @param descriptor
     */
    void generateMessage(const Descriptor *descriptor, ostream *output) {
        Message *message = createMessage(descriptor);
        if(message == NULL) {
            cout<< "message is NULL" << endl;
        }
        const Reflection *reflection = NULL;
        reflection = message->GetReflection();  

        const FieldDescriptor * field = NULL;
        field = descriptor->FindFieldByName("name");
        reflection->SetString(message, field, "allen"); 

        const FieldDescriptor * field_id = NULL;
        field_id = descriptor->FindFieldByName("id");
        reflection->SetInt32(message, field_id, 12345); 

        cout << message->DebugString();
        message->SerializeToOstream(output);    

        //delete message;   

    }   

    void parseMessage(const Descriptor *descriptor, istream *in_stream) {
        Message *message = createMessage(descriptor);   

        message->ParseFromIstream(in_stream);
        const Reflection *reflection = message->GetReflection();    

        const FieldDescriptor * field = NULL;
        field = descriptor->FindFieldByName("name");
        string name = reflection->GetString(*message, field);   

        const FieldDescriptor * field_id = NULL;
        field_id = descriptor->FindFieldByName("id");
        int id = reflection->GetInt32(*message, field_id);  

        cout << message->DebugString(); 

        delete message; 
    
    

    }   

    int main(int argc, char * args[]) {
        //buildFromProtoByImporter("addressbook.proto");
        //buildFromProto("tutorial.AddressBook");
        
        const Descriptor *descriptor = GetDescriptorFromProtoByTypeName("addressbook.proto", "tutorial.Person");
        cout << "-----------Generate use reflection" <<endl;
        ofstream output;
        output.open("file.pb");
        generateMessage(descriptor, &output);
        cout << "-----------Parseing use reflection" <<endl;
        output.flush();
        output.close();
        ifstream input;
        input.open("file.pb");
        //cerr<< input <<endl;
        parseMessage(descriptor, &input);   

    }




# 反射

这一段主要包含下面几个类：

+ Message
+ Descriptor
+ DynamicMessage
+ DynamicMessageFactory
+ Reflection

## Message

Message位于 google/protobuf/message.h。Message继承自MessageLite类。MessageLite已经可以处理protobuf的绝大部分日常使用了，Message在其基础上加上了descriptor和reflection的功能，也就是动态类型特征。可以使用在.proto文件中加上选项使compiler生成的代码是Lite的：
    
    option optimize_for = LITE_RUNTIME;

Message的主要方法：

    struct Metadata {
        const Descriptor* descriptor;
        const Reflection* reflection;
    };

    //返回一个新的实例，内存将由调用者管理
    virtual Message* New() const = 0;

    //得到Reflection指针
    virtual const Reflection* GetReflection() const {
        return GetMetadata().reflection;
    }

    //一般静态生成的代码中都会有这个方法的实现，将Descriptor和Reflection通过protobuf_AssignDescriptorsOnce();方法来生成xxx_descriptor_和xxx_reflection_，并将其赋给metadata,返回
    virtual Metadata GetMetadata() const  = 0;

在静态代码中生成Reflection：(以addressbook.proto生成的代码举例)

    namespace {
    const ::google::protobuf::Descriptor* Person_descriptor_ = NULL;
    const ::google::protobuf::internal::GeneratedMessageReflection*
      Person_reflection_ = NULL;
    const ::google::protobuf::Descriptor* Person_PhoneNumber_descriptor_ = NULL;
    const ::google::protobuf::internal::GeneratedMessageReflection*
      Person_PhoneNumber_reflection_ = NULL;
    const ::google::protobuf::EnumDescriptor* Person_PhoneType_descriptor_ = NULL;
    const ::google::protobuf::Descriptor* AddressBook_descriptor_ = NULL;
    const ::google::protobuf::internal::GeneratedMessageReflection*
      AddressBook_reflection_ = NULL;       
    }  // namespace

    void protobuf_AssignDesc_addressbook_2eproto() {
      // 这个方法用于注册静态信息
      protobuf_AddDesc_addressbook_2eproto();
      const ::google::protobuf::FileDescriptor* file =
        ::google::protobuf::DescriptorPool::generated_pool()->FindFileByName(
          "addressbook.proto");
      GOOGLE_CHECK(file != NULL);
      Person_descriptor_ = file->message_type(0);
      static const int Person_offsets_[4] = {
        GOOGLE_PROTOBUF_GENERATED_MESSAGE_FIELD_OFFSET(Person, name_),
        GOOGLE_PROTOBUF_GENERATED_MESSAGE_FIELD_OFFSET(Person, id_),
        GOOGLE_PROTOBUF_GENERATED_MESSAGE_FIELD_OFFSET(Person, email_),
        GOOGLE_PROTOBUF_GENERATED_MESSAGE_FIELD_OFFSET(Person, phone_),
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
      AddressBook_descriptor_ = file->message_type(1);
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

    namespace { 

    // 这里让protobuf_AssignDesc_addressbook_2proto方法有且被执行一遍
    GOOGLE_PROTOBUF_DECLARE_ONCE(protobuf_AssignDescriptors_once_);
    inline void protobuf_AssignDescriptorsOnce() {
      ::google::protobuf::GoogleOnceInit(&protobuf_AssignDescriptors_once_,
                     &protobuf_AssignDesc_addressbook_2eproto);
    }
    void protobuf_RegisterTypes(const ::std::string&) {
      protobuf_AssignDescriptorsOnce();
      ::google::protobuf::MessageFactory::InternalRegisterGeneratedMessage(
        Person_descriptor_, &Person::default_instance());
      ::google::protobuf::MessageFactory::InternalRegisterGeneratedMessage(
        Person_PhoneNumber_descriptor_, &Person_PhoneNumber::default_instance());
      ::google::protobuf::MessageFactory::InternalRegisterGeneratedMessage(
        AddressBook_descriptor_, &AddressBook::default_instance());
    }   

    }  // namespace

    void protobuf_ShutdownFile_addressbook_2eproto() {
      delete Person::default_instance_;
      delete Person_reflection_;
      delete Person_PhoneNumber::default_instance_;
      delete Person_PhoneNumber_reflection_;
      delete AddressBook::default_instance_;
      delete AddressBook_reflection_;
    }   

    void protobuf_AddDesc_addressbook_2eproto() {
      static bool already_here = false;
      if (already_here) return;
      already_here = true;
      GOOGLE_PROTOBUF_VERIFY_VERSION;   

      ::google::protobuf::DescriptorPool::InternalAddGeneratedFile(
        "\n\021addressbook.proto\022\010tutorial\"\332\001\n\006Person"
        "\022\014\n\004name\030\001 \002(\t\022\n\n\002id\030\002 \002(\005\022\r\n\005email\030\003 \001("
        "\t\022+\n\005phone\030\004 \003(\0132\034.tutorial.Person.Phone"
        "Number\032M\n\013PhoneNumber\022\016\n\006number\030\001 \002(\t\022.\n"
        "\004type\030\002 \001(\0162\032.tutorial.Person.PhoneType:"
        "\004HOME\"+\n\tPhoneType\022\n\n\006MOBILE\020\000\022\010\n\004HOME\020\001"
        "\022\010\n\004WORK\020\002\"/\n\013AddressBook\022 \n\006person\030\001 \003("
        "\0132\020.tutorial.PersonB)\n\024com.example.tutor"
        "ialB\021AddressBookProtos", 342);
      ::google::protobuf::MessageFactory::InternalRegisterGeneratedFile(
        "addressbook.proto", &protobuf_RegisterTypes);
      Person::default_instance_ = new Person();
      Person_PhoneNumber::default_instance_ = new Person_PhoneNumber();
      AddressBook::default_instance_ = new AddressBook();
      Person::default_instance_->InitAsDefaultInstance();
      Person_PhoneNumber::default_instance_->InitAsDefaultInstance();
      AddressBook::default_instance_->InitAsDefaultInstance();
      ::google::protobuf::internal::OnShutdown(&protobuf_ShutdownFile_addressbook_2eproto);
    }

在动态代码中生成Reflection：（这个就拿上面的动态信息Demo举例）

    //生成代码
    DynamicMessageFactory *factory = new DynamicMessageFactory;
    const Message *prototype = factory->GetPrototype(descriptor);
    const Reflection *reflection = message->GetReflection();

    //下面就要看看DynamicMessageFactory->GetPrototype(descriptor),这个方法会返回一个Message对象。dynamic_message.cc line 471

    const Message* DynamicMessageFactory::GetPrototypeNoLock( const Descriptor* type) {

        //1. 看看是不是已经编译进来的类型：
        if (delegate_to_generated_factory_ &&
            type->file()->pool() == DescriptorPool::generated_pool()) {
            return MessageFactory::generated_factory()->GetPrototype(type);
        }

        //2. 看看map_中有没有

        //3. 构建DynamicMessage::TypeInfo
        // 需要三个方面的数据来构造GeneratedMessageReflection: memory, offsets of each field, bitfield


        //4. 构建GeneratedMessageReflection对象

        //5. 构建DynamicMessage prototype

        //6. prototype->CrossLinkPrototypes(); 这个方法主要功能是将Message中的依然为Messge的属性也调用GetProtoType

        //7. 返回prototype
    }

## Descriptor

descriptor.h

## DynamicMessage

dynamic_message.cc

从任意的Descriptor构造Message，DynamicMessage提供了这个功能。该类中使用了[placement new](https://www.google.com.hk/search?q=placement+new)。

重要函数：

  //构造函数
  DynamicMessage::DynamicMessage(const TypeInfo* type_info){
    分配type_info中的offsets[i]所代表的FileDescriptor* 所指向的内存。
  }

  DynamicMessage::~DynamicMessage() {}

## DynamicMessageFactory

google/protobuf/dynamic_message.h

主要方法是MessageFactory的主要方法，上面讲Message的时候已经描述了：

  const Message *GetPrototype(const Descriptor* type);

## Reflection

google/protobuf/message.h

Reflection接口包含用于`动态地`访问和修改protocol消息的方法。大部分的Message的GetReflection()方法，都用的GeneratedMessageReflection实现(generated_message_reflection.h)。由于是反射接口，肯定不如生成的类那么安全。比如说：

+ FieldDescriptor 不属于这个message
+ 比如对INT type调用 GETSTRING，会有exception抛出
+ 在repeated field上调用非repeated GET方法
+ 在非repeated field上调用repeated GET方法
+ reflection与message不符

主要接口有Get/Set/Add，具体可以参考message.h。

### GeneratedMessageReflection

generated_message_reflection.h

  #define DEFINE_PRIMITIVE_ACCESSORS(TYPENAME, TYPE, PASSTYPE, CPPTYPE) 

## MessageFactory

  //单例模式，所有的compiled-in的Message，都会往MessageFactory中注册自己。
  static MessageFactory* generated_factory();

  //下面两个函数用于在自动化生成的类中，往MessageFactory中注册
  static void InternalRegisterGeneratedFile(
      const char* filename, void (*register_messages)(const string&));
  static void InternalRegisterGeneratedMessage(const Descriptor* descriptor,
                                               const Message* prototype);

