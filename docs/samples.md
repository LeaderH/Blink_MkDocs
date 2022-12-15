# Sample Analysis Scenarios
Here we are going to introduce few frequently encountered analysis scenarios, and demonstrate how one can tackle them leveraging BLINK APIs with some examples. 

## Analyzing Method Calls
In AllowAllHostnameVerifierDetector, we tries to detect if there is an usage of `AllowAllHostnameVerifier()` class or `SSLSocketFactory.ALLOW_ALL_HOSTNAME_VERIFIER` field as `setHostnameVerifier`'s argument.

The targets in java would be like:

```java
URL url = new URL("https://www.google.com");
HttpsURLConnection connection = (HttpsURLConnection) url.openConnection();
connection.setHostnameVerifier(new AllowAllHostnameVerifier());
connection.setHostnameVerifier(SSLSocketFactory.ALLOW_ALL_HOSTNAME_VERIFIER);
```

The corresponding detecting workflows include:

1. query constructor with the name via `getApplicableConstructorTypes()`
2. implement `visitConstructor()` callback method an report the issue
3. query target methods with their name via `getApplicableMethodNames()`
4. implement `visitMethodCall()`, conduct analysis and report the issue


```py title="AllowAllHostnameVerifierDetector.py" linenums="28"
@overrides
def getApplicableConstructorTypes(self):
    return ["org.apache.http.conn.ssl.AllowAllHostnameVerifier"]

@overrides
def visitConstructor(self, context, constructor):
    context.report(self.ISSUE, constructor, constructor.location, "Using the `AllowAllHostnameVerifier` HostnameVerifier is unsafe "
                    + "because it always returns true, which could cause insecure network "
                    + "traffic due to trusting TLS/SSL server certificates for wrong "
                    + "hostnames")

@overrides
def getApplicableMethodNames(self):
    return ["setHostnameVerifier", "setDefaultHostnameVerifier"]

@overrides
def visitMethodCall(self, context, method):
    if context.methodMatches(method, None, False, "javax.net.ssl.HostnameVerifier"):
        arg = method.get_argument(0)
        if isinstance(arg, ArgType.Field) and arg.name == "ALLOW_ALL_HOSTNAME_VERIFIER":
            message = "Using the `ALLOW_ALL_HOSTNAME_VERIFIER` HostnameVerifier "\
                + "is unsafe because it always returns true, which could cause "\
                + "insecure network traffic due to trusting TLS/SSL server "\
                + "certificates for wrong hostnames"
            context.report(self.ISSUE, method, method.location, message)
            return
```

##	Analyzing field references
One can look for field reference of its name with `getApplicableReferenceNames()` and handle the return FieldObject within `visitReferenceRead()` or `visitReferenceWrite()`.

Here we have an example looking for usage of field `android.os.Build.SERIAL`

```py title="HardwareIdDetector.py" linenums="45"
@overrides
def getApplicableReferenceNames(self):
    return ["SERIAL"]

@overrides
def visitReferenceRead(self, context, reference):
    if context.isMemberInSubClassOf(reference, "android.os.Build", False):
        message = f"Using \"SERIAL\" to get device identifiers is not recommended"
        context.report(self.ISSUE, reference, reference.location, message)

```

## 	Analyzing xml contents
When searching targets in xml files, one has to first specify the scope of applicable xml files with `getApplicableXmlFiles()` follows by `getApplicableElements()` or `getApplicableAttributes()` to locate target xml element/attribute. Their corresponding callbacks are `visitElement()` and `visitAttribute()` respectively.

Below is an example detecting process that look into elements with tag name `application` within `AndroidManifest.xml`.

```py title="ManifestDetector.py" linenums="54"
@overrides
def getApplicableXmlFiles(self):
    return [Scope.MANIFEST]

@overrides
def getApplicableElements(self):
    return [TAG_APPLICATION]

@overrides
def visitElement(self, context, element):
    if not element.hasAttributeNS(ANDROID_URI, ATTR_ALLOW_BACKUP) and context.minSdk >= 4:
        context.report(self.ALLOW_BACKUP, element, element.location,
            "Should explicitly set `android:allowBackup` to `true` or " +
            "`false` (it's `true` by default, and that can have some security " +
            "implications for the application's data)")
```

##  Analyzing resource (ARSC) contents
It is a little different when it comes to accessing ARSC content as there is only ever going to be single one ARSC file in an Apk. As a result, a Detector only has to implement `checkResource()` callback to receive the APSCParser Object(an androguard object).

Following is an example showing how we can operate on ARSC as well as implement multiple types of ScannerInterfaces at the same time, which is the very case where the Detector have to specifically further inform the RuleManager their evaluating order with `getImplementationSequence()`.

```py title="LocaleFolderDetector.py" linenums="28"
@overrides
def getImplementationSequence(self):
    return [ResourcesScanner, JavaScanner]

@overrides
def checkResource(self, context, resource):
    pkgs = resource.get_packages_names()
    locales = set()
    for pkg in pkgs:
        locales.update(set(resource.get_locales(pkg)))

    self.threeLetterLocales = set()
    for locale in locales:
        language = locale.split("-")[0]
        if len(language) >= 3:
            self.threeLetterLocales.add(language)

@overrides
def getApplicableMethodNames(self):
    return ["getLocales"]

@overrides
def visitMethodCall(self, context, method):
    if not self.threeLetterLocales or context.getMinSdk() >= 21:
        return
    if not context.isMemberInSubClassOf(method, "android.content.res.AssetManager", False):
        return
    locales = ", ".join(list(self.threeLetterLocales))
    msg = f"The app will crash on platforms older than v21 " +\
        f"(minSdkVersion is {context.getMinSdk()}) because `AssetManager#getLocales` is called and it " +\
        f"contains one or more v21-style (3-letter or BCP47 locale) locales: {locales}"
    context.report(self.ISSUE, method, method.location, msg)
```


##  Analyzing Other Files contents
BLINK also provide a ScannerInterface for other untypical files that may exist in an Apk. For these kind of files, one can provide the scope of target files with `getApplicableFiles()`, file names with `appliesTo()`, and their callback would be `visitFile()`.

Scope Enum
```py title="issue.py" linenums="59"
class Scope(Enum):
    """
    file scopes
    """
    OTHER = 0b1
    ALL_RESOURCE_FILES = 0b10  # res/...xml
    MANIFEST = 0b100  # AndroidManifest.xml
```

Following is an example where we are checking if there are potential private key files got packed into an Apk, which is often undesirable. 

```py title="PrivateKeyDetector.py" linenums="23"
@overrides
def getApplicableFiles(self):
    return [Scope.OTHER]

@overrides
def visitFile(self, context, File):
    if self.isPrivateKeyFile(File):
        message = f"The '{File.name}' file seems to be a private key file. " + \
            f"Please make sure not to embed this in your APK file."
        context.report(self.ISSUE, None, File.location, message)

def isPrivateKeyFile(self, File):
    if not File.name.endswith("pem") and not File.name.endswith("key"):
        return False
    firstline = next(File.read_line())
    if firstline and firstline.startswith("---") and\
            ("PRIVATE KEY" in firstline):
        return True
    return False
```