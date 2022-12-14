# Scanner Interfaces  
Scanner Interfaces are predefined analysis workflow with sets of query and call back methods to be overriden/implemented by Detectors.

![img](img/Detector%20implementing%20Scanner%20Interfaces.png "Detectors and Scanners")

## How do ScannerInterfaces works?
The main logic of ScannerInterface are defined within RuleManager(blink.Evaluate.rule_manager.py)  

A RuleManager will go through following steps when evaluting a Detector implementing a ScannerInterface:  

1.  load the detector class with **importlib**
    (line 60)
1. check if a class implement certain ScannerInterface via class's `getImplementationSequence()` method, the method needs to be predefined by ScannerInterfaces for returning itself at minimum. The method can also be overriden when a Detector implements multiple ScannerInterfaces and need to specify processing order.
    (line 139)


    ```py title="rule_manager.py" linenums="139"
    def run_single(self, rule_cls: BaseScanner):
        self.runCommonBefore(rule_cls)
        for implement in rule_cls.getImplementationSequence():
            try:
                if implement == JavaScanner:
                    self.runJavaRule(rule_cls)
                elif implement == ResourcesScanner:
                    self.runResourceRule(rule_cls)
                elif implement == OtherFileScanner:
                    self.runOtherFileRule(rule_cls)
                elif implement == XmlScanner:
                    self.runXmlRule(rule_cls)
                else:
                    raise NotImplementedError(
                        str(f'Scanner: {implement} unhandled'))
            except Exception as e:
                tqdm.write(traceback.format_exc())
                tqdm.write("continue next...")
                continue
            finally:
                pass
    ```

1. define scanning process  
    For example, the following is the code snippet within runJavaRule where(line 170) where we define the procedure and sequence of methods for accessing information from Apkinfo.  

    1. Gather target methods' name via Detectors' `getApplicableMethodNames()` 
    1. Locating target with APIs of Apkinfo, like `.all_methods`/`find_methods()`
    1. Retrieving MethodInstances (context sensitive objects) from MethodObjects
    1. Propagate the MethodInstances back to Detectors by invoking their call back `visitMethodCall()`

    ```py title="rule_manager.py" linenums="203"
    if rule_cls.getApplicableMethodNames():
        applicable = rule_cls.getApplicableMethodNames()
        if type(applicable) == tuple and applicable[0] == ALL_METHOD:
            filter_func = applicable[1]
            for method in self.apkinfo.all_methods:
                if not filter_func(method):
                    continue
                for instance in method.getMethodInstances(max_depth=MAX_METHODINSTANCE_TRACE_DEPTH, filter_regex=FILTER_CLASS_REGEXS):
                    rule_cls.visitMethodCall(self.apkinfo, instance)
                    del instance
                del method
        else:
            for target_method in applicable:
                for method in self.apkinfo.find_methods(method_name=target_method, filter_regex=FILTER_CLASS_REGEXS):
                    for instance in method.getMethodInstances(max_depth=MAX_METHODINSTANCE_TRACE_DEPTH, filter_regex=FILTER_CLASS_REGEXS):
                        rule_cls.visitMethodCall(self.apkinfo, instance)
                        del instance
                    del method
    ```

    Another example is the implementation of XmlScanner in method runXmlRule.  

    1. Gather target xmls' filename via Detectors' `getApplicableXmlFiles()`
    1. Propagate the the xml document with callback `visitDocument()`
    1. Gather target xml Element with element name via Detectors' `getApplicableElements()`
    1. Propagate the the xml element with callback `visitElement()`
    1. Gather target xml Attributes with Attributes name via Detectors' `getApplicableAttributes()`
    1. Propagate the the xml element with callback `visitAttribute()`

    ```py title="rule_manager.py" linenums="254"
    def runXmlRule(self, rule_cls: XmlScanner):
        file_scope = rule_cls.getApplicableXmlFiles()
        # skip searching part if we only consider AndroidManifest
        if len(file_scope) == 1 and file_scope[0] == Scope.MANIFEST:
            xmlFiles = [
                self.apkinfo.get_android_manifest_xml()]
        else:
            xmlFiles = self.apkinfo.get_scoped_files(
                file_scope, rule_cls.appliesTo)

        for xmlFile in xmlFiles:
            if xmlFile is None:
                continue
            rule_cls.visitDocument(self.apkinfo, xmlFile)
            applicable = rule_cls.getApplicableElements()
            if len(applicable) == 0:
                if len(rule_cls.getApplicableAttributes()) > 0:
                    for element in xmlFile.findall(".//"):
                        #rule_cls.visitElement(self.apkinfo, element)
                        for attr in rule_cls.getApplicableAttributes():
                            for attr_node in element.getLocalAttributeNodes(attr):
                                rule_cls.visitAttribute(
                                    self.apkinfo, attr_node)
                                del attr_node
            else:
                for element_tag in applicable:
                    for element in xmlFile.getElementsByTagName(element_tag):
                        rule_cls.visitElement(self.apkinfo, element)
                        for attr in rule_cls.getApplicableAttributes():
                            for attr_node in element.getLocalAttributeNodes(attr):
                                rule_cls.visitAttribute(
                                    self.apkinfo, attr_node)
                                del attr_node
    ```

    A Detector implementing above ScannerInterface would have communication sequence show as follows:

    ![img](img/Example%20communications%20between%20Detector%20RuleManager%20and%20apkinfo.png "Interaction between Detector and RuleManager when evaluated as XmlScanner")



### BaseScanner
|query / callback	|parameters	|description|
|:--|:--|:--|
|getImplementationSequence||		ensures checking order|
|getDeclaredIssues		|||
|commonBefore|	apkinfo	|the highest priority check|
|commonAfter|	apkinfo	|the lowest priority check|

### JavaScanner(BaseScanner)
|query / callback	|parameters	|description|
|:--|:--|:--|
|getApplicableMethodNames		|||
|getApplicableConstructorTypes		|||
|getApplicableReferenceNames	||	reference to static fields|
|getApplicableSuperClasses		|||
|visitMethodCall|	apkinfo, MethodInstance	||
|visitConstructor||	apkinfo, MethodInstance	||
|visitReferenceRead|	apkinfo, FieldInstance	|static fields|
|visitReferenceWrite		|||
|visitClass|	apkinfo, ClassObject||	
|otherAnalysis|	apkinfo	|general test over apkinfo|

### ResourceScanner(BaseScanner)
|query / callback|	parameters	|description|
|:--|:--|:--|
|checkResource|	apkinfo, ARSCParser|	overall resource files and R.java ID look up|

### OtherFileScanner(BaseScanner)
|query / callback	|parameters	|description|
|:--|:--|:--|
|getApplicableFiles|		|file type|
|appliesTo	|filename	|filter filename|
|visitFile	|apkinfo, File	||

### XmlScanner(BaseScanner)
|query / callback	|parameters|	description|
|:--|:--|:--|
|getApplicableXmlFiles	|	res xml, androidmanifest||
|appliesTo||		filter filename|
|getApplicableElement		|||
|getApplicableAttribute		|||
|visitElement	|apkinfo, Element	||
|visitAttribute	|apkinfo, Attr	||
