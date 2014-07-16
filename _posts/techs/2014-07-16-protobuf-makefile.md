---
layout: default
title:  protobuf中的MakeFile
---

本文根据protobuf的Makefile，看下如何从Makefile.am最终生成Makefile，以及是怎么做的？

# 目录结构

下载后的目录结构如下所示（只显示了三级子目录）：

    ├── aclocal.m4
    ├── autogen.sh
    ├── CHANGES.txt
    ├── config.guess
    ├── config.h.in
    ├── config.sub
    ├── configure
    ├── configure.ac
    ├── CONTRIBUTORS.txt
    ├── COPYING.txt
    ├── depcomp
    ├── editors
    │   ├── protobuf-mode.el
    │   ├── proto.vim
    │   └── README.txt
    ├── examples
    │   ├── ...
    ├── generate_descriptor_proto.sh
    ├── gtest
    │   ├── aclocal.m4
    │   ├── ...
    ├── install-sh
    ├── INSTALL.txt
    ├── java
    │   ├── pom.xml
    │   ├── README.txt
    │   └── src
    ├── ltmain.sh
    ├── m4
    │   ├── ac_system_extensions.m4
    │   ├── acx_check_suncc.m4
    │   ├── acx_pthread.m4
    │   ├── libtool.m4
    │   ├── lt~obsolete.m4
    │   ├── ltoptions.m4
    │   ├── ltsugar.m4
    │   ├── ltversion.m4
    │   └── stl_hash.m4
    ├── Makefile.am
    ├── Makefile.in
    ├── missing
    ├── protobuf-lite.pc.in
    ├── protobuf.pc.in
    ├── python
    │   ├── ez_setup.py
    │   ├── google
    │   ├── mox.py
    │   ├── README.txt
    │   ├── setup.py
    │   └── stubout.py
    ├── README.txt
    ├── src
    │   ├── google
    │   ├── Makefile.am
    │   ├── Makefile.in
    │   └── solaris
    └── vsprojects

# 编译

    ./configure --prefix=/home/users/liwei12/Tool

经过这一步骤后，发生变化的目录有(`find . -type d -mtime -1`)：

    .
    ./gtest
    ./gtest/build-aux
    ./gtest/fused-src/gtest
    ./gtest/fused-src/gtest/.deps
    ./gtest/samples
    ./gtest/samples/.deps
    ./gtest/scripts
    ./gtest/src
    ./gtest/src/.deps
    ./gtest/test
    ./gtest/test/.deps
    ./src
    ./src/.deps

经过这一步骤后，发生变化的文件有(`find . -type f -mtime -1`)：

    ./gtest/build-aux/config.h
    ./gtest/build-aux/stamp-h1
    ./gtest/fused-src/gtest/.deps/test_fused_gtest_test-gtest-all.Po
    ./gtest/fused-src/gtest/.deps/test_fused_gtest_test-gtest_main.Po
    ./gtest/samples/.deps/sample1.Plo
    ./gtest/samples/.deps/sample10_unittest.Po
    ./gtest/samples/.deps/sample1_unittest.Po
    ./gtest/samples/.deps/sample2.Plo
    ./gtest/samples/.deps/sample4.Plo
    ./gtest/samples/.deps/test_fused_gtest_test-sample1.Po
    ./gtest/samples/.deps/test_fused_gtest_test-sample1_unittest.Po
    ./gtest/scripts/gtest-config
    ./gtest/src/.deps/gtest-all.Plo
    ./gtest/src/.deps/gtest_main.Plo
    ./gtest/test/.deps/gtest_all_test.Po
    ./gtest/config.log
    ./gtest/config.status
    ./gtest/Makefile
    ./gtest/libtool
    ./src/Makefile
    ./src/.deps/atomicops_internals_x86_gcc.Plo
    ./src/.deps/atomicops_internals_x86_msvc.Plo
    ./src/.deps/code_generator.Plo
    ./src/.deps/coded_stream.Plo
    ./src/.deps/command_line_interface.Plo
    ./src/.deps/common.Plo
    ./src/.deps/cpp_enum.Plo
    ./src/.deps/cpp_enum_field.Plo
    ./src/.deps/cpp_extension.Plo
    ./src/.deps/cpp_field.Plo
    ./src/.deps/cpp_file.Plo
    ./src/.deps/cpp_generator.Plo
    ./src/.deps/cpp_helpers.Plo
    ./src/.deps/cpp_message.Plo
    ./src/.deps/cpp_message_field.Plo
    ./src/.deps/cpp_primitive_field.Plo
    ./src/.deps/cpp_service.Plo
    ./src/.deps/cpp_string_field.Plo
    ./src/.deps/descriptor.Plo
    ./src/.deps/descriptor.pb.Plo
    ./src/.deps/descriptor_database.Plo
    ./src/.deps/dynamic_message.Plo
    ./src/.deps/extension_set.Plo
    ./src/.deps/extension_set_heavy.Plo
    ./src/.deps/generated_message_reflection.Plo
    ./src/.deps/generated_message_util.Plo
    ./src/.deps/gzip_stream.Plo
    ./src/.deps/importer.Plo
    ./src/.deps/java_doc_comment.Plo
    ./src/.deps/java_enum.Plo
    ./src/.deps/java_enum_field.Plo
    ./src/.deps/java_extension.Plo
    ./src/.deps/java_field.Plo
    ./src/.deps/java_file.Plo
    ./src/.deps/java_generator.Plo
    ./src/.deps/java_helpers.Plo
    ./src/.deps/java_message.Plo
    ./src/.deps/java_message_field.Plo
    ./src/.deps/java_primitive_field.Plo
    ./src/.deps/java_service.Plo
    ./src/.deps/java_string_field.Plo
    ./src/.deps/main.Po
    ./src/.deps/message.Plo
    ./src/.deps/message_lite.Plo
    ./src/.deps/once.Plo
    ./src/.deps/parser.Plo
    ./src/.deps/plugin.Plo
    ./src/.deps/plugin.pb.Plo
    ./src/.deps/printer.Plo
    ./src/.deps/protobuf_lazy_descriptor_test-cpp_test_bad_identifiers.pb.Po
    ./src/.deps/protobuf_lazy_descriptor_test-cpp_unittest.Po
    ./src/.deps/protobuf_lazy_descriptor_test-file.Po
    ./src/.deps/protobuf_lazy_descriptor_test-googletest.Po
    ./src/.deps/protobuf_lazy_descriptor_test-test_util.Po
    ./src/.deps/protobuf_lazy_descriptor_test-unittest.pb.Po
    ./src/.deps/protobuf_lazy_descriptor_test-unittest_custom_options.pb.Po
    ./src/.deps/protobuf_lazy_descriptor_test-unittest_embed_optimize_for.pb.Po
    ./src/.deps/protobuf_lazy_descriptor_test-unittest_empty.pb.Po
    ./src/.deps/protobuf_lazy_descriptor_test-unittest_import.pb.Po
    ./src/.deps/protobuf_lazy_descriptor_test-unittest_import_lite.pb.Po
    ./src/.deps/protobuf_lazy_descriptor_test-unittest_import_public.pb.Po
    ./src/.deps/protobuf_lazy_descriptor_test-unittest_import_public_lite.pb.Po
    ./src/.deps/protobuf_lazy_descriptor_test-unittest_lite.pb.Po
    ./src/.deps/protobuf_lazy_descriptor_test-unittest_lite_imports_nonlite.pb.Po
    ./src/.deps/protobuf_lazy_descriptor_test-unittest_mset.pb.Po
    ./src/.deps/protobuf_lazy_descriptor_test-unittest_no_generic_services.pb.Po
    ./src/.deps/protobuf_lazy_descriptor_test-unittest_optimize_for.pb.Po
    ./src/.deps/protobuf_lite_test-lite_unittest.Po
    ./src/.deps/protobuf_lite_test-test_util_lite.Po
    ./src/.deps/protobuf_lite_test-unittest_import_lite.pb.Po
    ./src/.deps/protobuf_lite_test-unittest_import_public_lite.pb.Po
    ./src/.deps/protobuf_lite_test-unittest_lite.pb.Po
    ./src/.deps/protobuf_test-coded_stream_unittest.Po
    ./src/.deps/protobuf_test-command_line_interface_unittest.Po
    ./src/.deps/protobuf_test-common_unittest.Po
    ./src/.deps/protobuf_test-cpp_bootstrap_unittest.Po
    ./src/.deps/protobuf_test-cpp_plugin_unittest.Po
    ./src/.deps/protobuf_test-cpp_test_bad_identifiers.pb.Po
    ./src/.deps/protobuf_test-cpp_unittest.Po
    ./src/.deps/protobuf_test-descriptor_database_unittest.Po
    ./src/.deps/protobuf_test-descriptor_unittest.Po
    ./src/.deps/protobuf_test-dynamic_message_unittest.Po
    ./src/.deps/protobuf_test-extension_set_unittest.Po
    ./src/.deps/protobuf_test-file.Po
    ./src/.deps/protobuf_test-generated_message_reflection_unittest.Po
    ./src/.deps/protobuf_test-googletest.Po
    ./src/.deps/protobuf_test-importer_unittest.Po
    ./src/.deps/protobuf_test-java_doc_comment_unittest.Po
    ./src/.deps/protobuf_test-java_plugin_unittest.Po
    ./src/.deps/protobuf_test-message_unittest.Po
    ./src/.deps/protobuf_test-mock_code_generator.Po
    ./src/.deps/protobuf_test-once_unittest.Po
    ./src/.deps/protobuf_test-parser_unittest.Po
    ./src/.deps/protobuf_test-printer_unittest.Po
    ./src/.deps/protobuf_test-python_plugin_unittest.Po
    ./src/.deps/protobuf_test-reflection_ops_unittest.Po
    ./src/.deps/protobuf_test-repeated_field_reflection_unittest.Po
    ./src/.deps/protobuf_test-repeated_field_unittest.Po
    ./src/.deps/protobuf_test-stringprintf_unittest.Po
    ./src/.deps/protobuf_test-structurally_valid_unittest.Po
    ./src/.deps/protobuf_test-strutil_unittest.Po
    ./src/.deps/service.Plo
    ./src/.deps/protobuf_test-template_util_unittest.Po
    ./src/.deps/protobuf_test-test_util.Po
    ./src/.deps/protobuf_test-text_format_unittest.Po
    ./src/.deps/protobuf_test-tokenizer_unittest.Po
    ./src/.deps/protobuf_test-type_traits_unittest.Po
    ./src/.deps/protobuf_test-unittest.pb.Po
    ./src/.deps/protobuf_test-unittest_custom_options.pb.Po
    ./src/.deps/protobuf_test-unittest_embed_optimize_for.pb.Po
    ./src/.deps/protobuf_test-unittest_empty.pb.Po
    ./src/.deps/protobuf_test-unittest_import.pb.Po
    ./src/.deps/protobuf_test-unittest_import_lite.pb.Po
    ./src/.deps/protobuf_test-unittest_import_public.pb.Po
    ./src/.deps/protobuf_test-unittest_import_public_lite.pb.Po
    ./src/.deps/protobuf_test-unittest_lite.pb.Po
    ./src/.deps/protobuf_test-unittest_lite_imports_nonlite.pb.Po
    ./src/.deps/protobuf_test-unittest_mset.pb.Po
    ./src/.deps/protobuf_test-unittest_no_generic_services.pb.Po
    ./src/.deps/protobuf_test-unittest_optimize_for.pb.Po
    ./src/.deps/protobuf_test-unknown_field_set_unittest.Po
    ./src/.deps/protobuf_test-wire_format_unittest.Po
    ./src/.deps/protobuf_test-zero_copy_stream_unittest.Po
    ./src/.deps/python_generator.Plo
    ./src/.deps/reflection_ops.Plo
    ./src/.deps/repeated_field.Plo
    ./src/.deps/stringprintf.Plo
    ./src/.deps/structurally_valid.Plo
    ./src/.deps/strutil.Plo
    ./src/.deps/subprocess.Plo
    ./src/.deps/substitute.Plo
    ./src/.deps/test_plugin-file.Po
    ./src/.deps/test_plugin-mock_code_generator.Po
    ./src/.deps/test_plugin-test_plugin.Po
    ./src/.deps/text_format.Plo
    ./src/.deps/tokenizer.Plo
    ./src/.deps/unknown_field_set.Plo
    ./src/.deps/wire_format.Plo
    ./src/.deps/wire_format_lite.Plo
    ./src/.deps/zcgunzip.Po
    ./src/.deps/zcgzip.Po
    ./src/.deps/zero_copy_stream.Plo
    ./src/.deps/zero_copy_stream_impl.Plo
    ./src/.deps/zero_copy_stream_impl_lite.Plo
    ./src/.deps/zip_writer.Plo
    ./config.log
    ./config.status
    ./Makefile
    ./protobuf.pc
    ./protobuf-lite.pc
    ./config.h
    ./stamp-h1
    ./libtool

configure主要做的事情有：

生成config.h,config.status,Makefile,protobuf.pc,protobuf-lite.pc

config.h.in--                       --> config.h
             +----> config.status -+
Makefile.in--                       --> Makefile

看下configure.ac中的内容：

    AC_CONFIG_SUBDIRS([gtest])
    AC_CONFIG_FILES([Makefile src/Makefile protobuf.pc protobuf-lite.pc])

就是这二行决定了./configure 会去生成上面的5个文件。而第一行会让configure去gtest中重新执行一遍./configure。而众多的Plo文件就是configure生成的，每个编译源文件都会有一个.Plo文件。

编译的主要单位是Makefile，用户手工编写Makefile.am，这个文件可读性较高，然后configure会生成Makefile。

# Autoconf和Automake，自动生成Makefile

这里还是系统的讲一下上面的Makefile、configure都是怎么回事吧。

TODO