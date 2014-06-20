---
layout:	default
title:  Protobuf 初探
---

# Protobuf 初探

标签（空格分隔）： protobuf c++

---

## 安装

下载： [链接](https://code.google.com/p/protobuf/downloads/list)


编译安装：

     ./configure --prefix=/home/liwei12/Tool
     make
     make check
     make install
     这样protoc就安装在/home/liwei12/Tool/bin/protoc
     lib安装在/home/liwei12/Tool/lib/libprotobuf.a .so等等

运行example中的例子
    
    cd examples
    export PKG_CONFIG_PATH=/home/liwei12/Tool/lib/pkgconfig/
    在Makefile中加上-static，使用静态库链接。
    make cpp
    	protoc --cpp_out. --java_out=. --python_out=. addressbook.proto
    	c++ add_person.cc addressbook.pb.cc -static -o add_person_cpp `pkg-config --cflags --libs protobuf`
    	c++ list_person.cc addressbook.pb.cc -static -o list_person_cpp `pkg-config --cflags --libs protobuf`
    然后运行：
    ./add_person_cpp ADDRESS_BOOK_FILE
    ./list_person_cpp ADDRESS_BOOK_FILE

## 代码结构

### Makefile
先看根目录下的Makefile

	all: config.h
		$(MAKE) $(AM_MAKEFLAGS) all-recursice //这句话的含义是进入每一个子目录运行 `make all`

再看 src/Makefile
	
	all: $(BUILT_SOURCES)
		$(MAKE) $(AM_MAKEFLAGS) all-am
	all-am: Makefile $(LTLIBRARIES) $(PROGRAMS) $(DATA) $(HEADERS)
	PROGRAMS = $(bin_PROGRAMS)
	$(bin_PROGRAMS) = protoc$(EXEEXT)
	protoc$(EXEECT): $(protoc_OBJECTS) $(protoc_DEPENDENCIES) $(EXTRA_protoc_DEPENDENCIES)
		@rm -f protoc$(EXEEXT)
		$(CXXLINK) $(protoc_OBJECTS) $(proto_LDADD) $(LIBS)
	protoc_OBJECTS = $(am_protoc_OBJECTS)
	am_protoc_OBJECTS = main.$(OBJEXT)
	protoc_DEPENDENCIES = $(am__DEPENDENCIES_1) libprotobuf.la libprotoc.la
	libprotobuf.la: $(libprotobuf_la_OBJECTS) $(libprotobuf_la_DEPENDENCIES) $(EXTRA_libprotobuf_la_DEPENDENCIES) 
		$(libprotobuf_la_LINK) -rpath $(libdir) $(libprotobuf_la_OBJECTS) $(libprotobuf_la_LIBADD) $(LIBS)
	am_libprotobuf_la_OBJECTS = $(am__objects_1) strutil.lo substitute.lo \
        structurally_valid.lo descriptor.lo descriptor.pb.lo \
        descriptor_database.lo dynamic_message.lo \
        extension_set_heavy.lo generated_message_reflection.lo \
        message.lo reflection_ops.lo service.lo text_format.lo \
        unknown_field_set.lo wire_format.lo gzip_stream.lo printer.lo \
        tokenizer.lo zero_copy_stream_impl.lo importer.lo parser.lo
    libprotobuf_la_OBJECTS = $(am_libprotobuf_la_OBJECTS)
    libprotobuf_la_DEPENDENCIES = $(am__DEPENDENCIES_1)
	am__objects_1 = atomicops_internals_x86_gcc.lo \ //这个粒度基本是最基本的文件级别了
        atomicops_internals_x86_msvc.lo common.lo once.lo \
        stringprintf.lo extension_set.lo generated_message_util.lo \
        message_lite.lo repeated_field.lo wire_format_lite.lo \
        coded_stream.lo zero_copy_stream.lo \
        zero_copy_stream_impl_lite.lo
    libprotobuf_la_LINK = $(LIBTOOL) --tag=CXX $(AM_LIBTOOLFLAGS) \
        $(LIBTOOLFLAGS) --mode=link $(CXXLD) $(AM_CXXFLAGS) \
        $(CXXFLAGS) $(libprotobuf_la_LDFLAGS) $(LDFLAGS) -o $@
    parser.lo: google/protobuf/compiler/parser.cc
        $(LIBTOOL)  --tag=CXX $(AM_LIBTOOLFLAGS) $(LIBTOOLFLAGS) --mode=compile $(CXX) $(DEFS) $(DEFAULT_INCLUDES) $(INCLUDES) $(AM_CPPFLAGS) $(CPPFLAGS) $(AM_CXXFLAGS) $(CXXFLAGS) -MT parser.lo -MD -MP -MF $(DEPDIR)/parser.Tpo -c -o parser.lo `test -f 'google/protobuf/compiler/parser.cc' || echo '$(srcdir)/'`google/protobuf/compiler/parser.cc

通过上面的Makefile，基本上知道大概是怎么跑起来的了

### protoc

通过上面的Makefile分析，我们知道protoc可执行文件是由下列文件编成（通过看Makefile.am更直接）
才发现Makefile可读性很差，而Makefile.am可读性非常高。以后开到开源代码，先看Makefile.am的说。
	
	bin_PROGRAMS = protoc
	protoc_LDADD = $(PTHREAD_LIBS) libprotobuf.la libprotoc.la
	protoc_SOURCES = google/protobuf/compiler/main.cc

	//libprotobuf.la 与 libprotoc.la如下
	libprotobuf_la_LIBADD = $(PTHREAD_LIBS)
	libprotobuf_la_LDFLAGS = -version-info 8:0:0 -export-dynamic -no-undefined
	libprotobuf_la_SOURCES =                                       \
		$(libprotobuf_lite_la_SOURCES)                               \
		google/protobuf/stubs/strutil.cc                             \
		google/protobuf/stubs/strutil.h                              \
		google/protobuf/stubs/substitute.cc                          \
		google/protobuf/stubs/substitute.h                           \
		google/protobuf/stubs/structurally_valid.cc                  \
		google/protobuf/descriptor.cc                                \
		google/protobuf/descriptor.pb.cc                             \
		google/protobuf/descriptor_database.cc                       \
		google/protobuf/dynamic_message.cc                           \
		google/protobuf/extension_set_heavy.cc                       \
		google/protobuf/generated_message_reflection.cc              \
		google/protobuf/message.cc                                   \
		google/protobuf/reflection_ops.cc                            \
		google/protobuf/service.cc                                   \
		google/protobuf/text_format.cc                               \
		google/protobuf/unknown_field_set.cc                         \
		google/protobuf/wire_format.cc                               \
		google/protobuf/io/gzip_stream.cc                            \
		google/protobuf/io/printer.cc                                \
		google/protobuf/io/tokenizer.cc                              \
		google/protobuf/io/zero_copy_stream_impl.cc                  \
		google/protobuf/compiler/importer.cc                         \
		google/protobuf/compiler/parser.cc

	libprotobuf_lite_la_LIBADD = $(PTHREAD_LIBS)
	libprotobuf_lite_la_LDFLAGS = -version-info 8:0:0 -export-dynamic -no-undefined
	libprotobuf_lite_la_SOURCES =                                  \
		google/protobuf/stubs/atomicops_internals_x86_gcc.cc         \
		google/protobuf/stubs/atomicops_internals_x86_msvc.cc        \
		google/protobuf/stubs/common.cc                              \
		google/protobuf/stubs/once.cc                                \
		google/protobuf/stubs/hash.h                                 \
		google/protobuf/stubs/map-util.h                             \
		google/protobuf/stubs/stl_util.h                             \
		google/protobuf/stubs/stringprintf.cc                        \
		google/protobuf/stubs/stringprintf.h                         \
		google/protobuf/extension_set.cc                             \
		google/protobuf/generated_message_util.cc                    \
		google/protobuf/message_lite.cc                              \
		google/protobuf/repeated_field.cc                            \
		google/protobuf/wire_format_lite.cc                          \
		google/protobuf/io/coded_stream.cc                           \
		google/protobuf/io/coded_stream_inl.h                        \
		google/protobuf/io/zero_copy_stream.cc                       \
		google/protobuf/io/zero_copy_stream_impl_lite.cc

	libprotoc_la_LIBADD = $(PTHREAD_LIBS) libprotobuf.la
	libprotoc_la_LDFLAGS = -version-info 8:0:0 -export-dynamic -no-undefined
	libprotoc_la_SOURCES =                                         \
		google/protobuf/compiler/code_generator.cc                   \
		google/protobuf/compiler/command_line_interface.cc           \
		google/protobuf/compiler/plugin.cc                           \
		google/protobuf/compiler/plugin.pb.cc                        \
		google/protobuf/compiler/subprocess.cc                       \
		google/protobuf/compiler/subprocess.h                        \
		google/protobuf/compiler/zip_writer.cc                       \
		google/protobuf/compiler/zip_writer.h                        \
		google/protobuf/compiler/cpp/cpp_enum.cc                     \
		google/protobuf/compiler/cpp/cpp_enum.h                      \
		google/protobuf/compiler/cpp/cpp_enum_field.cc               \
		google/protobuf/compiler/cpp/cpp_enum_field.h                \
		google/protobuf/compiler/cpp/cpp_extension.cc                \
		google/protobuf/compiler/cpp/cpp_extension.h                 \
		google/protobuf/compiler/cpp/cpp_field.cc                    \
		google/protobuf/compiler/cpp/cpp_field.h                     \
		google/protobuf/compiler/cpp/cpp_file.cc                     \
		google/protobuf/compiler/cpp/cpp_file.h                      \
		google/protobuf/compiler/cpp/cpp_generator.cc                \
		google/protobuf/compiler/cpp/cpp_helpers.cc                  \
		google/protobuf/compiler/cpp/cpp_helpers.h                   \
		google/protobuf/compiler/cpp/cpp_message.cc                  \
		google/protobuf/compiler/cpp/cpp_message.h                   \
		google/protobuf/compiler/cpp/cpp_message_field.cc            \
		google/protobuf/compiler/cpp/cpp_message_field.h             \
		google/protobuf/compiler/cpp/cpp_options.h                   \
		google/protobuf/compiler/cpp/cpp_primitive_field.cc          \
		google/protobuf/compiler/cpp/cpp_primitive_field.h           \
		google/protobuf/compiler/cpp/cpp_service.cc                  \
		google/protobuf/compiler/cpp/cpp_service.h                   \
		google/protobuf/compiler/cpp/cpp_string_field.cc             \
		google/protobuf/compiler/cpp/cpp_string_field.h              \
		google/protobuf/compiler/java/java_enum.cc                   \
		google/protobuf/compiler/java/java_enum.h                    \
		google/protobuf/compiler/java/java_enum_field.cc             \
		google/protobuf/compiler/java/java_enum_field.h              \
		google/protobuf/compiler/java/java_extension.cc              \
		google/protobuf/compiler/java/java_extension.h               \
		google/protobuf/compiler/java/java_field.cc                  \
		google/protobuf/compiler/java/java_field.h                   \
		google/protobuf/compiler/java/java_file.cc                   \
		google/protobuf/compiler/java/java_file.h                    \
		google/protobuf/compiler/java/java_generator.cc              \
		google/protobuf/compiler/java/java_helpers.cc                \
		google/protobuf/compiler/java/java_helpers.h                 \
		google/protobuf/compiler/java/java_message.cc                \
		google/protobuf/compiler/java/java_message.h                 \
		google/protobuf/compiler/java/java_message_field.cc          \
		google/protobuf/compiler/java/java_message_field.h           \
		google/protobuf/compiler/java/java_primitive_field.cc        \
		google/protobuf/compiler/java/java_primitive_field.h         \
		google/protobuf/compiler/java/java_service.cc                \
		google/protobuf/compiler/java/java_service.h                 \
		google/protobuf/compiler/java/java_string_field.cc           \
		google/protobuf/compiler/java/java_string_field.h            \
		google/protobuf/compiler/java/java_doc_comment.cc            \
		google/protobuf/compiler/java/java_doc_comment.h             \
		google/protobuf/compiler/python/python_generator.cc

#### protoc: main Compiler相关


首先看主程：

	int main(){
		google::protobuf::compiler::CommandLineInterface cli;
		cli.AllowPlugins("protoc-");

		google::protobuf::compiler::cpp::CppGenerator cpp_generator;
		//cli注册cpp_out的generator
		cli.RegisterGenerator("--cpp_out","--cpp_opt",&cpp_generator,"Generate C++ header and source.");

		return cli.Run(argc, argv);
	}

看看CommandLineInterface：

	class CommandLineInterface {

		public :
			CommandLineInterface();
  			~CommandLineInterface();

  			void RegisterGenerator(const string& flag_name,
                         CodeGenerator* generator,
                         const string& help_text);
            void RegisterGenerator(const string& flag_name,
                         const string& option_flag_name,
                         CodeGenerator* generator,
                         const string& help_text);
            //plug-in
            void AllowPlugins(const string& exe_name_prefix);
            int Run(int argc, const char* const argv[]);

            void SetInputsAreProtoPathRelative(bool enable) {
    			inputs_are_proto_path_relative_ = enable;
  			}
  			void SetVersionInfo(const string& text) {
    			version_info_ = text;
  			}
	};

下面看看CommandLineInterface的Run方法：

	Run(){
		//1. 解析参数

		//2. 设置DiskSourceTree

		//3. foreach 解析proto文件生成每一个FileDescriptor parsed_files
		const FileDescriptor* parsed_file = importer.Import(input_files_[i]);

		//4. 如果是COMPILE模式，foreach每一个output_directives_,将所有parsed_files生成到对应的output_directories[output_location]
		GeneratorContextImpl** map_slot = &output_directories[output_location];
		*map_slot = new GeneratorContextImpl(parsed_files);
		GenerateOutput(parsed_files, output_directives_[i], *map_slot)// 内部调用output_directives_[i]中的generator来Generate(),生成的信息保存在map_slot中。

		//5. foreach output_directories，写磁盘
		const string& location = itr->first;
		GeneratorContextImpl* directory = iter->second;
		directory->WriteAllToDisk(location);
		
		//6. ENCODE或者DECODE模式
		EncodeOrDecode(&pool)
	}

Importer类 descriptor.h

	//构造参数
	Importer::Importer(SourceTree *source_tree, MultiFieldErrorCollector* error_collector)
	: database_(source_tree),
    pool_(&database_, database_.GetValidationErrorCollector()){
    	database_.RecordErrorsTo(error_collector);
	}
	//pool_ : DescriptorPool
	//database_: SourceTreeDescriptorDatabase

	const FileDescriptor* Importer::Import(const string& filename) {
  		return pool_.FindFileByName(filename); // 下面DescriptorPool有描述
	}


DescriptorPool类 descriptor.h
	
	explicit DescriptorPool(DescriptorDatabase* fallback_database,
                          ErrorCollector* error_collector = NULL);
    const FileDescriptor* FindFileByName(const string& name) const {
    	const FileDescriptor* result = tables_->FindFile(name);
  		if (result != NULL) return result;
  		if (underlay_ != NULL) {
    		result = underlay_->FindFileByName(name);
    		if (result != NULL) return result;
  		}
  		if (TryFindFileInFallbackDatabase(name)) {
    		result = tables_->FindFile(name);
    		if (result != NULL) return result;
  		}
  		return NULL;
    }

    underlay_ :DescriptorPool
    tables_: scoped_ptr<Tables>

    bool DescriptorPool::TryFindFileInFallbackDatabase(const string& name) const {
  		if (fallback_database_ == NULL) return false;

  		if (tables_->known_bad_files_.count(name) > 0) return false;

  		FileDescriptorProto file_proto;
  		if (!fallback_database_->FindFileByName(name, &file_proto) || //从fallback_database_中得到fileProto
      		BuildFileFromDatabase(file_proto) == NULL) { //该方法从fileProto中Build出来FileDescriptor
    		tables_->known_bad_files_.insert(name);
    		return false;
  		}

  		return true;
	}

	const FileDescriptor* DescriptorPool::BuildFileFromDatabase(const FileDescriptorProto& proto) const {
  		mutex_->AssertHeld();
  		return DescriptorBuilder(this, tables_.get(),
                           default_error_collector_).BuildFile(proto);
	}

DescriptorBuilder类 descriptor.cc

	public:
  		DescriptorBuilder(const DescriptorPool* pool,
                    DescriptorPool::Tables* tables,
                    DescriptorPool::ErrorCollector* error_collector);
  		~DescriptorBuilder();

  		//核心方法
  		const FileDescriptor* BuildFile(const FileDescriptorProto& proto){
  			//TODO 分析
  		}


Tables是DescriptorPool的内部类
	
	inline const FileDescriptor* DescriptorPool::Tables::FindFile(const string& key) const {
  		return FindPtrOrNull(files_by_name_, key.c_str());
	}
	private：
		SymbolsByNameMap      symbols_by_name_;
  		FilesByNameMap        files_by_name_;

SourceTreeDescriptorDatabase:public DescriptorDatabase importer.h

	SourceTreeDescriptorDatabase::SourceTreeDescriptorDatabase(SourceTree* source_tree)
  		: source_tree_(source_tree),
    	error_collector_(NULL),
    	using_validation_error_collector_(false),
    	validation_error_collector_(this) {}
	// implements DescriptorDatabase -----------------------------------
  	bool FindFileByName(const string& filename, FileDescriptorProto* output);


DiskSourceTree:public SourceTree importer.h



下面是CLI支持的参数

	CommandLineInterface::InterpretArgument(const string& name, const string& value) {
		name为空，value是文件名
			input_files_.push_back(value);
		-I --proto_path
			proto_path_.push_back(pair<string,string>(virtual_path, disk_path))
		-o --descriptor_set_out
			descriptor_set_name_ = value;
		--include_imports
			imports_in_descriptor_set_ = true;
		--include_source_info
			source_info_in_descriptor_set_ = true;
		-h --help
			PrintHelpText();
		--version
		--disallow_services
			disallow_services_ = true;
		--encode --decode --decode_row
			mode_ = (name == "--encode") ? MODE_ENCODE : MODE_DECODE;
			codec_type_ = value;
		--error_format
			error_format = ERROR_FORMAT_GCC ERROR_FORMAT_MSVS
		--plugin
			plugins_[plugin_name] = path;
		其他_out _opt flag
			generators_by_flag_name_
			generators_by_option_name_
			generator_parameters_
			output_directives_
			
	}

看看CppGenerator：

	class LIBPROTOC_EXPORT CppGenerator : public CodeGenerator {
		public:
		 CppGenerator();
		 ~CppGenerator();

		 // implements CodeGenerator ----------------------------------------
		 bool Generate(const FileDescriptor* file,
		               const string& parameter,
		               GeneratorContext* generator_context,
		               string* error) const;

		private:
		 GOOGLE_DISALLOW_EVIL_CONSTRUCTORS(CppGenerator);
	};

	CommangLineInterface  ------>> CodeGenerator
									/		\
								   /		 \
							CppGenerator   JavaGenerator
	
上面的类中有一个宏GOOGLE_DISALLOW_EVIL_CONSTRUCTORS，定义在src/google/protobuf/stubs/common.h中

	#undef GOOGLE_DISALLOW_EVIL_CONSTRUCTORS
	#define GOOGLE_DISALLOW_EVIL_CONSTRUCTORS(TypeName)    \
  	TypeName(const TypeName&);                           \
  	void operator=(const TypeName&)


#### protoc: libprotobuf.lo

这部分主要包括Descriptor，FileDescriptor，DescriptorProto等等，还是很有意思的。
TODO


#### protoc: libprotobuf-lite.lo

#### protoc: libprotoc.lo

## 语法

下面所有的内容，基本都可以在[官方文档](https://developers.google.com/protocol-buffers/docs/overview)里找到。

### 字段类型

### proto文件定义

1. required 每个标准消息必须要有一个
2. optional 消息格式中该字段可以有0个或1个。
3. repeated 可以有0-n个


## 编解码

有如下几类编解码方式：

编码 | 含义 | 适用类型
-----|------|----
0 | Variant  | int32, int64, uint32, uint64, sint32, sint64, bool, enum
1 | 64-bit  | fixed64, sfixed64, double
2 | Length-delimited | string, bytes, embedded messages, packed repeated fields
5 | 32-bit | fixed32, sfixed32, float

### Key的表示

    (id << 3) | wire_type
    
### Variantb 编码

该变长编码是基于128bit的变长编码。就是将正常二进制表示从低到高切分为7个一组，每一组第八位置0或1，1表示后面还有，0表示后面没有了。

整数 300 使用变长编码 ->1010 1100 0000 0010 现在分析一下二进制串,首先要了解变 长整数对于每个字节的编码都有一个最高有效位,如果为 1,表示后面还有字节; 如果为 0,表示后面没有字节。 这样把每个字节的第一位去掉,变成 010 1100 000 0010,变长整数采用的是小端编码,所以倒转一下字符串变成 000 0010 010 1100=300。


### sint32 sint64表示

对于 sint32 来说,采用 (n << 1)^(n >> 31)此方式编码，
对于 sint64 来说,采用 (n << 1)^(n >> 63)此方式编码

### length-delimited编码方法

顾名思义，就是先写入值的长度，再写值。有点类似于sequenceFile的处理方式。

## 其他



