---
layout: post
title:  protobuf中的UT
---

UnitTest无疑是代码最好的保镖，有时编写UT甚至可以指导我们写出更好的代码结构&设计。


# descriptor_unittest

文件位于 google/protobuf/descriptor_unittest.cc

头文件和namespace略过。

后面是一些辅助函数：

    DescriptorProto* AddMessage(FileDescriptorProto* file, const string& name);
    DescriptorProto* AddNestedMessage(DescriptorProto* parent, const string& name);
    EnumDescriptorProto* AddEnum(FileDescriptorProto* file, const string& name);
    EnumDescriptorProto* AddNestedEnum(DescriptorProto* parent, const string& name);
    ServiceDescriptorProto* AddService(FileDescriptorProto* file, const string& name);
    FieldDescriptorProto* AddField(DescriptorProto* parent,
                               const string& name, int number,
                               FieldDescriptorProto::Label label,
                               FieldDescriptorProto::Type type);
    FieldDescriptorProto* AddExtension(FileDescriptorProto* file,
                                   const string& extendee,
                                   const string& name, int number,
                                   FieldDescriptorProto::Label label,
                                   FieldDescriptorProto::Type type);
    FieldDescriptorProto* AddNestedExtension(DescriptorProto* parent,
                                         const string& extendee,
                                         const string& name, int number,
                                         FieldDescriptorProto::Label label,
                                         FieldDescriptorProto::Type type);
    DescriptorProto::ExtensionRange* AddExtensionRange(DescriptorProto* parent,
                                                   int start, int end);
    EnumValueDescriptorProto* AddEnumValue(EnumDescriptorProto* enum_proto,
                                       const string& name, int number);
    MethodDescriptorProto* AddMethod(ServiceDescriptorProto* service,
                                 const string& name,
                                 const string& input_type,
                                 const string& output_type);
    void AddEmptyEnum(FileDescriptorProto* file, const string& name);

第一个测试类是FileDescriptorTest，用户测试简单的文件的Build

    class FileDescriptorTest : public testing::Test {
     protected:
      virtual void SetUp() {
        FileDescriptorProto foo_file;
        foo_file.set_name("foo.proto");
        AddExtensionRange(AddMessage(&foo_file, "FooMessage"), 1, 2);
        AddEnumValue(AddEnum(&foo_file, "FooEnum"), "FOO_ENUM_VALUE", 1);
        AddService(&foo_file, "FooService");
        AddExtension(&foo_file, "FooMessage", "foo_extension", 1,
                     FieldDescriptorProto::LABEL_OPTIONAL,
                     FieldDescriptorProto::TYPE_INT32); 

        FileDescriptorProto bar_file;
        bar_file.set_name("bar.proto");
        bar_file.set_package("bar_package");
        bar_file.add_dependency("foo.proto");
        AddExtensionRange(AddMessage(&bar_file, "BarMessage"), 1, 2);
        AddEnumValue(AddEnum(&bar_file, "BarEnum"), "BAR_ENUM_VALUE", 1);
        AddService(&bar_file, "BarService");
        AddExtension(&bar_file, "bar_package.BarMessage", "bar_extension", 1,
                     FieldDescriptorProto::LABEL_OPTIONAL,
                     FieldDescriptorProto::TYPE_INT32); 

        FileDescriptorProto baz_file;
        baz_file.set_name("baz.proto"); 

        // Build the descriptors and get the pointers.
        foo_file_ = pool_.BuildFile(foo_file);
        ASSERT_TRUE(foo_file_ != NULL); 

        bar_file_ = pool_.BuildFile(bar_file);
        ASSERT_TRUE(bar_file_ != NULL); 

        baz_file_ = pool_.BuildFile(baz_file);
        ASSERT_TRUE(baz_file_ != NULL); 

        ASSERT_EQ(1, foo_file_->message_type_count());
        foo_message_ = foo_file_->message_type(0);
        ASSERT_EQ(1, foo_file_->enum_type_count());
        foo_enum_ = foo_file_->enum_type(0);
        ASSERT_EQ(1, foo_enum_->value_count());
        foo_enum_value_ = foo_enum_->value(0);
        ASSERT_EQ(1, foo_file_->service_count());
        foo_service_ = foo_file_->service(0);
        ASSERT_EQ(1, foo_file_->extension_count());
        foo_extension_ = foo_file_->extension(0);   

        ASSERT_EQ(1, bar_file_->message_type_count());
        bar_message_ = bar_file_->message_type(0);
        ASSERT_EQ(1, bar_file_->enum_type_count());
        bar_enum_ = bar_file_->enum_type(0);
        ASSERT_EQ(1, bar_enum_->value_count());
        bar_enum_value_ = bar_enum_->value(0);
        ASSERT_EQ(1, bar_file_->service_count());
        bar_service_ = bar_file_->service(0);
        ASSERT_EQ(1, bar_file_->extension_count());
        bar_extension_ = bar_file_->extension(0);
      } 

      //用于构造各种Descriptor的DP
      DescriptorPool pool_; 

      const FileDescriptor* foo_file_;
      const FileDescriptor* bar_file_;
      const FileDescriptor* baz_file_;  

      const Descriptor*          foo_message_;
      const EnumDescriptor*      foo_enum_;
      const EnumValueDescriptor* foo_enum_value_;
      const ServiceDescriptor*   foo_service_;
      const FieldDescriptor*     foo_extension_;    

      const Descriptor*          bar_message_;
      const EnumDescriptor*      bar_enum_;
      const EnumValueDescriptor* bar_enum_value_;
      const ServiceDescriptor*   bar_service_;
      const FieldDescriptor*     bar_extension_;
    };

    //为每个FileDescriptor的功能点，都创建一个TEST_F
    TEST_F(FileDescriptorTest, Name) {
      EXPECT_EQ("foo.proto", foo_file_->name());
      EXPECT_EQ("bar.proto", bar_file_->name());
      EXPECT_EQ("baz.proto", baz_file_->name());
    }   

    TEST_F(FileDescriptorTest, Package) {
      EXPECT_EQ("", foo_file_->package());
      EXPECT_EQ("bar_package", bar_file_->package());
    }   

    TEST_F(FileDescriptorTest, Dependencies) {
      EXPECT_EQ(0, foo_file_->dependency_count());
      EXPECT_EQ(1, bar_file_->dependency_count());
      EXPECT_EQ(foo_file_, bar_file_->dependency(0));
    }   

    //通过这些Test也可以看到，Find**By**如果没有找到会返回NULL
    TEST_F(FileDescriptorTest, FindMessageTypeByName) {
      EXPECT_EQ(foo_message_, foo_file_->FindMessageTypeByName("FooMessage"));
      EXPECT_EQ(bar_message_, bar_file_->FindMessageTypeByName("BarMessage"));  

      EXPECT_TRUE(foo_file_->FindMessageTypeByName("BarMessage") == NULL);
      EXPECT_TRUE(bar_file_->FindMessageTypeByName("FooMessage") == NULL);
      EXPECT_TRUE(baz_file_->FindMessageTypeByName("FooMessage") == NULL);  

      EXPECT_TRUE(foo_file_->FindMessageTypeByName("NoSuchMessage") == NULL);
      EXPECT_TRUE(foo_file_->FindMessageTypeByName("FooEnum") == NULL);
    }   

    TEST_F(FileDescriptorTest, FindEnumTypeByName) {
      EXPECT_EQ(foo_enum_, foo_file_->FindEnumTypeByName("FooEnum"));
      EXPECT_EQ(bar_enum_, bar_file_->FindEnumTypeByName("BarEnum"));   

      EXPECT_TRUE(foo_file_->FindEnumTypeByName("BarEnum") == NULL);
      EXPECT_TRUE(bar_file_->FindEnumTypeByName("FooEnum") == NULL);
      EXPECT_TRUE(baz_file_->FindEnumTypeByName("FooEnum") == NULL);    

      EXPECT_TRUE(foo_file_->FindEnumTypeByName("NoSuchEnum") == NULL);
      EXPECT_TRUE(foo_file_->FindEnumTypeByName("FooMessage") == NULL);
    }   

    TEST_F(FileDescriptorTest, FindEnumValueByName) {
      EXPECT_EQ(foo_enum_value_, foo_file_->FindEnumValueByName("FOO_ENUM_VALUE"));
      EXPECT_EQ(bar_enum_value_, bar_file_->FindEnumValueByName("BAR_ENUM_VALUE")); 

      EXPECT_TRUE(foo_file_->FindEnumValueByName("BAR_ENUM_VALUE") == NULL);
      EXPECT_TRUE(bar_file_->FindEnumValueByName("FOO_ENUM_VALUE") == NULL);
      EXPECT_TRUE(baz_file_->FindEnumValueByName("FOO_ENUM_VALUE") == NULL);    

      EXPECT_TRUE(foo_file_->FindEnumValueByName("NO_SUCH_VALUE") == NULL);
      EXPECT_TRUE(foo_file_->FindEnumValueByName("FooMessage") == NULL);
    }   

    TEST_F(FileDescriptorTest, FindServiceByName) {
      EXPECT_EQ(foo_service_, foo_file_->FindServiceByName("FooService"));
      EXPECT_EQ(bar_service_, bar_file_->FindServiceByName("BarService"));  

      EXPECT_TRUE(foo_file_->FindServiceByName("BarService") == NULL);
      EXPECT_TRUE(bar_file_->FindServiceByName("FooService") == NULL);
      EXPECT_TRUE(baz_file_->FindServiceByName("FooService") == NULL);  

      EXPECT_TRUE(foo_file_->FindServiceByName("NoSuchService") == NULL);
      EXPECT_TRUE(foo_file_->FindServiceByName("FooMessage") == NULL);
    }   

    TEST_F(FileDescriptorTest, FindExtensionByName) {
      EXPECT_EQ(foo_extension_, foo_file_->FindExtensionByName("foo_extension"));
      EXPECT_EQ(bar_extension_, bar_file_->FindExtensionByName("bar_extension"));   

      EXPECT_TRUE(foo_file_->FindExtensionByName("bar_extension") == NULL);
      EXPECT_TRUE(bar_file_->FindExtensionByName("foo_extension") == NULL);
      EXPECT_TRUE(baz_file_->FindExtensionByName("foo_extension") == NULL); 

      EXPECT_TRUE(foo_file_->FindExtensionByName("no_such_extension") == NULL);
      EXPECT_TRUE(foo_file_->FindExtensionByName("FooMessage") == NULL);
    }   

    TEST_F(FileDescriptorTest, FindExtensionByNumber) {
      EXPECT_EQ(foo_extension_, pool_.FindExtensionByNumber(foo_message_, 1));
      EXPECT_EQ(bar_extension_, pool_.FindExtensionByNumber(bar_message_, 1));  

      EXPECT_TRUE(pool_.FindExtensionByNumber(foo_message_, 2) == NULL);
    }   

    //这里测试rebuild的时候的表现
    TEST_F(FileDescriptorTest, BuildAgain) {
      // Test that if te call BuildFile again on the same input we get the same
      // FileDescriptor back.
      FileDescriptorProto file;
      foo_file_->CopyTo(&file);
      EXPECT_EQ(foo_file_, pool_.BuildFile(file));  

      // But if we change the file then it won't work.
      file.set_package("some.other.package");
      EXPECT_TRUE(pool_.BuildFile(file) == NULL);
    }

DescriptorTest 测试了BUILD Descriptor相关的逻辑。

    class DescriptorTest : public testing::Test {
        protected:
            //构造若干FileDescriptor，Descriptor，FieldDescriptor，EnumDescriptor
            virtual void SetUp() {}

            //保存上面build/find出来的信息的字段。
    }

    TEST_F(DescriptorTest, Name) {
      EXPECT_EQ("TestMessage", message_->name());
      EXPECT_EQ("TestMessage", message_->full_name());
      EXPECT_EQ(foo_file_, message_->file());   

      EXPECT_EQ("TestMessage2", message2_->name());
      EXPECT_EQ("corge.grault.TestMessage2", message2_->full_name());
      EXPECT_EQ(bar_file_, message2_->file());
    }   

    TEST_F(DescriptorTest, ContainingType) {
      EXPECT_TRUE(message_->containing_type() == NULL);
      EXPECT_TRUE(message2_->containing_type() == NULL);
    }   

    TEST_F(DescriptorTest, FieldsByIndex) {
      ASSERT_EQ(4, message_->field_count());
      EXPECT_EQ(foo_, message_->field(0));
      EXPECT_EQ(bar_, message_->field(1));
      EXPECT_EQ(baz_, message_->field(2));
      EXPECT_EQ(qux_, message_->field(3));
    }   

    TEST_F(DescriptorTest, FindFieldByName) {
      // All messages in the same DescriptorPool share a single lookup table for
      // fields.  So, in addition to testing that FindFieldByName finds the fields
      // of the message, we need to test that it does *not* find the fields of
      // *other* messages.  

      EXPECT_EQ(foo_, message_->FindFieldByName("foo"));
      EXPECT_EQ(bar_, message_->FindFieldByName("bar"));
      EXPECT_EQ(baz_, message_->FindFieldByName("baz"));
      EXPECT_EQ(qux_, message_->FindFieldByName("qux"));
      EXPECT_TRUE(message_->FindFieldByName("no_such_field") == NULL);
      EXPECT_TRUE(message_->FindFieldByName("quux") == NULL);   

      EXPECT_EQ(foo2_ , message2_->FindFieldByName("foo" ));
      EXPECT_EQ(bar2_ , message2_->FindFieldByName("bar" ));
      EXPECT_EQ(quux2_, message2_->FindFieldByName("quux"));
      EXPECT_TRUE(message2_->FindFieldByName("baz") == NULL);
      EXPECT_TRUE(message2_->FindFieldByName("qux") == NULL);
    }   

    TEST_F(DescriptorTest, FindFieldByNumber) {
      EXPECT_EQ(foo_, message_->FindFieldByNumber(1));
      EXPECT_EQ(bar_, message_->FindFieldByNumber(6));
      EXPECT_EQ(baz_, message_->FindFieldByNumber(500000000));
      EXPECT_EQ(qux_, message_->FindFieldByNumber(15));
      EXPECT_TRUE(message_->FindFieldByNumber(837592) == NULL);
      EXPECT_TRUE(message_->FindFieldByNumber(2) == NULL);  

      EXPECT_EQ(foo2_ , message2_->FindFieldByNumber(1));
      EXPECT_EQ(bar2_ , message2_->FindFieldByNumber(2));
      EXPECT_EQ(quux2_, message2_->FindFieldByNumber(6));
      EXPECT_TRUE(message2_->FindFieldByNumber(15) == NULL);
      EXPECT_TRUE(message2_->FindFieldByNumber(500000000) == NULL);
    }   

    TEST_F(DescriptorTest, FieldName) {
      EXPECT_EQ("foo", foo_->name());
      EXPECT_EQ("bar", bar_->name());
      EXPECT_EQ("baz", baz_->name());
      EXPECT_EQ("qux", qux_->name());
    }   

    TEST_F(DescriptorTest, FieldFullName) {
      EXPECT_EQ("TestMessage.foo", foo_->full_name());
      EXPECT_EQ("TestMessage.bar", bar_->full_name());
      EXPECT_EQ("TestMessage.baz", baz_->full_name());
      EXPECT_EQ("TestMessage.qux", qux_->full_name());  

      EXPECT_EQ("corge.grault.TestMessage2.foo", foo2_->full_name());
      EXPECT_EQ("corge.grault.TestMessage2.bar", bar2_->full_name());
      EXPECT_EQ("corge.grault.TestMessage2.quux", quux2_->full_name());
    }   

    TEST_F(DescriptorTest, FieldFile) {
      EXPECT_EQ(foo_file_, foo_->file());
      EXPECT_EQ(foo_file_, bar_->file());
      EXPECT_EQ(foo_file_, baz_->file());
      EXPECT_EQ(foo_file_, qux_->file());   

      EXPECT_EQ(bar_file_, foo2_->file());
      EXPECT_EQ(bar_file_, bar2_->file());
      EXPECT_EQ(bar_file_, quux2_->file());
    }   

    TEST_F(DescriptorTest, FieldIndex) {
      EXPECT_EQ(0, foo_->index());
      EXPECT_EQ(1, bar_->index());
      EXPECT_EQ(2, baz_->index());
      EXPECT_EQ(3, qux_->index());
    }   

    TEST_F(DescriptorTest, FieldNumber) {
      EXPECT_EQ(        1, foo_->number());
      EXPECT_EQ(        6, bar_->number());
      EXPECT_EQ(500000000, baz_->number());
      EXPECT_EQ(       15, qux_->number());
    }   

    TEST_F(DescriptorTest, FieldType) {
      EXPECT_EQ(FieldDescriptor::TYPE_STRING , foo_->type());
      EXPECT_EQ(FieldDescriptor::TYPE_ENUM   , bar_->type());
      EXPECT_EQ(FieldDescriptor::TYPE_MESSAGE, baz_->type());
      EXPECT_EQ(FieldDescriptor::TYPE_GROUP  , qux_->type());
    }   

    TEST_F(DescriptorTest, FieldLabel) {
      EXPECT_EQ(FieldDescriptor::LABEL_REQUIRED, foo_->label());
      EXPECT_EQ(FieldDescriptor::LABEL_OPTIONAL, bar_->label());
      EXPECT_EQ(FieldDescriptor::LABEL_REPEATED, baz_->label());
      EXPECT_EQ(FieldDescriptor::LABEL_OPTIONAL, qux_->label());    

      EXPECT_TRUE (foo_->is_required());
      EXPECT_FALSE(foo_->is_optional());
      EXPECT_FALSE(foo_->is_repeated());    

      EXPECT_FALSE(bar_->is_required());
      EXPECT_TRUE (bar_->is_optional());
      EXPECT_FALSE(bar_->is_repeated());    

      EXPECT_FALSE(baz_->is_required());
      EXPECT_FALSE(baz_->is_optional());
      EXPECT_TRUE (baz_->is_repeated());
    }   

    TEST_F(DescriptorTest, FieldHasDefault) {
      EXPECT_FALSE(foo_->has_default_value());
      EXPECT_FALSE(bar_->has_default_value());
      EXPECT_FALSE(baz_->has_default_value());
      EXPECT_FALSE(qux_->has_default_value());
    }   

    TEST_F(DescriptorTest, FieldContainingType) {
      EXPECT_EQ(message_, foo_->containing_type());
      EXPECT_EQ(message_, bar_->containing_type());
      EXPECT_EQ(message_, baz_->containing_type());
      EXPECT_EQ(message_, qux_->containing_type()); 

      EXPECT_EQ(message2_, foo2_ ->containing_type());
      EXPECT_EQ(message2_, bar2_ ->containing_type());
      EXPECT_EQ(message2_, quux2_->containing_type());
    }   

    TEST_F(DescriptorTest, FieldMessageType) {
      EXPECT_TRUE(foo_->message_type() == NULL);
      EXPECT_TRUE(bar_->message_type() == NULL);    

      EXPECT_EQ(foreign_, baz_->message_type());
      EXPECT_EQ(foreign_, qux_->message_type());
    }   

    TEST_F(DescriptorTest, FieldEnumType) {
      EXPECT_TRUE(foo_->enum_type() == NULL);
      EXPECT_TRUE(baz_->enum_type() == NULL);
      EXPECT_TRUE(qux_->enum_type() == NULL);   

      EXPECT_EQ(enum_, bar_->enum_type());
    }


ValidationErrorTest 是用于测试各种解析proto时发生错误的。

    //首先是一个Mock ErrorCollector的
    class MockErrorCollector : public DescriptorPool::ErrorCollector {
     public:
      MockErrorCollector() {}
      ~MockErrorCollector() {}  

      string text_; 

      // implements ErrorCollector ---------------------------------------
      void AddError(const string& filename,
                    const string& element_name, const Message* descriptor,
                    ErrorLocation location, const string& message) {
        const char* location_name = NULL;
        switch (location) {
          case NAME         : location_name = "NAME"         ; break;
          case NUMBER       : location_name = "NUMBER"       ; break;
          case TYPE         : location_name = "TYPE"         ; break;
          case EXTENDEE     : location_name = "EXTENDEE"     ; break;
          case DEFAULT_VALUE: location_name = "DEFAULT_VALUE"; break;
          case OPTION_NAME  : location_name = "OPTION_NAME"  ; break;
          case OPTION_VALUE : location_name = "OPTION_VALUE" ; break;
          case INPUT_TYPE   : location_name = "INPUT_TYPE"   ; break;
          case OUTPUT_TYPE  : location_name = "OUTPUT_TYPE"  ; break;
          case OTHER        : location_name = "OTHER"        ; break;
        }   

        strings::SubstituteAndAppend(
          &text_, "$0: $1: $2: $3\n",
          filename, element_name, location_name, message);
      }
    };

    class ValidationErrorTest : public testing::Test {
     protected:
      // Parse file_text as a FileDescriptorProto in text format and add it
      // to the DescriptorPool.  Expect no errors.
      void BuildFile(const string& file_text) {
        FileDescriptorProto file_proto;
        ASSERT_TRUE(TextFormat::ParseFromString(file_text, &file_proto));
        ASSERT_TRUE(pool_.BuildFile(file_proto) != NULL);
      } 

      // Parse file_text as a FileDescriptorProto in text format and add it
      // to the DescriptorPool.  Expect errors to be produced which match the
      // given error text.
      void BuildFileWithErrors(const string& file_text,
                               const string& expected_errors) {
        FileDescriptorProto file_proto;
        ASSERT_TRUE(TextFormat::ParseFromString(file_text, &file_proto));   

        MockErrorCollector error_collector;
        EXPECT_TRUE(
          pool_.BuildFileCollectingErrors(file_proto, &error_collector) == NULL);
        EXPECT_EQ(expected_errors, error_collector.text_);
      }

      DescriptorPool pool_;
    };

    //这个方法测试了AlreadyDefined的case，返回的错误描述会和expected的做比较。
    TEST_F(ValidationErrorTest, AlreadyDefined) {
      BuildFileWithErrors(
        "name: \"foo.proto\" "
        "message_type { name: \"Foo\" }"
        "message_type { name: \"Foo\" }",   

        "foo.proto: Foo: NAME: \"Foo\" is already defined.\n");
    }

Protobuf中没有使用GMock来做单元测试，相信是因为代码比较老的缘故。