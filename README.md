# Facts

<span class="octicon octicon-bug"/> Detects various [security vulnerability patterns](#rules): SQL Injection, Cross-Site Scripting (XSS), Cross-Site Request Forgery (CSRF), XML eXternal Entity Injection (XXE), etc.

<span class="octicon octicon-git-branch"/> Taint analysis to track user input data.

<span class="octicon octicon-tools"/> One click refactoring for some vulnerabilities.

<span class="octicon octicon-code"/> Analyzes .NET and [.NET Core](https://en.wikipedia.org/wiki/.NET_Framework#.NET_Core) projects in a background (intellisense) or during a build.

<span class="octicon octicon-pulse"/> Continuous Integration (CI) through [MSBuild](https://msdn.microsoft.com/en-us/library/dd393574.aspx).

<span class="octicon octicon-plug"/> Works on Visual Studio 2015 or higher. Free Visual Studio [Community](https://www.visualstudio.com/en-us/products/visual-studio-community-vs.aspx) and paid Professional and Enterprise editions are supported.

<span class="octicon octicon-mark-github"/> [Open Source](https://github.com/security-code-scan)
# Installation
Security Code Scan (SCS) can be installed as:
* [Visual Studio extension](https://marketplace.visualstudio.com/items?itemName=PhilippeArteau.RoslynSecurityGuard). Use the link or open "Tools > Extensions and Updates..." Select "Online" in the tree on the left and search for RoslynSecurityGuard in the right upper field. Click "Download" and install.
* [NuGet package](https://www.nuget.org/packages/RoslynSecurityGuard/).
  * Right-click on the root item in your solution. Select "Manage NuGet Packages for Solution...". Select "Browse" on the top and search for Security Code Scan. Select project you want to install into and click "Install".
  * Another option is to install the package into all projects in a solution: use "Tools > NuGet Package Manager > Package Manager Console". Run the command `Get-Project -All | Install-Package SecurityCodeScan`.

Installing it as NuGet package gives an advantage to choose projects in a solution that should be analyzed. It is a good idea to exclude test projects, because they do not make it into a final product. However it requires discipline to install SCS into every solution a developer works with. Installing it as a Visual Studio extension is a single install action.

Because of the [Roslyn](https://github.com/dotnet/roslyn) technology SCS is based on only the NuGet version runs during a build (VS extension provides intellisense only) and can be integrated to any Continuous Integration (CI) server that supports [MSBuild](https://msdn.microsoft.com/en-us/library/dd393574.aspx).
# Configuration
## Full Solution Analysis
*Full solution analysis* is a Visual Studio (2015 Update 3 RC and later) feature that enables you to choose whether you see code analysis issues only in open Visual C# or Visual Basic files in your solution, or in both open and closed Visual C# or Visual Basic files in your solution. For performance reasons it is disabled by default. It is not needed if SCS is installed as NuGet package. If VS extension case open Tools > Options in Visual Studio. Select Text Editor > C# (or Basic) > Advanced. Make sure the "Enable full solution analysis" is checked:

![Full Solution Analysis](/images/fullsolution.png)
## Analyzing Config Files
To enable analysis of config files you need to modify all C#(.csproj) and VB.NET(.vbproj) projects in a solution and add "AdditionalFileItemNames" element as shown below:
```xml
<?xml version="1.0" encoding="utf-8"?>
<Project DefaultTargets="Build" ToolsVersion="4.0" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <PropertyGroup>
    [..]
    <TargetFrameworkProfile />
    <!-- Add the line below -->
    <AdditionalFileItemNames>$(AdditionalFileItemNames);Content</AdditionalFileItemNames>
  </PropertyGroup>
```
The helper PowerShell script can be used to do it automatically for all projects in a subfolder:
```ps1
Get-ChildItem *.csproj -Recurse | ForEach-Object {
$content = [xml] (Get-Content $_)
     
if (-not $content.Project.PropertyGroup[0].AdditionalFileItemNames)
    {
    Write-Host "AdditionalFileItemNames missing in $_"
    $additionalFileItemNamesElt = $content.CreateElement("AdditionalFileItemNames",
    "http://schemas.microsoft.com/developer/msbuild/2003")
    $additionalFileItemNamesElt.set_InnerText('$(AdditionalFileItemNames);Content')
    $content.Project.PropertyGroup[0].AppendChild($additionalFileItemNamesElt)
    }

Set-ItemProperty $_ -name IsReadOnly -value $false
$content.Save($_)
# Normalize line endings
(Get-Content $_ -Encoding UTF8) | Set-Content $_ -Encoding UTF8
}
```
## Testing on WebGoat.NET
Download an intentionally vulnerable project [WebGoat.NET](https://github.com/OWASP/WebGoat.NET/zipball/master) for testing. Open the solution. If you have installed SCS as a VS extension you should see warning after few seconds in the "Errors" tab. Make sure IntelliSense results are not filtered in the window:

![Intellisense](/images/intellisense.png)

If SCS is installed as NuGet package you'll need to build the solution. Then you should see the warning in the "Errors" and "Output" tabs:

![Intellisense](/images/output.png)
## Severity
Each warning severity is configurable: expand References > Analyzers > SecurityCodeScan under the project in a Solution window, right click on a warning ID and modify the severity. WebGoat.NET.ruleset will be automatically saved in the project's directory:

![Intellisense](/images/severity.png)
## Troubleshooting
If no SCS warnings are displayed, temporarily disable other installed analyzers. A buggy analyzer may [affect results from other analyzers](https://github.com/dotnet/roslyn/issues/23879).
# Rules
## Injection
#### References
[OWASP: Top 10 2013-A1-Injection](https://www.owasp.org/index.php/Top_10_2013-A1-Injection)

<div id="SCS0001"></div>
### SCS0001 - Command Injection
The dynamic value passed to the command execution should be validated.
#### Risk
If a malicious user is able to controlled either the command FileName or Arguments, he might be able to execute unwanted commands or add unwanted argument. This behavior would not be possible if input parameter are validate against a white-list of characters.
#### Vulnerable Code
```cs
var p = new Process();
p.StartInfo.FileName = "exportLegacy.exe";
p.StartInfo.Arguments = " -user " + input + " -role user";
p.Start();
```
#### Solution
```cs
Regex rgx = new Regex(@"^[a-zA-Z0-9]+$");
if(rgx.IsMatch(input))
{
    var p = new Process();
    p.StartInfo.FileName = "exportLegacy.exe";
    p.StartInfo.Arguments = " -user " + input + " -role user";
    p.Start();
}
```
#### References
[OWASP: Command Injection](https://www.owasp.org/index.php/Command_Injection)

[OWASP: Top 10 2013-A1-Injection](https://www.owasp.org/index.php/Top_10_2013-A1-Injection)

[CWE-78: Improper Neutralization of Special Elements used in an OS Command ('OS Command Injection')](http://cwe.mitre.org/data/definitions/78.html)

<div id="SCS0003"></div>
### SCS0003 - XPath Injection
The dynamic value passed to the XPath query should be validated.
#### Risk
If the user input is not properly filtered, a malicious user could extend the XPath query.
#### Vulnerable Code
```cs
var doc = new XmlDocument();
doc.Load("/config.xml");
var results = doc.SelectNodes("/Config/Devices/Device[id='" + input + "']");
```
#### Solution
```cs
Regex rgx = new Regex(@"^[a-zA-Z0-9]+$");
if(rgx.IsMatch(input)) //Additional validation
{
    XmlDocument doc = new XmlDocument();
    doc.Load("/config.xml");
    var results = doc.SelectNodes("/Config/Devices/Device[id='" + input + "']");
}
```
#### References
[CWE-643: Improper Neutralization of Data within XPath Expressions ('XPath Injection')](http://cwe.mitre.org/data/definitions/643.html)

[CERT: IDS09-J. Prevent XPath Injection (archive)](https://www.securecoding.cert.org/confluence/pages/viewpage.action?pageId=61407250)

[Black Hat Europe 2012: Hacking XPath 2.0](http://media.blackhat.com/bh-eu-12/Siddharth/bh-eu-12-Siddharth-Xpath-WP.pdf)

[Balisage: XQuery Injection](http://www.balisage.net/Proceedings/vol7/html/Vlist02/BalisageVol7-Vlist02.html)

[WASC-39: XPath Injection](http://projects.webappsec.org/w/page/13246963/SQL%20Injection)

[OWASP: Top 10 2013-A1-Injection](https://www.owasp.org/index.php/Top_10_2013-A1-Injection)

<div id="SCS0007"></div>
### SCS0007 - XML eXternal Entity Injection (XXE)
The XML parser is configured incorrectly. The operation could be vulnerable to XML eXternal Entity (XXE) processing.
#### Risk
#### Vulnerable Code
Prior to .NET 4.0
```cs
var settings = new XmlReaderSettings();
settings.ProhibitDtd = false;
XmlReader reader = XmlReader.Create(inputXml, settings);
```
.NET 4.0 - .NET 4.5.2
```cs
var settings = new XmlReaderSettings();
settings.DtdProcessing = DtdProcessing.Parse;
XmlReader reader = XmlReader.Create(inputXml, settings);
```
#### Solution
Prior to .NET 4.0
```cs
var settings = new XmlReaderSettings();
settings.ProhibitDtd = true; // default is false!
XmlReader reader = XmlReader.Create(inputXml, settings);
```
.NET 4.0 - .NET 4.5.2
```cs
var settings = new XmlReaderSettings();
settings.DtdProcessing = DtdProcessing.Prohibit; // default is DtdProcessing.Parse!
XmlReader reader = XmlReader.Create(inputXml, settings);
```
.NET 4.5.2 and later

In .NET Framework versions 4.5.2 and up, XmlTextReader's internal XmlResolver is set to null by default, making the XmlTextReader ignore DTDs by default. The XmlTextReader can become unsafe if if you create your own non-null XmlResolver with default or unsafe settings.
#### References
[OWASP.org: XML External Entity (XXE) Prevention Cheat Sheet (.NET)](https://www.owasp.org/index.php/XML_External_Entity_(XXE)_Prevention_Cheat_Sheet#.NET)

[CWE-611: Improper Restriction of XML External Entity Reference ('XXE')](http://cwe.mitre.org/data/definitions/611.html)

[CERT: IDS10-J. Prevent XML external entity attacks](https://www.securecoding.cert.org/confluence/pages/viewpage.action?pageId=61702260)

[OWASP.org: XML External Entity (XXE) Processing](https://www.owasp.org/index.php/XML_External_Entity_%28XXE%29_Processing)

[WS-Attacks.org: XML Entity Expansion](http://www.ws-attacks.org/index.php/XML_Entity_Expansion)

[WS-Attacks.org: XML External Entity DOS](http://www.ws-attacks.org/index.php/XML_External_Entity_DOS)

[WS-Attacks.org: XML Entity Reference Attack](http://www.ws-attacks.org/index.php/XML_Entity_Reference_Attack)

[Identifying Xml eXternal Entity vulnerability (XXE)](http://blog.h3xstream.com/2014/06/identifying-xml-external-entity.html)

<div id="SCS0018"></div>
### SCS0018 - Path Traversal
A path traversal attack (also known as directory traversal) aims to access files and directories that are stored outside the expected directory.By manipulating variables that reference files with “dot-dot-slash (../)” sequences and its variations or by using absolute file paths, it may be possible to access arbitrary files and directories stored on file system including application source code or configuration and critical system files.
#### Risk
With a malicious relative path, an attacker could reach a secret file.
#### Vulnerable Code
```cs
[RedirectingAction]
public ActionResult Download(string fileName)
{
    byte[] fileBytes = System.IO.File.ReadAllBytes(Server.MapPath("~/ClientDocument/") + fileName);
    return File(fileBytes, System.Net.Mime.MediaTypeNames.Application.Octet, fileName);
}
```
The following request downloads a file of the attacker choice:
`http://www.address.com/Home/Download?fileName=../../somefile.txt`
#### Solution
Do not try to strip invalid characters. Fail if any unexpected character is detected.
```cs
private static readonly char[] InvalidFilenameChars = Path.GetInvalidFileNameChars();

[RedirectingAction]
public ActionResult Download(string fileName)
{
    if (fileName.IndexOfAny(InvalidFilenameChars) >= 0)
        return new HttpStatusCodeResult(HttpStatusCode.BadRequest);
        
    byte[] fileBytes = System.IO.File.ReadAllBytes(Server.MapPath("~/ClientDocument/") + fileName);
    return File(fileBytes, System.Net.Mime.MediaTypeNames.Application.Octet, fileName);
}
```
If the input is not supplied by user or a validation is in place the warning can be suppressed.
#### References
[OWASP: Path Traversal](https://www.owasp.org/index.php/Path_Traversal)

[OS Command Injection, Path Traversal & Local File Inclusion Vulnerability - Notes](https://riseandhack.blogspot.com/2015/02/os-command-injection-path-traversal.html)
<div id="SCS0029"></div>
### SCS0029 - Cross-Site Scripting (XSS)
A potential XSS was found. The endpoint returns a variable from the client input that has not been encoded. To protect against stored XSS attacks, make sure any dynamic content coming from user or data store cannot be used to inject JavaScript on a page. Most modern frameworks will escape dynamic content by default automatically (Razor for example) or by using special syntax (`<%: content %>`, `<%= HttpUtility.HtmlEncode(content) %>`).
#### Risk
XSS could be used to execute unwanted JavaScript in a client's browser.
XSS can be used to steal the cookie containing the user’s session ID. There is rarely a good reason to read or manipulate cookies in client-side JavaScript, so consider marking cookies as [HTTP-only](#SCS0009), meaning that cookies will be received, stored, and sent by the browser, but cannot be modified or read by JavaScript.
#### Vulnerable Code
```cs
public class TestController : Controller
{
    [HttpGet(""{myParam}"")]
    public string Get(string myParam)
    {
        return "value " + myParam;
    }
}
```
#### Solution
```cs
public class TestController : Controller
{
    [HttpGet(""{myParam}"")]
    public string Get(string myParam)
    {
        return "value " + HttpUtility.HtmlEncode(myParam);
    }
}
```
#### References
[WASC-8: Cross Site Scripting](http://projects.webappsec.org/w/page/13246920/Cross%20Site%20Scripting)

[OWASP: XSS Prevention Cheat Sheet](https://www.owasp.org/index.php/XSS_%28Cross_Site_Scripting%29_Prevention_Cheat_Sheet)

[OWASP: Top 10 2013-A3: Cross-Site Scripting (XSS)](https://www.owasp.org/index.php/Top_10_2013-A3-Cross-Site_Scripting_%28XSS%29)

[CWE-79: Improper Neutralization of Input During Web Page Generation ('Cross-site Scripting')](http://cwe.mitre.org/data/definitions/79.html)

## SQL Injection
SQL injection flaws are introduced when software developers create dynamic database queries that include user supplied input.
#### Risk
Malicious user might get direct read and/or write access to the database. If the database is poorly configured the attacker might even get Remote Code Execution (RCE) on the machine running the database.
#### References
[WASC-19: SQL Injection](http://projects.webappsec.org/w/page/13246963/SQL%20Injection)

[OWASP: SQL Injection Prevention Cheat Sheet](https://www.owasp.org/index.php/SQL_Injection_Prevention_Cheat_Sheet)

[OWASP: Query Parameterization Cheat Sheet](https://www.owasp.org/index.php/Query_Parameterization_Cheat_Sheet)

[CAPEC-66: SQL Injection](http://capec.mitre.org/data/definitions/66.html)

[CWE-89: Improper Neutralization of Special Elements used in an SQL Command ('SQL Injection')](http://cwe.mitre.org/data/definitions/89.html)
<div id="SCS0002"></div>
### SCS0002 - SQL Injection (LINQ)
#### Vulnerable Code
```cs
db.ExecuteQuery(@"SELECT name FROM dbo.Users WHERE UserId = " + inputId + " AND group = 5");
```
```cs
var query = "SELECT name FROM dbo.Users WHERE UserId = " + userId + " AND group = 5";
var id = context.ExecuteQuery<IEnumerable<string>>(query).SingleOrDefault();
```
#### Solution
```cs
var query = from user in db.Users
where user.UserId == inputId
select user.name;
```
```cs
var query = "SELECT name FROM dbo.Users WHERE UserId = {0} AND group = 5";
var id = context.ExecuteQuery<IEnumerable<string>>(query, userId).SingleOrDefault();
```
#### References
[LINQ: How to Query for Information](https://msdn.microsoft.com/en-us/library/bb546192(v=vs.110).aspx)
<div id="SCS0014"></div>
### SCS0014 - SQL Injection (WebControls)
Unsafe usage of System.Web.UI.WebControls.SqlDataSource, System.Web.UI.WebControls.SqlDataSourceView or Microsoft.Whos.Framework.Data.SqlUtility.
#### Vulnerable Code
```
"Select * From Customers where CustomerName = " & txtCustomerName.Value
```
#### Solution
To help protect against SQL statement exploits, never create SQL queries using string concatenation. Instead, use a parameterized query and assign user input to parameter objects.
By default, the SqlDataSource control uses the System.Data.SqlClient data provider to work with SQL Server as the data source. The System.Data.SqlClient provider supports named parameters as placeholders, as shown in the following example:
```cs
<asp:sqlDataSource ID="EmployeeDetailsSqlDataSource" 
  SelectCommand="SELECT EmployeeID, LastName, FirstName FROM Employees WHERE EmployeeID = @EmpID"

  InsertCommand="INSERT INTO Employees(LastName, FirstName) VALUES (@LastName, @FirstName); 
                 SELECT @EmpID = SCOPE_IDENTITY()"
  UpdateCommand="UPDATE Employees SET LastName=@LastName, FirstName=@FirstName 
                   WHERE EmployeeID=@EmployeeID"
  DeleteCommand="DELETE Employees WHERE EmployeeID=@EmployeeID"

  ConnectionString="<%$ ConnectionStrings:NorthwindConnection %>"
  OnInserted="EmployeeDetailsSqlDataSource_OnInserted"
  RunAt="server">

  <SelectParameters>
    <asp:Parameter Name="EmpID" Type="Int32" DefaultValue="0" />
  </SelectParameters>

  <InsertParameters>
    <asp:Parameter Name="EmpID" Direction="Output" Type="Int32" DefaultValue="0" />
  </InsertParameters>

</asp:sqlDataSource>
```
If you are connecting to an OLE DB or ODBC data source, you can configure the SqlDataSource control to use the System.Data.OleDb or System.Data.Odbc provider to work with your data source, respectively. The System.Data.OleDb and System.Data.Odbc providers support only positional parameters identified by the "?" character, as shown in the following example:
```cs
...
<asp:SqlDataSource ID="EmployeeDetailsSqlDataSource" 
  SelectCommand="SELECT EmployeeID, LastName, FirstName, Address, City, Region, PostalCode
                 FROM Employees WHERE EmployeeID = ?"

  InsertCommand="INSERT INTO Employees(LastName, FirstName, Address, City, Region, PostalCode)
                 VALUES (?, ?, ?, ?, ?, ?); 
                 SELECT @EmpID = SCOPE_IDENTITY()"

  UpdateCommand="UPDATE Employees SET LastName=?, FirstName=?, Address=?,
                   City=?, Region=?, PostalCode=?
                 WHERE EmployeeID=?"
...
```
#### References
[MSDN: Using Parameters with the SqlDataSource Control](https://msdn.microsoft.com/en-us/library/z72eefad(v=vs.110).aspx)

[MSDN: Script Exploits Overview](https://msdn.microsoft.com/en-us/library/w1sw53ds(v=vs.110).aspx)

[MSDN: Filtering Event](https://msdn.microsoft.com/en-us/library/system.web.ui.webcontrols.sqldatasource.filtering(v=vs.110).aspx)

[See references in the main SQL Injection section](#SQLInjection)
<div id="SCS0020"></div>
### SCS0020 - SQL Injection (OLE DB)
Use parametrized queries to mitigate SQL injection.
#### Vulnerable Code
```cs
string queryString = "SELECT OrderID, CustomerID FROM Orders WHERE OrderId = " + userInput;

using (var connection = new OleDbConnection(connectionString))
{
    OleDbCommand command = new OleDbCommand(queryString, connection);
    connection.Open();
    OleDbDataReader reader = command.ExecuteReader();
}
```
#### Solution
```cs
string queryString = "SELECT OrderID, CustomerID FROM Orders WHERE OrderId = ?";

using (var connection = new OleDbConnection(connectionString))
{
    OleDbCommand command = new OleDbCommand(queryString, connection);
    command.Parameters.Add("@p1", OleDbType.Integer).Value = userInput;
    connection.Open();
    OleDbDataReader reader = command.ExecuteReader();
}
```
#### References
[OleDbCommand Documentation](https://msdn.microsoft.com/en-us/library/system.data.oledb.oledbcommand(v=vs.110).aspx)

[See references in the main SQL Injection section](#SQLInjection)
<div id="SCS0025"></div>
### SCS0025 - SQL Injection (ODBC)
Use parametrized queries to mitigate SQL injection.
#### Vulnerable Code
```cs
var command = new OdbcCommand("SELECT * FROM [user] WHERE id = " + userInput, connection);
OdbcDataReader reader = command.ExecuteReader();
```
#### Solution
```cs
var command = new OdbcCommand("SELECT * FROM [user] WHERE id = ?", connection);
command.Parameters.Add("@id", OdbcType.Int).Value = 4;
OdbcDataReader reader = command.ExecuteReader();
```
#### References
[OdbcCommand Documentation](https://msdn.microsoft.com/en-us/library/system.data.odbc.odbccommand(v=vs.110).aspx)

[See references in the main SQL Injection section](#SQLInjection)
<div id="SCS0026"></div>
### SCS0026 - SQL Injection (MsSQL Data Provider)
Use parametrized queries to mitigate SQL injection.
#### Vulnerable Code
```cs
var cmd = new SqlCommand("SELECT * FROM Users WHERE username = '" + username + "' and role='user'");
```
#### Solution
```cs
var cmd = new SqlCommand("SELECT * FROM Users WHERE username = @username and role='user'");
cmd.Parameters.AddWithValue("username", username);
```
#### References
[SqlCommand Class Documentation](https://msdn.microsoft.com/en-us/library/system.data.sqlclient.sqlcommand(v=vs.110).aspx)

[See references in the main SQL Injection section](#SQLInjection)
## Cryptography

<div id="SCS0004"></div>
### SCS0004 - Certificate Validation Disabled
Certificate Validation has been disabled. The communication could be intercepted.
#### Risk
Disabling certificate validation is often used to connect easily to a host that is not signed by a root [certificate authority](http://en.wikipedia.org/wiki/Certificate_authority). As a consequence, this is vulnerable to [Man-in-the-middle attacks](http://en.wikipedia.org/wiki/Man-in-the-middle_attack) since the client will trust any certificate.
#### Vulnerable Code
```cs
ServicePointManager.ServerCertificateValidationCallback += (sender, cert, chain, sslPolicyErrors) => true;
```
#### Solution
* Make sure the validation is disabled only in testing environment or
* Use [certificate pinning](https://www.owasp.org/index.php/Certificate_and_Public_Key_Pinning) for development or
* Use properly signed certificates for development

```cs
#if DEBUG
    ServicePointManager.ServerCertificateValidationCallback += (sender, cert, chain, sslPolicyErrors) => true;
#endif
```
#### References
[WASC-04: Insufficient Transport Layer Protection](http://projects.webappsec.org/w/page/13246945/Insufficient%20Transport%20Layer%20Protection)

[CWE-295: Improper Certificate Validation](http://cwe.mitre.org/data/definitions/295.html)
<div id="SCS0005"></div>
### SCS0005 - Weak Random Number Generator
The random numbers generated could be predicted.
#### Risk
The use of a predictable random value can lead to vulnerabilities when used in certain security critical contexts.
#### Vulnerable Code
```cs
var rnd = new Random();
byte[] buffer = new byte[16];
rnd.GetBytes(buffer);
return BitConverter.ToString(buffer);
```
#### Solution
```cs
using System.Security.Cryptography;
var rnd = RandomNumberGenerator.Create();
```
#### References
[WASC-04: Insufficient Transport Layer Protection](http://projects.webappsec.org/w/page/13246945/Insufficient%20Transport%20Layer%20Protection)

[CWE-295: Improper Certificate Validation](http://cwe.mitre.org/data/definitions/295.html)

<div id="SCS0006"></div>
### SCS0006 - Weak hashing function
MD5 or SHA1 have known collision weaknesses and are no longer considered strong hashing algorithms.
#### Vulnerable Code
```cs
var hashProvider = new SHA1CryptoServiceProvider();
var hash = hashProvider.ComputeHash(str);
```
#### Solution
Use SHA256 or SHA512.
Notice, that hashing algorithms are designed to be fast and shouldn't be used directly for hashing passwords. Use [adaptive algorithms](https://crackstation.net/hashing-security.htm) for the purpose.
```cs
var hashProvider = SHA256Managed.Create();
var hash = hashProvider.ComputeHash(str);
```
#### References
[MSDN: SHA256 Class documentation](https://msdn.microsoft.com/en-us/library/system.security.cryptography.sha256(v=vs.110).aspx)

[Salted Password Hashing - Doing it Right](https://crackstation.net/hashing-security.htm)
<div id="SCS0010"></div>
### SCS0010 - Weak cipher algorithm
DES and 3DES are not considered a strong cipher for modern applications. Currently, NIST recommends the usage of AES block ciphers instead.
#### Risk
Broken or deprecated ciphers have typically known weakness. A attacker might be able to brute force the secret key use for the encryption. The confidentiality and integrity of the information encrypted is at risk.
#### Vulnerable Code
```cs
DES DESalg = DES.Create();

// Create a string to encrypt. 
byte[] encrypted;
ICryptoTransform encryptor = DESalg.CreateEncryptor(key, zeroIV);

// Create the streams used for encryption. 
using (MemoryStream msEncrypt = new MemoryStream())
{
    using (CryptoStream csEncrypt = new CryptoStream(msEncrypt,
                                                     encryptor,
                                                     CryptoStreamMode.Write))
    {
        using (StreamWriter swEncrypt = new StreamWriter(csEncrypt))
        {
            //Write all data to the stream.
            swEncrypt.Write(Data);
        }
        encrypted = msEncrypt.ToArray();
        return encrypted;
    }
}
```
#### Solution
Use AES for symmetric encryption.
```cs
// Create a string to encrypt. 
byte[] encrypted;
var encryptor = new AesManaged();
encryptor.Key = key;
encryptor.GenerateIV();
var iv = encryptor.IV;

// Create the streams used for encryption. 
using (MemoryStream msEncrypt = new MemoryStream())
{
    using (CryptoStream csEncrypt = new CryptoStream(msEncrypt,
                                                     encryptor.CreateEncryptor(),
                                                     CryptoStreamMode.Write))
    {
        using (StreamWriter swEncrypt = new StreamWriter(csEncrypt))
        {
            //Write all data to the stream.
            swEncrypt.Write(Data);
        }
        encrypted = msEncrypt.ToArray();
        return encrypted;
    }
}
```
Notice that AES itself doesn't protect from encrypted data tampering. For an example of authenticated encryption see the [Solution in Weak Cipher Mode](#SCS0013)
#### References
[NIST Withdraws Outdated Data Encryption Standard](http://www.nist.gov/itl/fips/060205_des.cfm)

[CWE-326: Inadequate Encryption Strength](http://cwe.mitre.org/data/definitions/326.html)

[StackOverflow: Authenticated encryption example](http://stackoverflow.com/questions/202011/encrypt-and-decrypt-a-string/10366194#10366194)
<div id="SCS0011"></div>
### SCS0011 - Weak CBC Mode
The CBC mode alone is susceptible to padding oracle attack.
#### Risk
If an attacker is able to submit encrypted payload and the server is decrypting its content. The attacker is likely to decrypt its content.
#### Vulnerable Code
```cs
using (var aes = new AesManaged {
    KeySize = KeyBitSize,
    BlockSize = BlockBitSize,
    Mode = CipherMode.CBC,
    Padding = PaddingMode.PKCS7
})
{
    //Use random IV
    aes.GenerateIV();
    iv = aes.IV;
    using (var encrypter = aes.CreateEncryptor(cryptKey, iv))
    using (var cipherStream = new MemoryStream())
    {
        using (var cryptoStream = new CryptoStream(cipherStream, encrypter, CryptoStreamMode.Write))
        using (var binaryWriter = new BinaryWriter(cryptoStream))
        {
            //Encrypt Data
            binaryWriter.Write(secretMessage);
        }
        cipherText = cipherStream.ToArray();
    }
}
//No HMAC suffix to check integrity!!!
```
#### Solution
See the [Solution in Weak Cipher Mode](#SCS0013).
#### References
[Padding Oracles for the masses (by Matias Soler)](http://www.infobytesec.com/down/paddingoracle_openjam.pdf)

[Wikipedia: Authenticated encryption](http://en.wikipedia.org/wiki/Authenticated_encryption)

[NIST: Authenticated Encryption Modes](http://csrc.nist.gov/groups/ST/toolkit/BCM/modes_development.html#01)

[CAPEC: Padding Oracle Crypto Attack](http://capec.mitre.org/data/definitions/463.html)

[CWE-696: Incorrect Behavior Order](http://cwe.mitre.org/data/definitions/696.html)
<div id="SCS0012"></div>
### SCS0012 - Weak ECB Mode
ECB mode will produce the same result for identical blocks (ie: 16 bytes for AES). An attacker could be able to guess the encrypted message. The use of AES in CBC mode with a HMAC is recommended guaranteeing integrity and confidentiality.
#### Risk
The ECB mode will produce identical encrypted block for equivalent plain text block. This could allow an attacker that is eavesdropping to guess the content sent. This same property can also allow the recovery of the original message. Furthermore, this cipher mode alone does not guarantee integrity.
#### Vulnerable Code
```cs
using (var aes = new AesManaged {
    KeySize = KeyBitSize,
    BlockSize = BlockBitSize,
    Mode = CipherMode.ECB, // !!!
    Padding = PaddingMode.PKCS7
})
{
    //Use random IV
    aes.GenerateIV();
    iv = aes.IV;
    using (var encrypter = aes.CreateEncryptor(cryptKey, iv))
    using (var cipherStream = new MemoryStream())
    {
        using (var cryptoStream = new CryptoStream(cipherStream, encrypter, CryptoStreamMode.Write))
        using (var binaryWriter = new BinaryWriter(cryptoStream))
        {
            //Encrypt Data
            binaryWriter.Write(secretMessage);
        }
        cipherText = cipherStream.ToArray();
    }
}
//No HMAC suffix to check integrity!!!
```
#### Solution
Use some other mode, but notice that CBC without authenticated integrity check is vulnerable to another type of attack. For an example of authenticated integrity check see the [Solution in Weak Cipher Mode](#SCS0013).
#### References
[Wikipedia: ECB mode](https://en.wikipedia.org/wiki/Block_cipher_mode_of_operation#Electronic_Codebook_(ECB))

[Padding Oracles for the masses (by Matias Soler)](http://www.infobytesec.com/down/paddingoracle_openjam.pdf)

[Wikipedia: Authenticated encryption](http://en.wikipedia.org/wiki/Authenticated_encryption)

[NIST: Authenticated Encryption Modes](http://csrc.nist.gov/groups/ST/toolkit/BCM/modes_development.html#01)

[CAPEC: Padding Oracle Crypto Attack](http://capec.mitre.org/data/definitions/463.html)

[CWE-696: Incorrect Behavior Order](http://cwe.mitre.org/data/definitions/696.html)
<div id="SCS0013"></div>
### SCS0013 - Weak Cipher Mode
The cipher text produced is susceptible to alteration by an adversary.
#### Risk
The cipher provides no way to detect that the data has been tampered with. If the cipher text can be controlled by an attacker, it could be altered without detection. The use of AES in CBC mode with a HMAC is recommended guaranteeing integrity and confidentiality.
#### Vulnerable Code
```cs
using (var aes = new AesManaged {
    KeySize = KeyBitSize,
    BlockSize = BlockBitSize,
    Mode = CipherMode.OFB,
    Padding = PaddingMode.PKCS7
})
{
    using (var encrypter = aes.CreateEncryptor(cryptKey, new byte[16]))
    using (var cipherStream = new MemoryStream())
    {
        using (var cryptoStream = new CryptoStream(cipherStream, encrypter, CryptoStreamMode.Write))
        using (var binaryWriter = new BinaryWriter(cryptoStream))
        {
            //Encrypt Data
            binaryWriter.Write(secretMessage);
        }
        cipherText = cipherStream.ToArray();
    }
}
//Missing HMAC suffix to assure integrity
```
#### Solution
Using bouncy castle:
```cs
using Org.BouncyCastle.Crypto;
using Org.BouncyCastle.Crypto.Engines;
using Org.BouncyCastle.Crypto.Generators;
using Org.BouncyCastle.Crypto.Modes;
using Org.BouncyCastle.Crypto.Parameters;
using Org.BouncyCastle.Security;

public static readonly int BlockBitSize = 128;
public static readonly int KeyBitSize = 256;

public static byte[] SimpleEncrypt(byte[] secretMessage, byte[] key)
{
    //User Error Checks
    if (key == null || key.Length != KeyBitSize / 8)
        throw new ArgumentException(String.Format("Key needs to be {0} bit!", KeyBitSize), "key");

    if (secretMessage == null || secretMessage.Length == 0)
        throw new ArgumentException("Secret Message Required!", "secretMessage");

    //Using random nonce large enough not to repeat
    var nonce = new byte[NonceBitSize / 8];
    Random.NextBytes(nonce, 0, nonce.Length);

    var cipher = new GcmBlockCipher(new AesFastEngine());
    var parameters = new AeadParameters(new KeyParameter(key), MacBitSize, nonce, new byte[0]);
    cipher.Init(true, parameters);

    //Generate Cipher Text With Auth Tag
    var cipherText = new byte[cipher.GetOutputSize(secretMessage.Length)];
    var len = cipher.ProcessBytes(secretMessage, 0, secretMessage.Length, cipherText, 0);
    cipher.DoFinal(cipherText, len);

    //Assemble Message
    using (var combinedStream = new MemoryStream())
    {
        using (var binaryWriter = new BinaryWriter(combinedStream))
        {
            //Prepend Nonce
            binaryWriter.Write(nonce);
            //Write Cipher Text
            binaryWriter.Write(cipherText);
        }
        return combinedStream.ToArray();
    }
}
```
Custom implementation of Encrypt and HMAC:
```cs
using System.IO;
using System.Security.Cryptography;
public static byte[] SimpleEncrypt(byte[] secretMessage, byte[] cryptKey, byte[] authKey, byte[] nonSecretPayload = null)
{
    //User Error Checks
    if (cryptKey == null || cryptKey.Length != KeyBitSize / 8)
        throw new ArgumentException(String.Format("Key needs to be {0} bit!", KeyBitSize), "cryptKey");

    if (authKey == null || authKey.Length != KeyBitSize / 8)
        throw new ArgumentException(String.Format("Key needs to be {0} bit!", KeyBitSize), "authKey");

    if (secretMessage == null || secretMessage.Length < 1)
        throw new ArgumentException("Secret Message Required!", "secretMessage");

    byte[] cipherText;
    byte[] iv;
    using (var aes = new AesManaged {
        KeySize = KeyBitSize,
        BlockSize = BlockBitSize,
        Mode = CipherMode.CBC,
        Padding = PaddingMode.PKCS7
    })
    {
        //Use random IV
        aes.GenerateIV();
        iv = aes.IV;
        using (var encrypter = aes.CreateEncryptor(cryptKey, iv))
        using (var cipherStream = new MemoryStream())
        {
            using (var cryptoStream = new CryptoStream(cipherStream, encrypter, CryptoStreamMode.Write))
            using (var binaryWriter = new BinaryWriter(cryptoStream))
            {
            //Encrypt Data
            binaryWriter.Write(secretMessage);
            }
            cipherText = cipherStream.ToArray();
        }
    }
    //Assemble encrypted message and add authentication
    using (var hmac = new HMACSHA256(authKey))
    using (var encryptedStream = new MemoryStream())
    {
        using (var binaryWriter = new BinaryWriter(encryptedStream))
        {
            //Prepend IV
            binaryWriter.Write(iv);
            //Write Ciphertext
            binaryWriter.Write(cipherText);
            binaryWriter.Flush();
            //Authenticate all data
            var tag = hmac.ComputeHash(encryptedStream.ToArray());
            //Postpend tag
            binaryWriter.Write(tag);
        }
        return encryptedStream.ToArray();
    }
}
```
#### References
[Padding Oracles for the masses (by Matias Soler)](http://www.infobytesec.com/down/paddingoracle_openjam.pdf)

[Wikipedia: Authenticated encryption](http://en.wikipedia.org/wiki/Authenticated_encryption)

[NIST: Authenticated Encryption Modes](http://csrc.nist.gov/groups/ST/toolkit/BCM/modes_development.html#01)

[CAPEC: Padding Oracle Crypto Attack](http://capec.mitre.org/data/definitions/463.html)

[CWE-696: Incorrect Behavior Order](http://cwe.mitre.org/data/definitions/696.html)

## Cookies

<div id="SCS0008"></div>
### SCS0008 - Cookie Without SSL Flag
It is recommended to specify the Secure flag to new cookie.
#### Risk
The Secure flag is a directive to the browser to make sure that the cookie is not sent by unencrypted channel
#### Vulnerable Code
The `requireSSL` value is explicitly set to `false` or the default is left.
```xml
<httpCookies requireSSL="false" [..] />
```
```cs
// default is left
var cookie = new HttpCookie("test");
// or explicitly set to false
var cookie = new HttpCookie("test");
cookie.Secure = false;
```
#### Solution
```xml
<httpCookies requireSSL="true" [..] />
```
```cs
var cookie = new HttpCookie("test");
cookie.Secure = true; //Add this flag
cookie.HttpOnly = true;
```
#### References
[CWE-614: Sensitive Cookie in HTTPS Session Without 'Secure' Attribute](https://cwe.mitre.org/data/definitions/614.html)

[CWE-315: Cleartext Storage of Sensitive Information in a Cookie](https://cwe.mitre.org/data/definitions/315.html)

[CWE-311: Missing Encryption of Sensitive Data](https://cwe.mitre.org/data/definitions/311.html)

[OWASP: Secure Flag](https://www.owasp.org/index.php/SecureFlag)

[Rapid7: Missing Secure Flag From SSL Cookie](https://www.rapid7.com/db/vulnerabilities/http-cookie-secure-flag)

<div id="SCS0009"></div>
### SCS0009 - Cookie Without HttpOnly Flag
It is recommended to specify the HttpOnly flag to new cookie.
#### Risk
Cookies that doesn't have the flag set are available to JavaScript running on the same domain. When a user is the target of a "Cross-Site Scripting", the attacker would benefit greatly from getting the session id.
#### Vulnerable Code
The `httpOnlyCookies` value is explicitly set to `false` or the default is left.
```xml
<httpCookies httpOnlyCookies="false" [..] />
```
```cs
// default is left
var cookie = new HttpCookie("test");
// or explicitly set to false
var cookie = new HttpCookie("test");
cookie.HttpOnly = false;
```
#### Solution
```xml
<httpCookies httpOnlyCookies="true" [..] />
```
```cs
var cookie = new HttpCookie("test");
cookie.Secure = true;
cookie.HttpOnly = true; //Add this flag
```
#### References
[Coding Horror blog: Protecting Your Cookies: HttpOnly](http://blog.codinghorror.com/protecting-your-cookies-httponly/)

[OWASP: HttpOnly](https://www.owasp.org/index.php/HttpOnly)

[Rapid7: Missing HttpOnly Flag From Cookie](https://www.rapid7.com/db/vulnerabilities/http-cookie-http-only-flag)

## View State

<div id="SCS0023"></div>
### SCS0023 - View State Not Encrypted
The `viewStateEncryptionMode` is not set to `Always` in configuration file.
#### Risk
Web Forms controls use hidden base64 encoded fields to store state information. If sensitve information is stored there it may be leaked to the client side.
#### Vulnerable Code
The default value is `Auto`:
```xml
<system.web>
   ...
   <pages [..] validateRequest="true" [..]/>
   ...
</system.web>
```
or
```xml
<system.web>
   ...
   <pages [..] viewStateEncryptionMode="Never" [..]/>
   ...
</system.web>
```
#### Solution
Explicitly set to `Always` and encrypt with with the .NET [machine key](https://msdn.microsoft.com/en-us/library/w8h3skw9(v=vs.100).aspx):
```xml
<system.web>
   ...
   <pages [..] viewStateEncryptionMode="Always" [..]/>
   ...
</system.web>
```
#### References
[MSDN: pages Element (ASP.NET Settings Schema)](https://msdn.microsoft.com/en-us/library/950xf363(v=vs.100).aspx)

[MSDN: machineKey Element (ASP.NET Settings Schema)](https://msdn.microsoft.com/en-us/library/w8h3skw9(v=vs.100).aspx)
<div id="SCS0024"></div>
### SCS0024 - View State MAC Disabled
The `enableViewStateMac` is disabled in configuration file. (This feature cannot be disabled starting .NET 4.5.1)
#### Risk
The view state could be altered by an attacker.
#### Vulnerable Code
```xml
<system.web>
   ...
   <pages [..] enableViewStateMac="false" [..]/>
   ...
</system.web>
```
#### Solution
The default value is secure - `true`.
Or set it explicitly:
```xml
<system.web>
   ...
   <pages [..] enableViewStateMac="true" [..]/>
   ...
</system.web>
```
#### References
[MSDN: pages Element (ASP.NET Settings Schema)](https://msdn.microsoft.com/en-us/library/950xf363(v=vs.100).aspx)
## Request Validation

<div id="SCS0017"></div>
### SCS0017 - Request Validation Disabled (Attribute)
Request validation is disabled. Request validation allows the filtering of some [XSS](#SCS0029) patterns submitted to the application.
#### Risk
[XSS](#SCS0029)
#### Vulnerable Code
```cs
public class TestController
{
    [HttpPost]
    [ValidateInput(false)]
    public ActionResult ControllerMethod(string input) {
        return f(input);
    }
}
```
#### Solution
Although it performs blacklisting (that is worse than whitelisting by definition) and bypasses are known. It provides a basic first line of defense for your application. Do not disable the validation:
```cs
public class TestController
{
    [HttpPost]
    public ActionResult ControllerMethod(string input) {
        return f(input);
    }
}
```
Always user proper encoder (Html, Url, etc.) before displaying or using user supplied data (even if it is loaded from database).
#### References
See [XSS](#SCS0029) references.

<div id="SCS0021"></div>
### SCS0021 - Request Validation Disabled (Configuration File)
The `validateRequest` which provides additional protection against [XSS](#SCS0029) is disabled in configuration file.
#### Risk
[XSS](#SCS0029)
#### Vulnerable Code
```xml
<system.web>
   ...
   <pages [..] validateRequest="false" [..]/>
   ...
</system.web>
```
#### Solution
Although it performs blacklisting (that is worse than whitelisting by definition) and bypasses are known. It provides a basic first line of defense for your application. Do not disable the validation:
The default value is `true`.
Or set it explicitly:
```xml
<system.web>
   ...
   <pages [..] validateRequest="true" [..]/>
   ...
</system.web>
```
#### References
[MSDN: pages Element (ASP.NET Settings Schema)](https://msdn.microsoft.com/en-us/library/950xf363(v=vs.100).aspx)

See [XSS](#SCS0029) references.

## Password Management

<div id="SCS0015"></div>
### SCS0015 - Hardcoded Password
The password configuration to this API appears to be hardcoded.
#### Risk
If hard-coded passwords are used, it is almost certain that malicious users will gain access through the account in question.
#### Vulnerable Code
```cs
config.setPassword("NotSoSecr3tP@ssword");
```
#### Solution
It is recommended to externalize configuration such as password to avoid leakage of secret information. The source code or its binary form is more likely to be accessible by an attacker than a production configuration. To be managed safely, passwords and secret keys should be stored encrypted in separate configuration files. The certificate for decryption should be installed as non-exportable on the server machine.

Configuration file :
```xml
<?xml version="1.0" encoding="utf-8" ?>
<configuration>
    <appSettings>
    <add key="api_password" value="b3e521073ca276dc2b7caf6247b6ddc72d5e2d2d" />
  </appSettings>
</configuration>
```
Code:
```cs
string apiPassword = ConfigurationManager.AppSettings["api_password"];
config.setPassword(apiPassword);
```
#### References
[CWE-259: Use of Hard-coded Password](http://cwe.mitre.org/data/definitions/259.html)

<div id="SCS0034"></div>
### SCS0034 - Password RequiredLength Not Set
The RequiredLength property must be set with a minimum value of 8.
#### Risk
Weak password can be guessed or brute-forced.
#### Vulnerable Code
ASP.NET Identity default is 6.
```cs
PasswordValidator pwdv = new PasswordValidator();
```
#### Solution
See the solution for [Password Complexity](#SCS0033)
#### References
(MSDN: ASP.NET Identity PasswordValidator Class)(https://msdn.microsoft.com/en-us/library/microsoft.aspnet.identity.passwordvalidator(v=vs.108).aspx)
<div id="SCS0032"></div>
### SCS0032 - Password RequiredLength Too Small
The minimal length of a password is recommended to be set at least to 8.
#### Risk
Weak password can be guessed or brute-forced.
#### Vulnerable Code
```cs
PasswordValidator pwdv = new PasswordValidator
{
    RequiredLength = 6,
};
```
#### Solution
See the solution for [Password Complexity](#SCS0033)
#### References
(MSDN: ASP.NET Identity PasswordValidator Class)(https://msdn.microsoft.com/en-us/library/microsoft.aspnet.identity.passwordvalidator(v=vs.108).aspx)
<div id="SCS0033"></div>
### SCS0033 - Password Complexity
PasswordValidator should have at least two requirements for better security (RequiredLength, RequireDigit, RequireLowercase, RequireUppercase and/or RequireNonLetterOrDigit).
#### Risk
Weak password can be guessed or brute-forced.
#### Vulnerable Code
```cs
PasswordValidator pwdv = new PasswordValidator
{
    RequiredLength = 6,
};
```
#### Solution
```cs
PasswordValidator pwdv = new PasswordValidator
{
    RequiredLength = 8,
    RequireNonLetterOrDigit = true,
    RequireDigit = true,
    RequireLowercase = true,
    RequireUppercase = true,
};
```
#### References
(MSDN: ASP.NET Identity PasswordValidator Class)(https://msdn.microsoft.com/en-us/library/microsoft.aspnet.identity.passwordvalidator(v=vs.108).aspx)
## Other

<div id="SCS0016"></div>
### SCS0016 - Cross-Site Request Forgery (CSRF)
Anti-forgery token is missing.
#### Risk
An attacker could send a link to the victim. By visiting the malicious link, a web page would trigger a POST request (because it is a blind attack - the attacker doesn't see a response from triggered request and has no use from GET request and GET requests should not change a state on the server by definition) to the website. The victim would not be able to acknowledge that an action is made in the background, but his cookie would be automatically submitted if he is authenticated to the website. This attack does not require special interaction other than visiting a website.
#### Vulnerable Code
```cs
public class TestController
{
    [HttpPost]
    public ActionResult ControllerMethod(string input)
    {
        //Do an action in the context of the logged in user
    }
}
```
#### Solution
In your view:
```html
@Html.AntiForgeryToken()
```
In your controller:
```cs
public class TestController
{
    [HttpPost]
    [ValidateAntiForgeryToken] //Annotation added
    public ActionResult ControllerMethod(string input)
    {
        //Do an action in the context of the logged in user
    }
}
```
#### References
[OWASP: Cross-Site Request Forgery](https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF))

[OWASP: CSRF Prevention Cheat Sheet](https://www.owasp.org/index.php/Cross-Site_Request_Forgery_%28CSRF%29_Prevention_Cheat_Sheet)
<div id="SCS0019"></div>
### SCS0019 - OutputCache Conflict
Caching conflicts with authorization.
#### Risk
Having the annotation [OutputCache] will disable the annotation [Authorize] for the requests following the first one.
#### Vulnerable Code
```cs
[Authorize]
public class AdminController : Controller
{
    [OutputCache]
    public ActionResult Index()
    {
        return View();
    }
}
```
#### Solution
Remove the caching:
```cs
[Authorize]
public class AdminController : Controller
{
    public ActionResult Index()
    {
        return View();
    }
}
```
#### References
[Improving Performance with Output Caching](https://docs.microsoft.com/en-us/aspnet/mvc/overview/older-versions-1/controllers-and-routing/improving-performance-with-output-caching-cs)
<div id="SCS0022"></div>
### SCS0022 - Event Validation Disabled
The `enableEventValidation` is disabled in configuration file.
#### Risk
This feature reduces the risk of unauthorized or malicious post-back requests and callbacks. It is strongly recommended that you do not disable event validation. When the EnableEventValidation property is set to true, ASP.NET validates that a control event originated from the user interface that was rendered by that control.
#### Vulnerable Code
```xml
<system.web>
   ...
   <pages [..] enableEventValidation="false" [..]/>
   ...
</system.web>
```
#### Solution
The default value is secure - `true`.
Or set it explicitly:
```xml
<system.web>
   ...
   <pages [..] enableEventValidation="true" [..]/>
   ...
</system.web>
```
#### References
[MSDN: pages Element (ASP.NET Settings Schema)](https://msdn.microsoft.com/en-us/library/950xf363(v=vs.100).aspx)

[MSDN: Page.EnableEventValidation Property](http://msdn.microsoft.com/en-us/library/system.web.ui.page.enableeventvalidation.aspx)