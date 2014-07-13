---
layout: default
title:  protobuf中的几个概念
---

主要讲一下FileDescriptorProto、FileDescriptor、Descriptor等等。

一个.proto文件，可以打开成流，使用Tokenizer读取，使用Parser可以Parse成为FileDescriptorProto。

一个FileDescriptorProto再经过DescriptorBuilder的BuildFile(proto)方法，build成FileDescriptor

# descriptor.proto中的xxxProto

引用该文件的注释中所说：

	// The messages in this file describe the definitions found in .proto files.
	// A valid .proto file can be translated directly to a FileDescriptorProto
	// without any other information (e.g. without reading its imports).

这个文件中的消息描述.proto文件中的定义。一个合法的.proto文件可以被直接翻译为一个FileDescriptorProto而不需要任何其他信息（比如说，不需要读取它的imports）


	// Describes a complete .proto file.
	message FileDescriptorProto {
	  optional string name = 1;       // file name, relative to root of source tree
	  optional string package = 2;    // e.g. "foo", "foo.bar", etc.	

	  // Names of files imported by this file.
	  repeated string dependency = 3;
	  // Indexes of the public imported files in the dependency list above.
	  repeated int32 public_dependency = 10;
	  // Indexes of the weak imported files in the dependency list.
	  // For Google-internal migration only. Do not use.
	  repeated int32 weak_dependency = 11;	

	  // All top-level definitions in this file.
	  repeated DescriptorProto message_type = 4;
	  repeated EnumDescriptorProto enum_type = 5;
	  repeated ServiceDescriptorProto service = 6;
	  repeated FieldDescriptorProto extension = 7;	

	  optional FileOptions options = 8;	

	  // This field contains optional information about the original source code.
	  // You may safely remove this entire field whithout harming runtime
	  // functionality of the descriptors -- the information is needed only by
	  // development tools.
	  optional SourceCodeInfo source_code_info = 9;
	}

	// Describes a message type.
	message DescriptorProto {
	  optional string name = 1;	

	  repeated FieldDescriptorProto field = 2;
	  repeated FieldDescriptorProto extension = 6;	

	  repeated DescriptorProto nested_type = 3;
	  repeated EnumDescriptorProto enum_type = 4;	

	  message ExtensionRange {
	    optional int32 start = 1;
	    optional int32 end = 2;
	  }
	  repeated ExtensionRange extension_range = 5;	

	  optional MessageOptions options = 7;
	}

# 一系列Descriptor

## FileDescriptor 

定义在descriptor.h中 line 888。该类描述了一个.proto文件。如果想要得到一个编译文件的FileDescriptor，只要拿到这个文件中定义的某个descriptor，再调用descriptor->file()就可以。使用DescriptorPool来构建你自己的descriptors。

	// See Descriptor::CopyTo().
  	void CopyTo(MethodDescriptorProto* proto) const;

## Descriptor

定义在descriptor.h中 line 127。该类描述了一个protocol message，或者一个message中的特定组。要想得到一个message的Descriptor，只要调用Message::GetDescriptor()。自动生成的message类也有个静态方法descriptor()可以返回类型的descriptor。使用DescriptorPool生成自己的descriptors。

对于DescriptorProto：

	// Write the contents of this Descriptor into the given DescriptorProto.
  	// The target DescriptorProto must be clear before calling this; if it
  	// isn't, the result may be garbage.
  	void CopyTo(DescriptorProto* proto) const;

## FieldDescriptor

对应FieldDescriptorProto，
	
	void CopyTo(FieldDescriptorProto* proto) const;
	
# xxxDescriptor 与 xxxDescriptorProto的关系

我们就以Descriptor这个简单的例子来说明两者的关系。

如descriptor.proto文件中所说，一个文件都可以无缝的翻译成FileDescriptorProto，所以说xxxDescriptorProto基本可以认为是和文件相关的。

DescriptorProto：

	// Describes a message type.
	message DescriptorProto {
	  optional string name = 1;	

	  repeated FieldDescriptorProto field = 2;
	  repeated FieldDescriptorProto extension = 6;	

	  repeated DescriptorProto nested_type = 3;
	  repeated EnumDescriptorProto enum_type = 4;	

	  message ExtensionRange {
	    optional int32 start = 1;
	    optional int32 end = 2;
	  }
	  repeated ExtensionRange extension_range = 5;	

	  optional MessageOptions options = 7;
	}

Descriptor：

	// Describes a type of protocol message, or a particular group within a
	// message.  To obtain the Descriptor for a given message object, call
	// Message::GetDescriptor().  Generated message classes also have a
	// static method called descriptor() which returns the type's descriptor.
	// Use DescriptorPool to construct your own descriptors.
	class LIBPROTOBUF_EXPORT Descriptor {
	 public:
	  // The name of the message type, not including its scope.
	  const string& name() const;	
	  const string& full_name() const;	
	  int index() const;	
	  const FileDescriptor* file() const;	
	  const Descriptor* containing_type() const;	
	  const MessageOptions& options() const;	
	  void CopyTo(DescriptorProto* proto) const;	
	  string DebugString() const;	

	  // Field stuff -----------------------------------------------------	
	  int field_count() const;
	  const FieldDescriptor* field(int index) const;	
	  const FieldDescriptor* FindFieldByNumber(int number) const;
	  const FieldDescriptor* FindFieldByName(const string& name) const;	
	  const FieldDescriptor* FindFieldByLowercaseName(
	  const string& lowercase_name) const;	
	  const FieldDescriptor* FindFieldByCamelcaseName(
	  const string& camelcase_name) const;	
	  // Nested type stuff -----------------------------------------------	
	  int nested_type_count() const;
	  const Descriptor* nested_type(int index) const;	
	  const Descriptor* FindNestedTypeByName(const string& name) const;	

	  // Enum stuff ------------------------------------------------------	
	  int enum_type_count() const;
	  const EnumDescriptor* enum_type(int index) const;	
	  const EnumDescriptor* FindEnumTypeByName(const string& name) const;	
	  const EnumValueDescriptor* FindEnumValueByName(const string& name) const;	

	  // Extensions ------------------------------------------------------	

	  struct ExtensionRange {
	    int start;  // inclusive
	    int end;    // exclusive
	  };	

	  int extension_range_count() const;
	  const ExtensionRange* extension_range(int index) const;	
	  bool IsExtensionNumber(int number) const;	
	  int extension_count() const;
	  const FieldDescriptor* extension(int index) const;	
	  const FieldDescriptor* FindExtensionByName(const string& name) const;	
	  const FieldDescriptor* FindExtensionByLowercaseName(const string& name) const;	
	  const FieldDescriptor* FindExtensionByCamelcaseName(const string& name) const;	

	  // Source Location ---------------------------------------------------	
	  bool GetSourceLocation(SourceLocation* out_location) const;	

	 private:
	  //...

	  const string* name_;
	  const string* full_name_;
	  const FileDescriptor* file_;
	  const Descriptor* containing_type_;
	  const MessageOptions* options_;	

	  // True if this is a placeholder for an unknown type.
	  bool is_placeholder_;
	  // True if this is a placeholder and the type name wasn't fully-qualified.
	  bool is_unqualified_placeholder_;	

	  int field_count_;
	  FieldDescriptor* fields_;
	  int nested_type_count_;
	  Descriptor* nested_types_;
	  int enum_type_count_;
	  EnumDescriptor* enum_types_;
	  int extension_range_count_;
	  ExtensionRange* extension_ranges_;
	  int extension_count_;
	  FieldDescriptor* extensions_;
	  // IMPORTANT:  If you add a new field, make sure to search for all instances
	  // of Allocate<Descriptor>() and AllocateArray<Descriptor>() in descriptor.cc
	  // and update them to initialize the field.	

	  // Must be constructed using DescriptorPool.
	  Descriptor() {}
	  friend class DescriptorBuilder;
	  friend class EnumDescriptor;
	  friend class FieldDescriptor;
	  friend class MethodDescriptor;
	  friend class FileDescriptor;
	  GOOGLE_DISALLOW_EVIL_CONSTRUCTORS(Descriptor);
	};

重点看下Descriptor的private部分和DescriptorProto的比较，基本是差不多的。二者的相互转化如下：

1. Descriptor转DescriptorProto 
	
	void Descriptor::CopyTo(DescriptorProto* proto) const;

2. DescriptorProto转Descriptor
	// descriptor.cc line 3251
	void DescriptorBuilder::BuildMessage(const DescriptorProto& proto,
                                     const Descriptor* parent,
                                     Descriptor* result)

## 1. Descriptor::CopyTo

	void Descriptor::CopyTo(DescriptorProto* proto) const {
	  proto->set_name(name());	

	  for (int i = 0; i < field_count(); i++) {
	    field(i)->CopyTo(proto->add_field());
	  }
	  for (int i = 0; i < nested_type_count(); i++) {
	    nested_type(i)->CopyTo(proto->add_nested_type());
	  }
	  for (int i = 0; i < enum_type_count(); i++) {
	    enum_type(i)->CopyTo(proto->add_enum_type());
	  }
	  for (int i = 0; i < extension_range_count(); i++) {
	    DescriptorProto::ExtensionRange* range = proto->add_extension_range();
	    range->set_start(extension_range(i)->start);
	    range->set_end(extension_range(i)->end);
	  }
	  for (int i = 0; i < extension_count(); i++) {
	    extension(i)->CopyTo(proto->add_extension());
	  }	

	  if (&options() != &MessageOptions::default_instance()) {
	    proto->mutable_options()->CopyFrom(options());
	  }
	}

## 2. DescriptorBuilder::BuildMessage

	void DescriptorBuilder::BuildMessage(const DescriptorProto& proto,
                                     const Descriptor* parent,
	                                     Descriptor* result) {
	  const string& scope = (parent == NULL) ?
	    file_->package() : parent->full_name();
	  string* full_name = tables_->AllocateString(scope);
	  if (!full_name->empty()) full_name->append(1, '.');
	  full_name->append(proto.name());	

	  ValidateSymbolName(proto.name(), *full_name, proto);	

	  result->name_            = tables_->AllocateString(proto.name());
	  result->full_name_       = full_name;
	  result->file_            = file_;
	  result->containing_type_ = parent;
	  result->is_placeholder_  = false;
	  result->is_unqualified_placeholder_ = false;	

	  BUILD_ARRAY(proto, result, field          , BuildField         , result);
	  BUILD_ARRAY(proto, result, nested_type    , BuildMessage       , result);
	  BUILD_ARRAY(proto, result, enum_type      , BuildEnum          , result);
	  BUILD_ARRAY(proto, result, extension_range, BuildExtensionRange, result);
	  BUILD_ARRAY(proto, result, extension      , BuildExtension     , result);	

	  // Copy options.
	  if (!proto.has_options()) {
	    result->options_ = NULL;  // Will set to default_instance later.
	  } else {
	    AllocateOptions(proto.options(), result);
	  }	

	  AddSymbol(result->full_name(), parent, result->name(),
	            proto, Symbol(result));	

	  // Check that no fields have numbers in extension ranges.
	  for (int i = 0; i < result->field_count(); i++) {
	    const FieldDescriptor* field = result->field(i);
	    for (int j = 0; j < result->extension_range_count(); j++) {
	      const Descriptor::ExtensionRange* range = result->extension_range(j);
	      if (range->start <= field->number() && field->number() < range->end) {
	        AddError(field->full_name(), proto.extension_range(j),
	                 DescriptorPool::ErrorCollector::NUMBER,
	                 strings::Substitute(
	                   "Extension range $0 to $1 includes field \"$2\" ($3).",
	                   range->start, range->end - 1,
	                   field->name(), field->number()));
	      }
	    }
	  }	

	  // Check that extension ranges don't overlap.
	  for (int i = 0; i < result->extension_range_count(); i++) {
	    const Descriptor::ExtensionRange* range1 = result->extension_range(i);
	    for (int j = i + 1; j < result->extension_range_count(); j++) {
	      const Descriptor::ExtensionRange* range2 = result->extension_range(j);
	      if (range1->end > range2->start && range2->end > range1->start) {
	        AddError(result->full_name(), proto.extension_range(j),
	                 DescriptorPool::ErrorCollector::NUMBER,
	                 strings::Substitute("Extension range $0 to $1 overlaps with "
	                                     "already-defined range $2 to $3.",
	                                     range2->start, range2->end - 1,
	                                     range1->start, range1->end - 1));
	      }
	    }
	  }
	}

## BUILD_ARRAY

BUILD_ARRAY是将xxxDescriptor转化为xxxDescriptorProto的时候一个很通用的宏。定义在descriptor.cc line 3013:

	// A common pattern:  We want to convert a repeated field in the descriptor
	// to an array of values, calling some method to build each value.
	#define BUILD_ARRAY(INPUT, OUTPUT, NAME, METHOD, PARENT)             \
	  OUTPUT->NAME##_count_ = INPUT.NAME##_size();                       \
	  AllocateArray(INPUT.NAME##_size(), &OUTPUT->NAME##s_);             \
	  for (int i = 0; i < INPUT.NAME##_size(); i++) {                    \
	    METHOD(INPUT.NAME(i), PARENT, OUTPUT->NAME##s_ + i);             \
	  }

下面的宏：

	  BUILD_ARRAY(proto, result, message_type, BuildMessage  , NULL);

经过替换变成了：

	result->message_type_count = proto.message_type_size(); \
	AllocateArray(proto.message_type_size(),&result->message_type_s);\
	for (int i = 0; i < proto.message_type_size(); i++) {                    \
	    BuildMessage(proto.message_type(i), NULL, result->message_types_ + i);             \
	}

