# API - Compression for IIS or NodeJS

IIS and NodeJS do not enable HTTP response compression by default.  Enabling compression can reduce the total amount of bytes sent in the response improving network bandwidth utilization and any associated bandwidth charges.  This can be especially beneficial for HTML, JavaScript not minified at compile time, or JSON responses where whitespace removal is not already enabled.

## Enabling on IIS Express for Local Development Testing

1. Start command prompt and go to IIS Express installation folder `%PROGRAMFILES%\IIS Express` and run following command:

```cmd
appcmd set config -section:urlCompression /doDynamicCompression:true
```

2. To add compression for JSON run the following two commands from the IIS Express installation directory:

```cmd
appcmd set config /section:staticContent /+[fileExtension='.json',mimeType='application/json']
appcmd.exe set config -section:system.webServer/httpCompression /+"dynamicTypes.[mimeType='application/json',enabled='True']" /commit:apphost
```

3. Restart IIS Express.

## Enabling on Remote IIS Server

You can enable GZIP compression entirely in the Web.config file as shown below

```xml
<system.webServer>
   <httpCompression directory="%SystemDrive%\inetpub\temp\IIS Temporary Compressed Files">
      <scheme name="gzip" dll="%Windir%\system32\inetsrv\gzip.dll"/>
      <dynamicTypes>
         <add mimeType="text/*" enabled="true"/>
         <add mimeType="message/*" enabled="true"/>
         <add mimeType="application/javascript" enabled="true"/>
         <add mimeType="*/*" enabled="false"/>
      </dynamicTypes>
      <staticTypes>
         <add mimeType="text/*" enabled="true"/>
         <add mimeType="message/*" enabled="true"/>
         <add mimeType="application/javascript" enabled="true"/>
         <add mimeType="*/*" enabled="false"/>
      </staticTypes>
   </httpCompression>
   <urlCompression doStaticCompression="true" doDynamicCompression="true"/>
</system.webServer>
```

## Enabling on NodeJS

1. For file named `package.json` deployed with the NodeJS `server.js`, add the NPM package named `compression` 
2. In the file named `server.js`, add the following code:

```javascript
// GZIP/Deflate Middleware
var compression = require('compression');
app.use(compression());
```

## References
* [Enabling Dynamic Compression on IIS](http://www.hanselman.com/blog/EnablingDynamicCompressionGzipDeflateForWCFDataFeedsODataAndOtherCustomServicesInIIS7.aspx)
* [GZIP response on IIS Express](http://stackoverflow.com/questions/10102743/gzip-response-on-iis-express)
* [Enable IIS 7 GZIP](http://stackoverflow.com/questions/702124/enable-iis7-gzip)

