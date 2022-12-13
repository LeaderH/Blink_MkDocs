# Apkinfo
Apkinfo is the core object of an instance of analysis on single target APK; it not only is in charge of manipulating all the communication between test rules defined in the evaluation module and fundamental elements gathered in the previous decompile module, but also has to collect and produce final reports.

|category|sample methods|info|
|:--:|:--:|:--:|
|metadata|	pkg_name, file_size, versionCode, md5,permissions…| APK’s basic information|
|resource|	get_all_strings, get_files, get_xml, find_methods…|locating target resources and objects|
|auxiliary|	isMemberInClass, methodMatches, parametersMatch…|helper methods for test rules|
|reporting|	set_reportor, report|collect reports from test rules|



|_.property_/ **method**|	parameters|	description|
|:--|:--|:--|
|.androguard_objects||		fallback to androguard analysis object|
|.all_files	|||
|.reportor	||	report collecting object|
|.filename	||	apk filename|
|.filesize  |||
|.dex_filesize		|||
|.md5		|||
|.permissions		|||
|.minSdk		|||
|.targetSdk		|||
|.versionName		|||
|.versionCode		|||
|.packagename		|||
|.all_files	||	iterate all files|
|.all_methods||		iterate all methods|
|get_file_read|	file_name	|get raw file|
|get_files	||	iterate all files|
|get_file	|file_name	|get FileObject|
|get_xml	xml_filename	|||
|get_android_manifest_xml		|||
|get_android_resources	||	ARSCParser|
|get_res_value	|ref_id	|resolve resource reference id (format @...)|
|get_all_strings		|||
|find_classes	|class_name, no_external, filter_regex|	find classes with class_name, can apply filters or exclude Android built-in APIs |
|find_methods	|class_name,method_name,descriptor,filter_regex	|find methods matching method_name, member class_name, and descriptor (parameter types)|
|find_references	|class_name,ref_name,ref_type	|find a static field with specified name, type and member class|
|methodMatches	|method, class_name, allow inherit, parameter_types|	check whether a method is a member of a class or subclass with class_name (allow inherit) and takes certain types of parameters|
|parametersMatch|	method, parameter_types	|check whether a method takes certain types of parameters|
|isMemberInClass	|method_field, class_name|	check whether a field or method is a member of a class with class_name|
|isMemberSubClassOf	|method_field, class_name, allow_inherit	|check whether a field or method is a member of a subclass(allow inherit)  with class_name|
|set_reporter|	reporter||
|report	|issue, method, message, location|	report an issue record |
