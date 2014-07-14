---
layout: default
title:  cpp代码的自动生成
---

接着上面说的，本篇讲讲一个FileDescriptor是如何变成c++文件的。

# 执行流程

1. 对于每个输出目录，调用generator根据filedescriptor生成内容到GeneratorContext中
2. 对于每个输出目录，调用GeneratorContextImpl的WriteAllToDisk(location)方法，写文件。

先看command_line_interface.cc中的Run()方法中：
	
	  // We construct a separate GeneratorContext for each output location.  Note
	  // that two code generators may output to the same location, in which case
	  // they should share a single GeneratorContext so that OpenForInsert() works.
	  typedef hash_map<string, GeneratorContextImpl*> GeneratorContextMap;
	  GeneratorContextMap output_directories;	

	  // Generate output.
	  if (mode_ == MODE_COMPILE) {
	    for (int i = 0; i < output_directives_.size(); i++) {
	      string output_location = output_directives_[i].output_location;
	      if (!HasSuffixString(output_location, ".zip") &&
	          !HasSuffixString(output_location, ".jar")) {
	        AddTrailingSlash(&output_location);
	      }
	      GeneratorContextImpl** map_slot = &output_directories[output_location];	

	      if (*map_slot == NULL) {
	        // First time we've seen this output location.
	        *map_slot = new GeneratorContextImpl(parsed_files);
	      }	

	      if (!GenerateOutput(parsed_files, output_directives_[i], *map_slot)) {
	        STLDeleteValues(&output_directories);
	        return 1;
	      }
	    }
	  }

	  // Write all output to disk.
 	  for (GeneratorContextMap::iterator iter = output_directories.begin();
	       iter != output_directories.end(); ++iter) {
	    const string& location = iter->first;
	    GeneratorContextImpl* directory = iter->second;
	    if (HasSuffixString(location, "/")) {
	      if (!directory->WriteAllToDisk(location)) {
	        STLDeleteValues(&output_directories);
	        return 1;
	      }
	    } else {
	      if (HasSuffixString(location, ".jar")) {
	        directory->AddJarManifest();
	      }	

	      if (!directory->WriteAllToZip(location)) {
	        STLDeleteValues(&output_directories);
	        return 1;
	      }
	    }
	  }	

	  STLDeleteValues(&output_directories);

其中output_directives_定义如下：
	
	// output_directives_ lists all the files we are supposed to output and what
	// generator to use for each.
	struct OutputDirective {
	  string name;                // E.g. "--foo_out"
	  CodeGenerator* generator;   // NULL for plugins
	  string parameter;
	  string output_location;
	};
	vector<OutputDirective> output_directives	

看下GenerateOutput方法，在command_line_interface.cc中：

	bool CommandLineInterface::GenerateOutput(
	    const vector<const FileDescriptor*>& parsed_files,
	    const OutputDirective& output_directive,
	    GeneratorContext* generator_context) {
	  // Call the generator.
	  string error;
	  if (output_directive.generator == NULL) {
	    // This is a plugin.
	    GOOGLE_CHECK(HasPrefixString(output_directive.name, "--") &&
	          HasSuffixString(output_directive.name, "_out"))
	        << "Bad name for plugin generator: " << output_directive.name;	

	    // Strip the "--" and "_out" and add the plugin prefix.
	    string plugin_name = plugin_prefix_ + "gen-" +
	        output_directive.name.substr(2, output_directive.name.size() - 6);	

	    if (!GeneratePluginOutput(parsed_files, plugin_name,
	                              output_directive.parameter,
	                              generator_context, &error)) {
	      cerr << output_directive.name << ": " << error << endl;
	      return false;
	    }
	  } else {
	    // Regular generator.
	    string parameters = output_directive.parameter;
	    if (!generator_parameters_[output_directive.name].empty()) {
	      if (!parameters.empty()) {
	        parameters.append(",");
	      }
	      parameters.append(generator_parameters_[output_directive.name]);
	    }
	    for (int i = 0; i < parsed_files.size(); i++) {
	      if (!output_directive.generator->Generate(parsed_files[i], parameters,
	                                                generator_context, &error)) {
	        // Generator returned an error.
	        cerr << output_directive.name << ": " << parsed_files[i]->name() << ": "
	             << error << endl;
	        return false;
	      }
	    }
	  }	

	  return true;	
	}

主要是这个方法:

	output_directive.generator->Generate(parsed_files[i], parameters,
                                                generator_context, &error)

# CppGenerator


## Genrate方法

看下CppGenerator的Generate方法：cpp/cpp_generator.cc line 54

	  string basename = StripProto(file->name());
	  basename.append(".pb");	

	  FileGenerator file_generator(file, file_options);	

	  // Generate header.
	  {
	    scoped_ptr<io::ZeroCopyOutputStream> output(
	      generator_context->Open(basename + ".h"));
	    io::Printer printer(output.get(), '$');
	    file_generator.GenerateHeader(&printer);
	  }	

	  // Generate cc file.
	  {
	    scoped_ptr<io::ZeroCopyOutputStream> output(
	      generator_context->Open(basename + ".cc"));
	    io::Printer printer(output.get(), '$');
	    file_generator.GenerateSource(&printer);
	  }	

	  return true;

## FileGenerator

上面的FileGenerator定义在cpp/cpp_file.h中，这个类是生成文件的内容的类。

GenerateHeader 在cpp_file.cc line 91

	1. 首先输出头文件GUARD
	2. 输出版本声明 版本号相关的信息在src/google/protobuf/stubs/common.h
	3. include其他相关的头文件
	4. namespace，前向声明一些类似“protobuf_AddDesc_...的方法”
	5. 前向声明Proto中包含的类 message_generators_[i]->GenerateForwardDeclaration(printer);
	6. 生成enum定义代码，类定义代码，service定义代码，extension定义代码，类的inline定义代码。关闭namespace
		+ enum_generators_[i]->GenerateDefinition(printer);
		+ message_generators_[i]->GenerateClassDefinition(printer);
		+ service_generators_[i]->GenerateDeclarations(printer);
		+ extension_generators_[i]->GenerateDeclaration(printer);
		+ message_generators_[i]->GenerateInlineMethods(printer);

FileGenerator中有一系列Generator，用以自动化生成代码：
	
	scoped_array<scoped_ptr<MessageGenerator> > message_generators_;
	scoped_array<scoped_ptr<EnumGenerator> > enum_generators_;
	scoped_array<scoped_ptr<ServiceGenerator> > service_generators_;
	scoped_array<scoped_ptr<ExtensionGenerator> > extension_generators_;

GenerateSource 在cpp_file.cc line 300

## MessageGenerator

这个类用于生成Message相关的cpp代码：看一个方法GenerateClassDefinition:
	
	void MessageGenerator::
	GenerateClassDefinition(io::Printer* printer) {
		//1. 首先声明嵌套的类
	  for (int i = 0; i < descriptor_->nested_type_count(); i++) {
	    nested_generators_[i]->GenerateClassDefinition(printer);
	    printer->Print("\n");
	    printer->Print(kThinSeparator);
	    printer->Print("\n");
	  }	
	  	// 2. 类名，构造析构，拷贝构造函数，赋值操作符

	  	// 3. CopyFrom、MergeFrom、Clear、IsInitialized、ByteSize等

	  	// 4. FileAccssor
	  	GenerateFieldAccessorDeclarations(printer);

	  	// 5.  private域，为了最小化paddig，三段：
	  		 8 bytes对齐的域 _extensions_ _unknown_fields_
	  		 不定的 Field private声明
	  		 4 bytes对齐的
	  		 	mutable int _cached_size_;
	  		 	::google::protobuf::uint32 _has_bits_[($field_count$ + 31) / 32];
	  		 	::google::protobuf::uint32 _has_bits_[1];

	  	// 6. AddDescriptors、BuildDescriptors、ShutdownFile 变为友元。 		 	

	  	// 7. static 方法 
	  		InitAsDefaultInstance()
	  		default_instance_

	}









# GeneratorContextImpl

看下GeneratorContext 在code_generator.h中

	class LIBPROTOC_EXPORT GeneratorContext {
 	public:
	  inline GeneratorContext() {}
	  virtual ~GeneratorContext();	

	  // Opens the given file, truncating it if it exists, and returns a
	  // ZeroCopyOutputStream that writes to the file.  The caller takes ownership
	  // of the returned object.  This method never fails (a dummy stream will be
	  // returned instead).
	  //
	  // The filename given should be relative to the root of the source tree.
	  // E.g. the C++ generator, when generating code for "foo/bar.proto", will
	  // generate the files "foo/bar.pb.h" and "foo/bar.pb.cc"; note that
	  // "foo/" is included in these filenames.  The filename is not allowed to
	  // contain "." or ".." components.
	  virtual io::ZeroCopyOutputStream* Open(const string& filename) = 0;	

	  // Creates a ZeroCopyOutputStream which will insert code into the given file
	  // at the given insertion point.  See plugin.proto (plugin.pb.h) for more
	  // information on insertion points.  The default implementation
	  // assert-fails -- it exists only for backwards-compatibility.
	  //
	  // WARNING:  This feature is currently EXPERIMENTAL and is subject to change.
	  virtual io::ZeroCopyOutputStream* OpenForInsert(
	      const string& filename, const string& insertion_point);	

	  // Returns a vector of FileDescriptors for all the files being compiled
	  // in this run.  Useful for languages, such as Go, that treat files
	  // differently when compiled as a set rather than individually.
	  virtual void ListParsedFiles(vector<const FileDescriptor*>* output);	

	 private:
	  GOOGLE_DISALLOW_EVIL_CONSTRUCTORS(GeneratorContext);
	};

GeneratorContextImpl是CommandLineInterface的内部类，定义在command_line_interface.cc line 236

	// A GeneratorContext implementation that buffers files in memory, then dumps
	// them all to disk on demand.
	class CommandLineInterface::GeneratorContextImpl : public GeneratorContext {
	 public:
	  GeneratorContextImpl(const vector<const FileDescriptor*>& parsed_files);
	  ~GeneratorContextImpl();	

	  // Write all files in the directory to disk at the given output location,
	  // which must end in a '/'.
	  bool WriteAllToDisk(const string& prefix);	

	  // Write the contents of this directory to a ZIP-format archive with the
	  // given name.
	  bool WriteAllToZip(const string& filename);	

	  // Add a boilerplate META-INF/MANIFEST.MF file as required by the Java JAR
	  // format, unless one has already been written.
	  void AddJarManifest();	

	  // implements GeneratorContext --------------------------------------
	  io::ZeroCopyOutputStream* Open(const string& filename);
	  io::ZeroCopyOutputStream* OpenForInsert(
	      const string& filename, const string& insertion_point);
	  void ListParsedFiles(vector<const FileDescriptor*>* output) {
	    *output = parsed_files_;
	  }	

	 private:
	  friend class MemoryOutputStream;	

	  // map instead of hash_map so that files are written in order (good when
	  // writing zips).
	  map<string, string*> files_;
	  const vector<const FileDescriptor*>& parsed_files_;
	  bool had_error_;
	};


# 总结

至此，一个hello.proto用户自定义文件，是如何变成hello.pb.h和hello.pb.cc的，就已经水落石出了，下面看看PB的动态编译。