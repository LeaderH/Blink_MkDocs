# Detector Implementation

To implement a custom Detector that operates on BLINK, there envolves three steps as follow:

![img](img/workflow.png "Implementation workflow")

BLINK boost straightforward detecting rule conversion from [Android Lint](https://developer.android.com/studio/write/lint). We can get access to Android Lint's Detector Implementation via [AOSP](https://android.googlesource.com/platform/tools/base/+/refs/heads/mirror-goog-studio-main/lint/libs/lint-checks/src/main/java/com/android/tools/lint/checks/). Most of the checks used to be written in Java and converted to Kotlin gradually. However, regardless of the programing language they are in, they share the same disign logic and process. 

Let's take a look at one of rule of Android Lint and mark its critical sections:

![img](img/detector_java.png "AddJavaScriptInterface.java")

And here is how a converted Detector rule would be like in BLINK:

![img](img/detector_python.png "AddJavaScriptInterface.py")

## Issue Declaration
In the beginning, the Detector has to specify what kind of issues it is about to detect and provides basic information and an unique IssueId in forms of **Issue** objects to the RuleManger via `getDeclaredIssues()`.

An example would be like:
```py title="AddJavaScriptInterfaceDetector.py" linenums="12"
def __init__(self):
    self.ISSUE = Issue.create(
        "AddJavascriptInterface",
        "addJavascriptInterface Called",
        "For applications built for API levels below 17, `WebView#addJavascriptInterface` "
        + "presents a security hazard as JavaScript on the target web page has the "
        + "ability to use reflection to access the injected object's public fields and "
        + "thus manipulate the host application in unintended ways."
        + "https://labs.mwrinfosecurity.com/blog/2013/09/24/webview-addjavascriptinterface-remote-code-execution/",
        Category.SECURITY,
        9,
        Severity.WARNING
    )

@overrides
def getDeclaredIssues(self):
    return [self.ISSUE]
```


An Issue Object has following data fields:
```py
Issue(id, briefDescription, explanation, category, priority, severity, moreInfo)
```

|field|des|
|:--|:--|
|id|an unique Id| 
| briefDescription|short description|
|explanation|detailed information|
|category|7 predefined categories|
|priority|1-9|
|severity|5 severity levels|
|moreInfo|any other messages|

Category Enum
```py title="issue.py" linenums="41"
class Category(Enum):
    SECURITY = "Security"
    A11Y = "Accessibility"
    LINT = "Lint"
    CORRECTNESS = "Correctness"
    PERFORMANCE = "Performance"
    USABILITY = "Usability"
    I18N = "Internationalization"
```

Severity Enum
```py title="issue.py" linenums="51"
class Severity(Enum):
    INFO = "Info"
    WARNING = "Warning"
    CRITICAL = "Critical"
    ERROR = "Error"
    FATAL = "Fatal"
```
## Query Target Registration
Depends on what kind of resource a Detector would like to access, a Detector will implement various set of queries and call back methods defined in the ScannerInterfaces. Therefore, in this step, a Detector has to override these corresponding methods.

An example could be like for requesting a method with name:
```py title="AddJavaScriptInterfaceDetector.py" linenums="33"
@overrides
def getApplicableMethodNames(self):
    return [ADD_JAVASCRIPT_INTERFACE]
```
For more information about what kind of queries BLINK can provide, please look further into [ScannerInterfaces](scanner.md).

## Callback handling and Reporting
Most of the customized analysis magic happens in this step, this is when we receive our analysis targets and determine whether they have to be reported.

This is what it would be like in our example:
```py title="AddJavaScriptInterfaceDetector.py" linenums="37"
@overrides
def visitMethodCall(self, context, method):
    """
    context: Apkinfo
    method: MethodAnalysis
    """
    message = "`WebView.addJavascriptInterface` should not be called with " +\
        "minSdkVersion < 17 for security reasons: JavaScript can use reflection " +\
        "to manipulate application"
    if context.minSdk >= 17:
        return
    if context.methodMatches(method, WEB_VIEW, True, TYPE_OBJECT, TYPE_STRING):
        if DEBUG:
            print("addJavascriptInterface found!!")
        context.report(self.ISSUE, method, method.location, message)
        return
    pass
```

Detectors can report their finds with `report()` of Apkinfo(context), which will come along with every callbacks.

A report contains these messages:
```py 
Apkinfo.report(issue, node, location, message)
```

|field|des|
|:--|:--|
|issue|an issue instance| 
|node|reported element(could be method, xml node,...etc)|
|location|where the target is located|
|message|any addtional message|

