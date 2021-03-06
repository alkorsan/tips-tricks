# CSharp
Oh C#... the language of love! :D The one to rule them all, the best of the bests :D This document will be constantly in progress, at least I hope so. It is not a from zero to hero guide to teach C# language. It is rather some gathering of things that was useful or interesting for me while working as a programmer, especially at the beginning.

* [Language tips](#language-tips)
* [WCF](#wcf)
* [Performance and under the hood](#performance-and-under-the-hood) 
* [Other](#other)
* [Useful links](#useful-links)

### CSC Compiler
If you want to test some little piece of code and creating a new solution in Visual Studio is defenietely waste of time for you for such a little piece, than you can use CSC compiler directly from command line, see how to do it:
```powershell
$cscPath = "C:\Windows\Microsoft.NET\Framework64\v4.0.30319"
$cscFullPath = "$cscPath\csc.exe"
& "$cscFullPath" /out:program.exe /target:exe program.cs
```
If you need to add some DLL's you have to use `/r` switch:
```powershell
& "$cscFullPath" /r:System.DirectoryServices.dll, System.DirectoryServices.AccountManagement.dll /out:program.exe /target:exe program.cs
```
It is a very good idea to use CSC while learning and trying new things. You just have to create one .cs file, compile it and run!

## Language tips

### IDisposable
Every class that has some unmanaged resources should implement **IDisposable** interface. It is because Garbage Collector knows nothing about such resources. Here is an example of implementation proposed by Microsoft:
```csharp
public class MyClass : IDisposable
{
   private bool isDisposed=false;

   public void Dispose()
   {
      Dispose(true);
      GC.SuppressFinalize(this);
   }

   protected void Dispose(bool disposing)
   {
      if(!isDisposed)
      {
         if(disposing)
         {
            // free MANAGED resources
         }
         // free UNMANAGED resources
      }
      isDisposed=true;
   }

   ~MyClass() 
   {
      Dispose(false);
   }
}
```

### Syntax
There are many nice syntax construction that may help you a lot while working with C#. With every new version of the language there is also some nice new syntactic sugar added. Here you will find few examples from basic to some less known language constructions.

##### Extension methods
```csharp
static bool Contains(this string keyString, char c)
{
   return keyString.IndexOf(c) >= 0;
}

bool test = "Some string".Contains('s');
```

##### User-defined conversion
```csharp
public class Digit
{
   public Digit(double d) { val = d; }
   public double val;

   // User-defined conversion from Digit to double
   public static implicit operator double(Digit d)
   {
      return d.val;
   }

   //  User-defined conversion from double to Digit
   public static implicit operator Digit(double d)
   {    
      return new Digit(d);
   }
}
```
Keep in mind that implicit conversion should **never** throw any exception or lose data. If you cannot fullfil those conditions you should rather implement explicit conversion.

##### Check if integer is not overflowing the maximum value
```csharp
checked {
   int value = int.MaxValue;
   value += 1;
}
```
The code above will throw **OverflowException** if the variable `value` will exceed the maximum integer value, instead of for example giving a strange result.

##### Logical consequence
Logical consequence, also known as logical implication or entailment, is one of the basic boolean operation, but there is no logical operator in C# that implements it. You can use the following boolean operation to implement logical consequence:
```csharp
bool a = true;
bool b = false;

bool consequence = ( !a || b );
```
Here you can see what are the results of the consequence for different boolean values:
```
a   b   
0   0   (!0 || 0) = 1 || 0 = 1
1   0   (!1 || 0) = 0 || 0 = 0
0   1   (!0 || 1) = 1 || 1 = 1
1   1   (!1 || 1) = 0 || 1 = 1
```

### Enums

##### Enums with descriptions
Descriptions may be useful if you want to list enums to display them to user, it is very easy to add them:
```csharp
using System.ComponentModel;

public enum Result 
{
   [Description("No result")]
   None = 0,
   [Description("Correct result")]
   Correct = 1
}
```

##### Get description - extension method
Here is an example of an extension method that will allow you to get the description of given enum value:
```csharp
public static string GetDescription(this Enum value)
{
   Type type = value.GetType();
   string name = Enum.GetName(type, value);
   if (!string.IsNullOrEmpty(name))
   {
      FieldInfo field = type.GetField(name);
      if (field != null)
      {
         DescriptionAttribute descriptionAttribute = 
            (DescriptionAttribute)Attribute.GetCustomAttribute(field, typeof(DescriptionAttribute));
         if (descriptionAttribute != null)
            return descriptionAttribute.Description;
       }
   }
   return string.Empty;
}
```

##### Iterate through all enum values
```csharp
foreach(EnumType value in Enum.GetValues(typeof(EnumType)))
{
   Console.WriteLine(value.GetDescription());
}
```

### Image and byte arrays

##### Convert Image to byte[]
```csharp
MemoryStream mstream = new MemoryStream();
image.Save(mstream, System.Drawing.Imaging.ImageFormat.Png);
byte[] bytes  = mstream.ToArray();
```

##### Convert byte[] to Image
```csharp
MemoryStream mstream = new MemoryStream(imageBytes);
Image image = Image.FromStream(mstream);
```

##### Image file MIME type
```csharp
ImageCodecInfo codec = ImageCodecInfo.GetImageDecoders().FirstOrDefault(x => x.FormatID == image.RawFormat.Guid);
string fileMimeType = codec.MimeType;
```

##### Write byte array to file
```csharp
System.IO.File.WriteAllBytes("fileName", data); 
```

### Collections

##### Get value from dictionary
```csharp
string value = String.Empty;
dictionary.TryGetValue("Key", out value);
```

##### Add an element to IEnumerable 
```csharp
someIEnumerableOfString.Concat(new[]{ "new element" });
```

### Current week number
```csharp
DateTimeFormatInfo dtFormatInfo = DateTimeFormatInfo.CurrentInfo;
Calendar calendar = dtFormatInfo.Calendar;
int currentWeek = calendar.GetWeekOfYear(DateTime.Now, dtFormatInfo.CalendarWeekRule, DayOfWeek.Sunday);
```

### Sending mails
```csharp
public void SendMail(string subject, string body, List<string> recipients, string attachment = null)
{
   string recipientsString = string.Join(",", recipients);
   SendMail(subject, body, recipientsString, attachment);
}

public void SendMail(string subject, string body, string recipients, string attachment = null)
{
   MailMessage mail = new MailMessage();
   SmtpClient SmtpServer = new SmtpClient("203.204.205.206");
   mail.From = new MailAddress("j.smith@evilcorp.com");
   mail.Subject = subject;
   mail.SubjectEncoding = Encoding.UTF8;
   mail.Body = body;
   mail.To.Add(recipients);
   mail.BodyEncoding = Encoding.UTF8;
   mail.IsBodyHtml = true;

   if (attachment != null)
   {
      Attachment attachmentM = new Attachment(attachment);
      mail.Attachments.Add(attachment);
   }

   SmtpServer.Send(mail);
}
```

### Write to Windows Event Log
```csharp
using System.Diagnostics;

string eventSource = "MyApp";
string log = "Application";

if (!EventLog.SourceExists(eventSource))
   EventLog.CreateEventSource(eventSource, log);

EventLog eventLog = new EventLog();
eventLog.Source = eventSource;
eventLog.Log = log;
eventLog.WriteEntry("Test message", EventLogEntryType.Information);
```

### Get property value with reflection
```csharp
object GetAnyPropertyValue<T>(object obj, string propertyName)
{
   List<PropertyInfo> properties = typeof(T).GetProperties().ToList();
   PropertyInfo property = properties.First(x => x.Name == propertyName);

   if(property != null && obj != null)
   {
        return property.GetValue(obj);
   }
   
   return null;
}
```

## WCF

### WCF Test Client
If you develop WCF services, there is a simple but nice tool distributed toegether with Visual Studio 2017 to test WCF services. You can find it in the following path: `C:\Program Files (x86)\Microsoft Visual Studio 14.0\Common7\IDE`.

### Service config editor
->Right click on: Web.config (in WCF project) ->Edit WCF Configuration

### Service method returning JSON
If you want to create a service method that will return JSON, so the data can be easily accessed and consumed through AJAX you have to add **WebInvoke** attribute to method defintion in the contract like this:
```csharp
[OperationContract]
[WebInvoke(Method = "GET", ResponseFormat = WebMessageFormat.Json, BodyStyle = WebMessageBodyStyle.Wrapped")]
[return: MessageParameter(Name = "result")]
ResultDto GetData();
```
Then in **Web.config** you have to add endopoint with **webHttpBinging**.
```xml
<system.serviceModel>
    <services>
        <service name="App.Service.MainService">
           <endpoint address="../MainService.svc"
             binding="webHttpBinding"
             contract="App.Contracts.IMainService"
             behaviorConfiguration="webBehaviour" />
        </service>
    </services>
</system.serviceModel>
<behaviors>
    <endpointBehaviors>
          <behavior name="webBehaviour">
             <webHttp/>
          </behavior>
    </endpointBehaviors> 
</behaviors>
```
It is also a good idea to enable CORS requests, also in **Web.config**:
```xml
<system.webServer>
    <httpProtocol>
        <customHeaders>
           <add name="Access-Control-Allow-Origin" value="*" />
           <add name="Access-Control-Allow-Headers" value="Content-Type, Accept" />
        </customHeaders>
    </httpProtocol>
</system.webServer>
```

### Returning big amounts of data
Sometimes you may want to return a lot of data through service method, but the default maximum size may not allow you. To change it, in Web.config add this:
```xml
<system.serviceModel>
<bindings>
    <basicHttpBinding>
        <binding name="LargeBuffer" 
                 maxBufferSize="2147483647" 
                 maxReceivedMessageSize="2147483647">
            <readerQuotas maxDepth="2147483647"
                          maxStringContentLength="2147483647"
                          maxArrayLength="2147483647"
                          maxBytesPerRead="2147483647"
                          maxNameTableCharCount="2147483647" />
        </binding>
    </basicHttpBinding>
</bindings>
</system.serviceModel>
```
and then while defining endpoints assign this configuration with **bindingConfiguration** attribute. Be careful, if you want to use configuration for other type of endpoint binding than basic HTTP binding, you have to define them separately with correct markups (for example **webHttpBinding**).

### Many endpoints for one service
Sometimes it is required to allow to communicate with service through different protocols. To achieve this you can define several endpoints for the service. Here you can see a basic HTTP endpoint and web HTTP endpoint:
```xml
<system.serviceModel>
<services>
    <service name="App.Service.MainService">
        <!--- BASIC ENDPOINT -->
        <endpoint address="base"
                  binding="basicHttpBinding"
                  name="MainServiceBasicHttpEndpoint"        
                  contract="App.Contracts.IMainService" 
                  bindingConfiguration="LargeBuffer" />
   
        <!-- WEB ENDPOINT --> 
        <!-- http://localhost:55555/MainService.svc/web -->
        <endpoint address="web"
                  binding="webHttpBinding"
                  name="MainServiceWebHttpEndpoint"        
                  contract="App.Contracts.IMainService"
                  behaviorConfiguration="webBehaviour"
                  bindingConfiguration="WebLargeBuffer" />
    </service>
</services>
</system.serviceModel>
```

### Error details
While developing WCF services you may want to know the details of errors that happen. As default WCF doesn't show any details which is quite smart in the context of security. To allow to send error details add the following in **Web.config**:
```xml
<behaviors>
<serviceBehaviors>
    <behavior name="">
        <serviceMetadata httpGetEnabled="true" />
        <serviceDebug includeExceptionDetailInFaults="true">
    </behavior>
</serviceBehaviors>
</behaviors>
```

### WCF errors
WCF is quite advanced technology and you may encounter many different errors while working with it. Here are listed some of them with solution that may help.

##### Cannot find type "App.Service.Service1" provided as the Service attribute value in the ServiceHost directive could not be found
->Right click on the service file (svc) ->Open With ->Web Service Editor ->Make sure that Service and Code Behind attributes have correct values.

##### Access to the path 'xyz' is denied
If you see such error, probably WCF service is trying to save or modify a file in a path where it is not allowed. You have to grant write permission on the given directory for application pool to which the service is assigned:<br />
->Right click on directory ->Properties ->Security ->Edit ->Add ->Advanced ->Locations... ->Choose current machine ->Object name: IIS APPPOOL\MyServiceAppPool<br /><br />
In case if you couldn't find the correct application pool (actually you always should), you can add the permission for **IIS_IUSRS** group:<br />
->Right click on directory ->Properties ->Security ->Edit ->Add ->Advanced ->Locations... ->Choose current machine ->Find now: IIS_IUSRS<br /><br />
You can also use the command:
```
cacls '\\100.110.60.111\c$\inetpub\MyApp_Service\Files' /E /G BUILTIN\IIS_IUSRS:F
```
* /E - do not delete current permissions
* /G - grant (/R - revoke) 
* :F - full access (:N - none)

Read more: https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2008-R2-and-2008/cc726004(v=ws.11)

##### The target assembly contains no service types. 
->Right click on the project that contains WCF contracts ->Properties ->WCF Options ->Start WCF Service Host when debugging another project in the same solution.

### Problems with using statement
Here is a common way in which you could call some method of WCF service - with using statement:
```csharp
using(DataServiceClient client = new DataServiceClient())
{
     client.ExecuteMethod();
}
```
This is actually translated to this one under the hood:
```csharp
DataServiceClient client = new DataServiceClient();
try
{
    client.ExecuteMethod();
}
finally
{
    client.Close(); // here an exception can be thrown and it will NOT be catched
}
```
If an exception will be thrown in the **finally** block then your connection to WCF service will not be closed properly. The safe way of executing WCF methods looks like this:
```csharp
DataServiceClient client = new DataServiceClient()
try
{
    client.ExecuteMethod();
}
catch(CommunicationException)
{
    client.Abort();
}
catch(TimeoutException)
{
    client.Abort();
}
catch(Exception)
{
    client.Abort();
    throw;
}
```
You can read more about this [here](https://docs.microsoft.com/en-us/dotnet/framework/wcf/samples/avoiding-problems-with-the-using-statement).

### AutoMapper
AutoMapper is an extremely useful library that allows you to map objects in C# from one class to another. It is very useful in the context of WCF, for example when you get some objects through your ORM and you have to convert them to DTO (Data Contracts) objects to be able to return them from a service method. With AutoMapper such operations are quite easy to do.

##### Add AutoMapper to project
->Tools ->NuGet Package Manager ->Package Manager Console ->`install-package AutoMapper`
* AutomapperConfiguration.cs
```csharp
public class AutomapperConfiguration
{
    public static void Configure()
    {
        Mapper.Initialize(ctg =>
        {
            ctg.CreateMap<Customer, CustomerDTO>();
        });
    }
}
```
* AutomapServiceBehavior.cs
```csharp
public sealed class AutomapServiceBehavior : Attribute, IServiceBehavior
{
    public AutomapServiceBehavior(){ }

    public void AddBindingParameters
        (ServiceDescription serviceDescription, ServiceHostBase serviceHostBase,
         Collection<ServiceEndpoint> endpoints, BindingParameterCollection bindingParameters)
    {
        AutomapperConfiguration.Configure();
    }

    public void ApplyDispatchBehavior
        (ServiceDescription serviceDescription, ServiceHostBase serviceHostBase){ }

    public void Validate
        (ServiceDescription serviceDescription, ServiceHostBase serviceHostBase){ }
}
```
* Service.svc
```csharp
[AutomapServiceBehavior]
public class Service : IService { }
```

##### Modify properties while mapping
It is possible to define quite advanced mapping rules in AutoMapper configuration (inside of **Initialize** method). It is very common to set or modify values of some properties, see an example: 
```csharp
ctg.CreateMap<Employee, EmployeeDTO>()
    .ForMember(
        dest => dest.DateText, 
        opt => opt.MapFrom(src => src.Date.ToShortDateString())
    );
```

##### Collections
If you have the following mapping configuration:
```csharp
Mapper.Initialize(cfg => 
{ 
    cfg.CreateMap<Source, Dest>();
});
```
then you can also use mapped types with collections and arrays, so all the following expressions are correct:
```csharp
IEnumerable<Dest> dest = Mapper.Map<Source[], IEnumerable<Dest>>(sources);
ICollection<Dest> dest = Mapper.Map<Source[], ICollection<Dest>>(sources);
IList<Dest> dest = Mapper.Map<Source[], IList<Dest>>(sources);
List<Dest> dest = Mapper.Map<Source[], List<Dest>>(sources);
Dest[] dest = Mapper.Map<Source[], Dest[]>(sources);
```

##### Nested mappings
Nested mappings is a very important feature of AutoMapper. If a class internally stores an instance of another class, for example as a property or field, then if you will define mappings for the internally used class, AutoMapper will be able to map it toegether with the class that uses it.
```csharp
public class OuterSource{
     public InnerSource field1;
}

Mapper.Initialize(c=> {
     cfg.CreateMap<OuterSource, OuterDest>();
     cfg.CreateMap<InnerSource, InnerDest>();
});
```

### Fault contracts
Fault contracts are an elegant way of error handling in WCF. You can create a specified data contract that will be used for exceptions.
* DTO - Firstly you have to create a data contranct, the same way as a typical DTO
```csharp
[DataContract]
public class CustomFault 
{
    [DataMember]
    public string Message { get; set; }
    [DataMember]
    public string Code { get; set; }
}
```
* Service contract - Mark a method with **FaultContract** attribute
```csharp
[OperationContract]
[FaultContract(typeof(CustomFault))]
void RunMethod();
```
* Service - Catch an exception and throw **FaultException** of your DTO 
```csharp
public void RunMethod()
{
    try 
    {
        DoSomething();
    }
    catch(Exception)
    {
        CustomFault fault = new CustomFault();
        fault.Code = "XB10";
        fault.Message = "Unknown error";
        throw new FaultException<CustomFault>(fault);
    }
}
```

## Performance and under the hood
Performance is often a very important factor. But to be able to optimize and speed up something it is crucial to know how C# runtime and the language itself work. Also keep in mind that you should optimize stuff only when it is really necessary. If something works pretty fast there is no point in waisting your time trying to gain few miliseconds speed up unless you develop some real time software of a game.

### Measuring performance
The most basic thing if you want to optimize a code is to measure its performance. You cannot do some real optimization if you cannot measure how long a code executes. There quite many ways how to measure performance, here you will see few basic easy to implement methods.

##### Method 1
```csharp
DateTime start = DateTime.UtcNow;

// some code to measure

DateTime stop = DateTime.UtcNow;
int measuredTime = (stop - start).TotalMilliseconds;
```
It is a quite fast method, but with low accuracy - requires many loops over measured code. May give unrealiable results if system time changes while measurements are taken.

##### Method 2
```csharp
TimeSpan start = Process.GetCurrentProcess().TotalProcessorTime;

// some code to test

TimeSpan stop = Process.GetCurrentProcess().TotalProcessorTime;
int measuredTime = (stop - start).TotalMilliseconds;
```
Slow method, but with good accuracy, it uses different mechanism - measures how long CPU was busy, not how much time passed. Important things to note are that time inside sleep is not counted, but excution on two CPUs is counted as 200%!

##### Method 3
```csharp
Stopwatch watch = new Stopwatch();
watch.Start();

// some code to test

watch.Stop();
int measuredTime = watch.Elapsed.TotalMilliseconds;
```
This method has a very good accuracy and is quite common. Two methods can be very useful while working with **Stopwatch** class: 
* **Reset** - stops and sets the result to 0
* **Restart** - stops, sets the result to 0 and starts again

### GC object's generations
Garbage Collector puts every object in one of 3 levels called generations. Generation 0 holds small short living objects. It is the fastest generation, but it is also the smallest one. Generation 2 is the slowest, but also the biggest one. It holds big objects that are supposed to be used many times in the application lifetime. Generation 1 is something in between of generatons 0 and 2. If you want to know in which generation is given object hold, use the following code:
```csharp
byte[] data = new byte[90000]; 
GC.GetGeneration(data); //returns 0, 1 or 2 [here 2]
```
Large objects, over 85000 bytes are stored in Large Object Heap (LOH), which is not divided by generations.

### Fast binary string comparison
```csharp
String.Equals("test", "TEST", StringComparison.OrdinalIgnoreCase);
```

### Decimal vs. double
**Double** and **float** store numbers in the base of 2, while **Decimal** stores numbers in the base of 10. Double and float are much faster because the processor can work on them directly. On the other hand Decimal is much more accurate in calculations.

### Reference and value types
Objects are given as reference by default. Value types and structs are given by value, **ref** and **out** keywords have any meaning only when they are used with them.<br />
Value types cannot be modified. Every time when some kind of modification is tried to be done, a new new value is created instead. That's why for example if you want to modify some string many times, it is recomended to use **StringBuilder** class. See an example:
```csharp
//in such little example it is ok, but for much more modifications it is not a good way   
string tmp1 = "a" + "b" + "c"; 

StringBuilder tmp2 = new StringBuilder();
tmp2.append("a").append("b").append("c");
```
In case of reference types it is important to remember that their values are not copied, but they reference are. In the following example both `tmp1` and `tmp2` are references to **the same object**:
```csharp
List<int> tmp1;
List<int> tmp2 = tmp1; 
```

### Performance tips
* Avoid boixng and uboxing (casting variables to **object**) - use `int[]` and `List<int>` instead of **List** and **ArrayList**
* Use **StringBuilder** - it uses the pointer to the string internally and does not create new string every time you try to modify it.
* Array of int (`int[]`) is faster than list of int (`List<int>`) and faster than anything else actually.
* For loop is faster than foreach loop while iterating lists (but both are very fast for `int[]`).
* Do NOT use two-dimensional arrays. Instead flatten the array. Instead of indexing the array like this `[x, y]`, you will have to this like this `[x * columns + y]`. If you cannot flatten the array (probably always you can) better use jagged arrays than two-dimensional.
* If it is possible avoid throwing exceptions and never catch general exceptions of **Exception** class.
* Use `int.TryParse` instead of `int.Parse` - **TryParse** doesn't throw exceptions.
* Use many small and short living objects, don't let them to be put into 1 and 2 generation of GC and use few large and long living objects that will be put on large objects heap. This way you will go along with Garbage Collector strategy and take advantage of it.

### Intermediate language

Programs written in C# are translated to [**Common Intermediate Language** (CIL)](https://en.wikipedia.org/wiki/Common_Intermediate_Language). This is a language which looks like Assembler a bit with and it is then executed by the .NET Runtime. That's why C# applications can be portable because if you have .NET Runtime installed on your operating system you will be able to run C# program. It is not mandatory to know CIL and its instructions but it is quite advisable to at least understand some most basic concepts of how C# programs look after compilation to CIL.<br />
Here you can see some most basic instructions (this is not an example of a program):
```
ldc.i4.1       //load constant 1 as integer 32bit and put it on stack
ldloc.0        //load value from variable location 0 and put it on stack
stloc.1        //take a value from the top of stack and store it in variable location 1
add            //add two top values on stack
bne #000000a   //branch not equal - go to given instruction if two top values on stack are not equal
br #000000a    //branch to given instruction
```
And here you can see an example.<br/>
C# code:
```csharp
array[5] = 10;
```
CIL code:
```
ldloc.0     /// array
ldc.i4.5    /// [5]
ldc.i4.s 10 /// 10
stelem.i4   /// =
```
[Here](https://en.wikipedia.org/wiki/List_of_CIL_instructions) you can see the whole list of instructions. You can decompile .NET programs with **ildasm.exe**. You can find it usually in the following path: `C:\Program Files (x86)\Microsoft SDKs\Windows\v8.1A\bin\NETFX 4.5.1 Tools` and [here](https://docs.microsoft.com/en-us/previous-versions/dotnet/netframework-2.0/f7dy01k1(v=vs.80)) you can find the docs.

## Other

### NLog
NLog (or other logging utility) allows you to write logs of your application to be able to analyze the way it behaves, find bugs or quickly troubleshoot issues. It is easy to use and install - you can download it from NuGet. [Here](https://github.com/abik11/Dev-Suite/blob/master/Dev/Merge-NLogs.psm1) is a little Powershell script that helps analyzing logs generated by NLog.

##### Example of NLog configuration
NLog.config:
```xml
<?xml version="1.0" encoding="utf-8" ?>
<nlog xmlns="http://www.nlog-project.org/schemas/NLog.xsd"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:schemaLocation="http://www.nlog-project.org/schemas/NLog.xsd NLog.xsd"
      autoReload="true"
      throwExceptions="false"
      internalLogLevel="Off" internalLogFile="c:\temp\nlog-internal.log">

   <targets>
      <target name="traceLog"
         xsi:type="File"
         fileName="${basedir}/logs/trace.txt"
         layout="APP_LOGIC|${longdate}|${level}|${callsite}|${machinename} ${message}"
         archiveFileName="${basedir}/logs/trace.{#}.txt"
         archiveEvery="Day"
         archiveNumbering="Rolling"
         maxArchiveFiles="10" />

      <target name="errorLog"
         xsi:type="File"
         fileName="${basedir}/logs/error.txt"
         layout="APP_LOGIC|${longdate}|${level}|${callsite}|${machinename} ${message}"
         archiveFileName="${basedir}/logs/error.{#}.txt"
         archiveEvery="Month"
         archiveNumbering="Rolling"
         maxArchiveFiles="50" />
   </targets>

   <rules>
      <logger name="*" minlevel="Trace" writeTo="traceLog" />
      <logger name="*" minlevel="Error" writeTo="errorLog" />
   </rules>
</nlog>
```
The simplest way to add NLog to a class:
```csharp
protected static NLog.Logger logger = NLog.LogManager.GetCurrentClassLogger();
//logger.Trace(); logger.Error(); etc...
```
it is a good idea to use dependency injection and inject **NLog.Logger** class - see [here](https://ludwigstuyck.wordpress.com/2013/03/04/avoiding-the-use-of-static-variables-when-using-log4net/).

##### NLog is not logging
If you have configured NLog and it is not logging nothing at all the first step you should do is to change the value of **internalLogLevel** from **Off** (`internalLogLevel="Off"`) to **Debug** (`internalLogLevel="Debug"`) and set **internalLogFile** where NLog will write its own logs (`internalLogFile="c:\temp\nlog-internal.log"`).<br />
Quite often the problem is about the permissions to a given path to which NLog is supposed to write, but if it is something else you will be able to find answer from NLog internal logs.

### Colors in .Net
In general colors in .Net are represented by **Color** class form **System.Drawing** namespace. They consist of four 8 bit numbers representing transparency (alpha), red, green and blue and the whole color can be represented simply by a 32 bit integer value. For example *-5658199* is some kind of grey. To convert it to hex notation or ARGB you can use Powershell. **FromArgb** static method of **Color** class returns an object of type **Color** with all the color details.
```powershell
Add-Type -AssemblyName System.Drawing
$color = [System.Drawing.Color]::FromArgb(-5658199)
$color.Name
```
This integer is nothing more than just color written as ARGB in decimal notation instead of hexadecimal, the one that we are all used to when it comes to dealing with colors. You can see the hex notation accessing **Name** property of color object. Building a new color is quite easy also if you know red, green and blue elements. You just have to use **ToArgb** method to get the color representation as decimal integer:
```powershell
$color = [System.Drawing.Color]::FromArgb(216, 216, 216)
$color.ToArgb()
```

## Useful links

##### General stuff
[Ventajas de yield return - ESP](https://bmegias.wordpress.com/2010/11/10/que-es-yield-y-por-que-hay-que-usarlo/)<br/>

##### Online tools
[.Net error translator 1](http://finderr.net/)<br />
[.Net error translator 2](http://www.errortoenglish.com/)<br />
[Regex editor](http://regexr.com/)<br />
[Barcode generator](http://barcode.tec-it.com/en)<br />

##### WCF
[WCF Web Services Tutorial](http://mikesknowledgebase.azurewebsites.net/pages/Services/WebServices.htm)<br />
[Hosting WCF Services](https://docs.microsoft.com/en-us/dotnet/framework/wcf/hosting-services)<br />
[Hosting and consuming WCF Services](https://msdn.microsoft.com/en-us/library/bb332338.aspx)<br />
[WCF project's structure](https://www.codemag.com/article/0809101)<br />
[AutoMapper](https://github.com/AutoMapper/AutoMapper/wiki)<br />
