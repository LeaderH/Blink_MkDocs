# Usage

## Command Line Entry point
---
`blink.cli`  
Entry point for an analysis task, one can specify input target APKs and ruleset directory, output types, format, and several analysis parameters.

**Example usage:**

`python -m blink.cli -i sample.apk -o csv -o txt -s -t -db -T test`

- output csv, txt and show to console
- time the analysis critical steps
- store result in database
- assign a tag to mark this analysis run

```
Usage: python -m blink.cli [OPTIONS]

  Blink: Andoird binary lint kit

Options:
  -i, --input PATH                input APK File/ Folder(single layer)
                                  [required]
  -d, --outdir DIRECTORY          Analysis Report Output Directory
  -o, --output-type [txt|xml|csv]
                                  Output file type
  -r, --rules DIRECTORY           Rules folder need to be checked
  -s, --show                      Show result to console
  -v, --verbose                   Show detailed result(imply show)
  -db, --database              Store result to database(need to set db_config)
  -T, --analyze-tag TEXT       Analysis tag mark this round of analysis(also
                               used as output dirname)
  -t, --timed                  Time critical steps
  -p, --prediction             Riskware Predicton(CICMalDroid2020)
  -seq, --sequential           Analyze target sequentially (disable
                               multiprocessing)
  --timeout INTEGER            Timeout value (in sec)
  -n, --process-cnt INTEGER    Mutiprocess core count
  --version
  --help                       Show this message and exit.
```



## Rule Management
---
`blink.config`  
Manage availabilities of rules(Detectors) in a ruleset, support regular expression syntax.

**Example usage:**

`python -m blink.config -e "Allow*" blink/Rules`

- eable rule with regex wildcard

```
Usage: python -m blink.config [OPTIONS] [RULE_DIR]

  If Enable/Disable all is set, single operations will be ignored Operation
  order: enable -> disable

  :param [RULE_DIR]: default to 'blink/Rules'

Options:
  -e, --enable TEXT   Enable rule with name match regular expression
  -d, --disable TEXT  Disable rule with name match regular expression
  -l, --show          [main op] List/Show rules
  -ea                 [main op] Enable all
  -da                 [main op] Disable all
  --help              Show this message and exit.
```

## Report format
---
### Result (single apk)
#### xml

```
<analysis target="lintTest1.apk" total="78">
	<report rule_file="C2dmDetector" count="1" target="lintTest1.apk">
		<issue id="UsingC2DM"...>
			<instance location="..." message="..."/>
		</issue>
	</report>
</analysis> 
```

#### txt

```
[C2dmDetector] - 1
	> UsingC2DM: 1
		- AndroidManifest.xml@4->android:name
		* The C2DM library does not work on Android P or newer devices; you should migrate to Firebase Cloud Messaging to ensure reliable message delivery
```
#### mongoDB document
```
{
	_id:62e21496af32c00c62d3107e,
	_AnalysisTag:"072822_124550",
	_APK:"com.blink.test.minApi16.debug",
	_VersionCode:"1",
	_MD5:"5b30bf0f130b5dff7cfd59c553612f33",
	AcceptsUserCertificates:1,
	AddJavascriptInterface:11,
	AllowAllHostnameVerifier:2,
	AllowBackup:0,
	AuthLeak:1,
	...
}
```
### Result (multiple apk)
#### csv
| _APK| _VersionCode | _MD5 |AcceptsUserCertificates|AddJavascriptInterface |
| :-- | :-- |:--|:--|:--|
| xxx.apk  | 1.0 | xxxx |0|1|
| yyy.apk  | 1.3 | xxxx |1|1|
| zzz.apk  | 2.0 | xxxx |1|0|

## Database lookup
---
```blink.db_lookup```  
Auxiliary functions interacting with BLINK's results stored with mongoDB.

**Example usage:**

`python -m blink.db_lookup rule_detail -i AllowBackup`
- lookup details about a rule

`python -m blink.db_lookup result -i sample.apk -T tag`
- lookup result of a apk 
- provide analyze_tag to filter result of specific run

`python -m blink.db_lookup rule_result -i AllowBackup`
- lookup apks that triggered a specific rule

```
Usage: python -m blink.db_lookup [OPTIONS] {rule_detail|result|rule_result}

  Manipulate MongoDB results

Options:
  -i, --identity TEXT     Rule Id or packageName
  -T, --analyze-tag TEXT  Analysis tag mark this round of analysis
  --apk-md5 TEXT          APK's md5
  --help                  Show this message and exit.
```

### Output
#### rule_detail
```
ID: AllowBackup
Category: SECURITY
Severity: WARNING
Priority: 3
Description: The allowBackup attribute determines if an application's data can be backed up and restored. It is documented at http://developer.android.com/reference/android/R.attr.html#allowBackup 
...
```
#### result
```
PackageName: com.blink.test.minApi16.debug
VesionCode: 1
MD5: 5b30bf0f130b5dff7cfd59c553612f33
AnalyzeTag: 072822_124550
        [AcceptsUserCertificates]: 1
        [AddJavascriptInterface]: 11
        [AllowAllHostnameVerifier]: 2
		...
        [WorldWriteableFiles]: 0
Appearence: 22, Total: 79
```
#### rule_result
```
RuleID [AllowBackup]
<Tag: 072822_122556>
at.tacticaldevc.panictrigger: 1
com.pierreduchemin.smsforward: 1
com.saverio.pdfviewer: 1
com.wesaphzt.privatelocation: 1
de.chagemann.regexcrossword: 1
nl.patrickkostjens.kandroid: 1
org.mozilla.klar: 1
sh.ftp.rocketninelabs.meditationassistant.opensource: 1
Appearence: 8, Total: 8
```
## RandomFileSelector
---
```blink.Util.RandomFileSelector```  
Auxiliary fuction for randomly select given number of target files and rename them.

**Example usage:**

`python -m blink.Util.RandomFileSelector -f DataSet/ -o Samples/ -n 10 -r --prefix case`
- randomly pick 10 test case from /DataSet to /Samples and rename as case\[\d\]
```
Usage: python -m blink.Util.RandomFileSelector [OPTIONS]

  Randomly chose and copy files in a folder.

Options:
  -f, --folder DIRECTORY  Target folder  [required]
  -o, --output DIRECTORY  Output folder  [required]
  -n, --number INTEGER    Amount  [required]
  --prefix TEXT           Rename copied file with prefix  [default: case]
  -r, --rename            Rename copied file
  --help                  Show this message and exit.
```