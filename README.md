#KT_WSO2_PHPCPP

WSO2 Web services Framework (WSF) for PHP (client only) based on PHP-CPP. This is under developpement and can not be used for production. Lot of features are not yet backported due to lack of time and testing scenarii.

**Important** : please keep in mind i'm not a C/C++ dev and there is certainly a lot of ugly things that must be fixed and/or refactored. ( PR are welcome to help and fix ) 

The original WSO2 WSF for PHP extension was a abstract layer for Axis2/c with some additional ( actually optional from axis2 perspective ) modules such as Rampart/c (WS-SEC), Sandesha2/c (WS-RM ) etc. 

In addtion to this layer, WSO2 developped additional PHP scripts handling serialize/unserialize ( when users want works in WSDL). I did not managed nor will do any backport for the old fashion scripts. 

If you plan to work w/ Object, I suggest to use JMS Serializer and pass the serialized Objects to the `WSMessage::setPayload` method. However, i will investigate how to implement JMS serializer in a while.

The API is refactored but is "almost" similar to the native WSO2 extension. I will try to document it ASAP. 

SoapFault are not thrown from a KTWS\WSFault object since we are waiting for custom exceptions support in PHP-CPP. This could be normally compiled for PHP7 through the master branch of PHPCPP. I only tested for Php 5.6.x under Ubuntu 14.04-LTS w/ the PHPCPP-LEGACY. 

## Installation

####Preparing the build env

```
sudo apt-get install libssl-dev build-essential libtool automake pkgconf g++-4.4 g++-4.8 libjson0-dev libboost-dev 

sudo mkdir -p /opt/build && cd /opt/build
sudo chown agruet:agruet . ( <- change by your username:group )
```

### PHP 5.6 - Ubuntu 

```
sudo LC_ALL=en_US.UTF-8 add-apt-repository ppa:ondrej/php
sudo apt-get update
sudo apt-get install php5.6 php5.6-dev php-config
```

####Axis2/c & co

*TODO: maven / PPA Launchpad*
```
git clone https://github.com/CopernicaMarketingSoftware/PHP-CPP-LEGACY.git
git clone https://github.com/alexis-gruet/kt_wso2-phpcpp-client.git
git clone https://github.com/alexis-gruet/axis2c-trunk.git
git clone https://github.com/alexis-gruet/kt_rampart.git
git clone https://github.com/alexis-gruet/kt_sandesha2.git
git clone https://github.com/alexis-gruet/kt_savan.git

mv PHP-CPP-LEGACY php-cpp-1.5.4
cd axis2c-trunk

export CC="gcc-4.8"

./autogen.sh
./configure --prefix=/usr/local/axis2c --enable-openssl=yes --with-openssl=/usr/include/openssl --disable-libxml2 --enable-guththila=yes --enable-json=yes --enable-trace=yes --enable-tcp --with-json=/usr/include/json-c
make && sudo make install

cd ../kt_rampart
export CC="gcc-4.4"

./autogen.sh
 ./configure --with-openssl=/usr/include/openssl --with-axis2=/usr/local/axis2c/include/axis2-1.7.0 --prefix=/usr/local/axis2c 
 make && sudo make install
 
 cd ../kt_sandesha2
 
 ./autogen.sh
 ./configure --with-axis2=/usr/local/axis2c/include/axis2-1.7.0 --prefix=/usr/local/axis2c
 make && sudo make install
 
 cd ../kt_savan
 
 ./autogen.sh
 ./configure --with-axis2=/usr/local/axis2c/include/axis2-1.7.0 --prefix=/usr/local/axis2c 
 make && sudo make install
 
 cd ../kt_wso2-phpcpp-client
 sudo cp conf/axis2.xml /usr/local/axis2c/
 sudo cp conf/modules/sandesha2/module.xml /usr/local/axis2c/modules/sandesha2
 sudo cp conf/modules/rampart/module.xml   /usr/local/axis2c/modules/rampart
 
 sudo bash 
 sudo cp conf/ktwso2.conf /etc/ld.so.conf.d
 sudo ldconfig
 exit
```
When running the `ldconfig` command you will notice of link expected... it must be fixed in the sandesha2 makefile. 

### PHP-CPP

This is a dependency for the PHP Extension

(we are always under /opt/build/kt_wso2-phpcpp-client)
```
cd ../
cd php-cpp-1.5.4

export CC="g++-4.8"

make && sudo make install
```

### KT_WSO2_PHPCPP 

```
cd ../kt_wso2-phpcpp-client

make && make install

```

voilà!
```
php --ri ktwso2 

------
ktwso2

Version => 0.0.1

Directive => Local Value => Master Value
ktwso2.axis2_directory => /usr/local/axis2c => /usr/local/axis2c
ktwso2.log_path => /tmp => /tmp
ktwso2.log_filename => client.log => client.log
ktwso2.log_level => 4 => 4
------
```

####If you installed PHP by a different way

Line 32 (INI_DIR) you must fill the full path for the .ini file. Using the PPA for PHP ( above ) path was  /etc/php/5.6/cli/conf.d

Final step, type `php-config --extension-dir` in the terminal and copy the full path in the `Makefile` line 44 ( EXTENSION_DIR )




## API

###Objects
```
$WSClient   = new KTWS\WSClient;
$WSMessage  = new KTWS\WSMessage;
$WSHeader   = new KTWS\WSHeader;
$WSHeader1  = new KTWS\WSHeader;
$WSSecToken = new KTWS\WSSecurityToken;
$WSPolicy   = new KTWS\WSPolicy;
```

### WSHeader

Soap header can be defined by a KTWS\WSHeader instance, it support nested headers through 
the setData() method. 

Soap Header(s) must be set by `WSMessage::setHeaders()` method, which support a array of unbound 
headers

```
$WSHeader  = new KTWS\WSHeader;
$WSMessage = new KTWS\WSMessage;

$WSHeader
->setNs("myNamespace")
->setPrefix("myPrefix")
->setName("MyHeaderName")
->setMustUnderstand(true)

//Int as the role : 0,1,3 <- check the WSO2 API
->setRole(1) 

Array of unbound KTWS\Header or string or xml
->setData();

$WSMessage->setHeaders([$WSHeader, $WSHeader1, $WSHeader2]);
```

According to your definition, Axis2 will produce for example a log similar to : 

```
<?xml version="1.0" encoding="UTF-8"?>
<myHeader:name xmlns:myHeader="header0" xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" soapenv:mustUnderstand="1" soapenv:actor="http://www.w3.org/2003/05/soap-envelope/role/none">
   <myHeader1:theHeader1 xmlns:myHeader1="header1" soapenv:actor="http://www.w3.org/2003/05/soap-envelope/role/none">data1</myHeader1:theHeader1>
   <myHeader2:theHeader2 xmlns:myHeader2="header2" soapenv:actor="http://www.w3.org/2003/05/soap-envelope/role/none">data2</myHeader2:theHeader2>
   <myHeader3:theHeader3 xmlns:myHeader3="header3" soapenv:actor="http://www.w3.org/2003/05/soap-envelope/role/none">data3</myHeader3:theHeader3>
   <myHeader4:theHeader4 xmlns:myHeader4="header4" soapenv:actor="http://www.w3.org/2003/05/soap-envelope/role/next">
      <ns4:theHeader5 xmlns:ns4="header5" soapenv:actor="http://www.w3.org/2003/05/soap-envelope/role/none">data5</ns4:theHeader5>
      <ns5:theHeader6 xmlns:ns5="header6">data6</ns5:theHeader6>
   </myHeader4:theHeader4>
</myHeader:name>
```

### Policy
```
$policy_xml = <<<XML
<wsp:Policy>
    ... OASIS
</wsp:Policy>
XML;

$WSPolicy->setXMLPolicy($policy_xml);
```

###Security Token 
`KTWS\SecurityToken` and `KTWS\WSPolicy` provide a nice API to deal with WS-Security. WS-Security relay heavily on Rampart/c and neethi/c, both are modules for axis2/c 

Example for Username Token 
```
$WSSecToken
->setUser("username")
->setPassword("password");
->setPasswordType("PlainText || Digest")
```

The Digest passwordtype will add a mustUnderstand=1 in SoapHeader, hence response must have a well-formed soapHeader. There is a workaround if response does not have 
a soapHeader, rampart/module.xml, disable InFlow... 

In order to use this scenario, you must implement a valid Policy through `KTWS\WSPolicy`

```
$my_policy = <<<XML
<wsp:Policy wsu:Id="RmPolicy" xmlns:wsp="http://schemas.xmlsoap.org/ws/2004/09/policy" xmlns:wsrm="http://schemas.xmlsoap.org/ws/2005/02/rm/policy" xmlns:sp="http://docs.oasis-open.org/ws-sx/ws-securitypolicy/200702" xmlns:sanc="http://ws.apache.org/sandesha2/c/policy" xmlns:wsu="http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-wssecurity-utility-1.0.xsd">
    <wsp:ExactlyOne>
        <wsp:All>
	<sp:TransportBinding>
                <wsp:Policy>
                </wsp:Policy>
            </sp:TransportBinding>
            <sp:SignedSupportingTokens>
                <wsp:Policy>
                    <sp:UsernameToken
                        sp:IncludeToken="http://docs.oasis-open.org/ws-sx/ws-securitypolicy/200702/IncludeToken/AlwaysToRecipient">
                        <wsp:Policy>
                            <sp:WssUsernameToken10 />
                        </wsp:Policy>
                    </sp:UsernameToken>
                </wsp:Policy>
            </sp:SignedSupportingTokens
        </wsrm:RMAssertion>
        </wsp:All>
    </wsp:ExactlyOne>
</wsp:Policy>
XML;

$WSPolicy->setXMLPolicy($my_policy);
```

###WSFault

Since PHPCPP does not provive custom exception, hence this object could not be catchable by PHP. However 
some helpers are implemented such as `WSClient::hasSoapFault` identifing if the Soap request was fault.

getter are pretty clear :  

```
try {

	$WSClient->request();
	
	$WSClient->getMessage()->getResponse();
		-or-
	$WSClient->getResponse();
	
} catch(\Exception $e) {
	
	if($WSClient->hasSoapFault())
	{
		$WSFault = $WSClient->getSoapFault();
		
		$WSFault->getXMLNode();
		$WSFault->getCode();
		$WSFault->getRole();
		$WSFault->getReason();
		$WSFault->getDetails();
			
			-or-
		
		echo $e->getMessage(); which give the raw xml node
	} 
	else 
	{
		//Std exception
	}
}
```

###WSMessage

WSMessage handle the SOAP message that must be sent. 

`WSMessage::setPayload [string]` : expect a string as the soap xml payload *without* specifying the SoapEnvelope, SoapHeaders nor SoapBody ; This is handled internally by Axis2/c and by the parameters you provide through `WSPolicy`, `WSSecurityToken` or `WSHeaders`. You only focus on the business logic by serializing your objects.  

`WSMessage::setEndpoint` expect a string as the endpoint where reside the soap contract. 

Basic usage : 
```
$WSMessage = KTWS\WSMessage;

$WSMessage
->setEndpoint("http(S)://endpoint")
->setPayload("<ws:calculate xmlns:ws=""></ws:calculate>");
```

WS-Adressing is backported, but not yet tested on my side. 
To get the full documentation, please check the official WSO2 Documentation


```
$WSMessage = KTWS\WSMessage;

$WSMessage
->setEndpoint("http(S)://endpoint")
->setPayload("<ws:calculate xmlns:ws=""></ws:calculate>");

->setAction("remoteAction")
->setFrom([
	'address' => "http://.....",
	
	//Array of unbound wsa Endpoint reference  
	'referenceParameters' => [
		
		//
	    	'<wsa:EndpointReference xmlns:wsa="..." xmlns:fabrikam="...">
   		 	<wsa:Address>http://www.fabrikam123.example/acct</wsa:Address>
   		 	<wsa:PortType>fabrikam:InventoryPortType</wsa:PortType>
		</wsa:EndpointReference>',
		
		//
		'<wsa:EndpointReference xmlns:wsa="..." xmlns:fabrikam="...">
   		 	<wsa:Address>http://www.fabrikam123.example/acct</wsa:Address>
   		 	<wsa:PortType>fabrikam:InventoryPortType</wsa:PortType>
		</wsa:EndpointReference>'
		
	],
	//Array of unbound wsa metadatas
	'metadatas' => [
	]
])
```
This is the same for ReplyTo and Fault : 

```
->setReply([ same as above ])
->setFault([ still the same ])
```

One more thing regarding WS-Adressing, in order to use it you must set, at the KTWS\WSClient level, 
the following method :

```
$WSClient->setWSA([string])
```
This method expect : 

- "1.0"
- "submission"
- "disabled"

###WSClient
```
$WSClient

//Soap version 1.1 or 1.2
->setSoapVersion(1.2)

//Work w/ REST
->disableSoap()

//
->setMessage($WSMessage)
->setSecToken($WSSecToken)
->setPolicy($WSPolicy)

//Proxy
->setProxyHost("127.0.0.1")
->setProxyPort("8080")
->setProxyUsername("proxyUsername")
->setProxyPassword("proxyPassword")
->setProxyAuthType("proxyAuth")

//HTTP Auth
->setHTTPUsername("username")
->setHTTPPassword("pasword")
->setHTTPAuth("Basic");

//Timeout in sec
->setTimeout(30)

//Mandatory if working w/ SSL
->setSSLServerCert("Absolute path to SSL CA");
```

###Requesting

```
try {
	$WSClient->request();
	
	print_r($WSMessage->getResponse());
	
	//print_r($WSClient->getMessage());

} catch(\Exception $e) {
	echo $e->getMessage();
}
```

###WS-RM / Reliable Messaging

Work in progress, Sandesha2/c is loaded by default. I successfully created a two-way channel with a successfull CreateSequence : 

```
<?xml version="1.0" encoding="UTF-8"?>
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/">
   <soapenv:Header xmlns:wsa="http://www.w3.org/2005/08/addressing">
      <wsa:To>http://localhost:2080</wsa:To>
      <wsa:Action>http://schemas.xmlsoap.org/ws/2005/02/rm/CreateSequence</wsa:Action>
      <wsa:ReplyTo>
         <wsa:Address>http://www.w3.org/2005/08/addressing/anonymous</wsa:Address>
      </wsa:ReplyTo>
      <wsa:MessageID>urn:uuid:b13cec5a-4868-1e61-22c6-000c290a6b1d</wsa:MessageID>
      <wsse:Security xmlns:wsse="http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-wssecurity-secext-1.0.xsd" soapenv:mustUnderstand="1" />
   </soapenv:Header>
   <soapenv:Body>
      <wsrm:CreateSequence xmlns:wsrm="http://schemas.xmlsoap.org/ws/2005/02/rm">
         <wsrm:AcksTo>
            <wsa:Address xmlns:wsa="http://www.w3.org/2005/08/addressing">http://www.w3.org/2005/08/addressing/anonymous</wsa:Address>
         </wsrm:AcksTo>
         <wsrm:Offer>
            <wsrm:Identifier>b13d1338-4868-1e61-22c7-000c290a6b1d</wsrm:Identifier>
         </wsrm:Offer>
      </wsrm:CreateSequence>
   </soapenv:Body>
</soapenv:Envelope>
```

the policy used is : 

```
<?xml version="1.0" encoding="UTF-8"?>
<wsp:Policy xmlns:wsp="http://schemas.xmlsoap.org/ws/2004/09/policy" xmlns:sanc="http://ws.apache.org/sandesha2/c/policy" xmlns:sp="http://docs.oasis-open.org/ws-sx/ws-securitypolicy/200702" xmlns:wsrm="http://schemas.xmlsoap.org/ws/2005/02/rm/policy" xmlns:wsu="http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-wssecurity-utility-1.0.xsd" wsu:Id="RmPolicy">
   <wsp:ExactlyOne>
      <wsp:All>
         <sp:TransportBinding>
            <wsp:Policy />
         </sp:TransportBinding>
         <wsrm:RMAssertion>
            <wsrm:InactivityTimeout Milliseconds="600000" />
            <wsrm:AcknowledgementInterval Milliseconds="200" />
            <wsrm:BaseRetransmissionInterval Milliseconds="2" />
            <wsrm:ExponentialBackoff />
            <sanc:InactivityTimeout>64</sanc:InactivityTimeout>
            <sanc:StorageManager>persistent</sanc:StorageManager>
            <sanc:MessageTypesToDrop>none</sanc:MessageTypesToDrop>
            <sanc:MaxRetransCount>4</sanc:MaxRetransCount>
            <sanc:SenderSleepTime>1</sanc:SenderSleepTime>
            <!--In seconds-->
            <sanc:InvokerSleepTime>1</sanc:InvokerSleepTime>
            <sanc:PollingWaitTime>4</sanc:PollingWaitTime>
            <sanc:TerminateDelay>4</sanc:TerminateDelay>
         </wsrm:RMAssertion>
      </wsp:All>
   </wsp:ExactlyOne>
</wsp:Policy>

```


#Credits 
 - Alexis Gruet
 - WSO2 team
 - PHP-CPP team
