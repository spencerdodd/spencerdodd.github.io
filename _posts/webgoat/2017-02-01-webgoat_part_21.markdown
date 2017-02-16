---
layout: post
title:  "OWASP BWA WebGoat Challenge: Web Services"
subtitle: "Create a SOAP Request"
date:   2017-02-02 11:00:00 -0500
author: "coastal"
header-img: "images/site-resources/webgoat-header.jpg"
---
# Create a SOAP Request
Instructions:

- Web Services communicate through the use of SOAP requests. These requests are submitted to a web service in an attempt to execute a function defined in the web service definition language (WSDL). Let's learn something about WSDL files. Check out WebGoat's web service description language (WSDL) file. 
- Try connecting to the WSDL with a browser or Web Service tool. The URL for the web service is: http://localhost/WebGoat/services/SoapRequest The WSDL can usually be viewed by adding a ?WSDL on the end of the web service request. You must access 2 of the operations to pass this lesson.

From [soapuser.com][soap-user]:
SOAP is an XML-based messaging protocol. It defines a set of rules for structuring messages that can be used for simple one-way messaging but is particularly useful for performing RPC-style (Remote Procedure Call) request-response dialogues.

Let's navigate to the WSDL file and see how it is formatted. Here we see that it is an XML tree of actions and definitions bound to the ```targetNamespace``` of ```http://localhost:8080/WebGoat/services/SoapRequest```. It also looks absolutely unreadable. Instead of diving right into source without knowing what we're reading, how about we break SOAP down a little bit.

<center><img src="{{ site.baseurl }}/images/webgoat/2017-02-01-webgoat_part_21/soap-protocol.gif"></center>

So our application takes data, serializes it into a SOAP/XML format, wraps it in an HTTP request, and sends it to a waiting SOAP server. Then we receive an HTTP-wrapped SOAP/XML response, deserialize, and read the response data.

Let's look at a generic SOAP request-response chain and see if we can get some insight on how to format our data. Let's take a hypothetical service that is hosted at ```http://www.soapservice.org/MySoapServices```. i.e. if we appended ```?WSDL``` to that URL, we could see the WSDL page that defines our server's SOAP architecture. Let's look at a request and response for a SOAP function ```doubleAnInteger```:
<hr>
#### Request:
Pseudo-code:

```
doubleAnInteger(123)
```

SOAP:

```
<?xml version="1.0" encoding="UTF-8" standalone="no" ?>
  <SOAP-ENV:Envelope
   SOAP-ENV:encodingStyle="http://schemas.xmlsoap.org/soap/encoding/"
   xmlns:SOAP-ENV="http://schemas.xmlsoap.org/soap/envelope/"
   xmlns:SOAP-ENC="http://schemas.xmlsoap.org/soap/encoding/"
   xmlns:xsi="http://www.w3.org/1999/XMLSchema-instance"
   xmlns:xsd="http://www.w3.org/1999/XMLSchema">
	<SOAP-ENV:Body>
		<ns1:doubleAnInteger
		 xmlns:ns1="urn:MySoapServices">
			<param1 xsi:type="xsd:int">123</param1>
		</ns1:doubleAnInteger>
	</SOAP-ENV:Body>
  </SOAP-ENV:Envelope>
```

<hr>
#### Response
Pseudo-code:

```
246
```

SOAP:

```
<?xml version="1.0" encoding="UTF-8" ?>
  <SOAP-ENV:Envelope
   xmlns:SOAP-ENV="http://schemas.xmlsoap.org/soap/envelope/"
   xmlns:xsi="http://www.w3.org/1999/XMLSchema-instance"
   xmlns:xsd="http://www.w3.org/1999/XMLSchema">
	<SOAP-ENV:Body>
		<ns1:doubleAnIntegerResponse
		 xmlns:ns1="urn:MySoapServices"
		 SOAP-ENV:encodingStyle="http://schemas.xmlsoap.org/soap/encoding/">
			<return xsi:type="xsd:int">246</return>
		</ns1:doubleAnIntegerResponse>
	</SOAP-ENV:Body>
  </SOAP-ENV:Envelope>
```

Now we can break down the SOAP/XML a further and look at each element. The ```<?xml ?>``` header defines what version of XML we are using and the text encoding for the XML. Next we have the ```<SOAP-ENV:Envelope``` tag. This specifies parameters of the SOAP architecture including the ```SOAP-ENV:encodingStyle``` tag which defines the encoding schema for the envelope itself. The other envelope elements (prepended with ```xmlns``` define other aspects of the message schema) are standard identifiers in SOAP envelopes. The meat of our specific action is in the body.

The body of the message is wrapped in ```<SOAP-ENV:Body></SOAP-ENV:Body>``` tags. Next, the method that we wish to interact with is defined as ```<ns1:doubleAnInteger``` (which I read as namespace 1). we then define what ```ns1``` is with the line ```xmlns:ns1="urn:MySoapServices"```, where the urn is the URL location of the SOAP services (aka namespace) and ```MySoapServices``` is the name of the SOAP service handler operation. Now that we know what method we want to use, we pass it any parameters it needs. In this case we pass it ```<param1 xsi:type="xsd:int">123</param1>```. We have defined the first parameter to be passed as ```param1``` with a type of ```xsd:int``` or int, and have passed the value 123. This should match the definition of the method that is defined on the WSDL page for our service.

The response is defined very similarly, but the method name is ```doubleAnIntegerResponse``` and contains the return value for our input.

So now that we understand how SOAP requests and responses are formed, let's write a python script that interacts with our SOAP service at ```http://192.168.56.101/WebGoat/services/SoapRequest``` and interact with two of the methods ```getFirstName``` and ```getLastName```, which take in a user_id number (int) and return the first and last name (string) of the user respectively:

{% highlight python %}
import sys
import requests
import lxml.etree as etree

def send_soap():
	soap_url = "http://192.168.56.101/WebGoat/services/SoapRequest?WSDL"
	headers={
		'content-type': 'application/soap+xml',
		"SOAPAction":""
	}
	cookies = dict(
		JSESSIONID="D49B1C7B6EE2243929D320213C6A9C1B",
		acopendivids="swingset,jotto,phpbb2,redmine",
		acgroupswithpersist="nada"
	)
	for method in [getFirstNameRequest, getLastNameRequest]:
		body = method(101)
		r = requests.post(soap_url, data=body, headers=headers, cookies=cookies)
		print etree.tostring(etree.fromstring(r.content), pretty_print=True)


def getFirstNameRequest(user_id):
	request_body = """<?xml version="1.0" encoding="UTF-8" standalone="no" ?>
	<SOAP-ENV:Envelope
	SOAP-ENV:encodingStyle="http://schemas.xmlsoap.org/soap/encoding/"
	xmlns:SOAP-ENV="http://schemas.xmlsoap.org/soap/envelope/"
	xmlns:SOAP-ENC="http://schemas.xmlsoap.org/soap/encoding/"
	xmlns:xsd="http://www.w3.org/2001/XMLSchema"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
	<SOAP-ENV:Body>
		<m:getFirstName xmlns:m="urn:SoapRequest">
			<id xsi:type="xsd:int">{}</id>
		</m:getFirstName>
	</SOAP-ENV:Body>
	</SOAP-ENV:Envelope>
	""".format(user_id)

	return request_body

def getLastNameRequest(user_id):
	request_body = """<?xml version="1.0" encoding="UTF-8" standalone="no" ?>
	<SOAP-ENV:Envelope
	SOAP-ENV:encodingStyle="http://schemas.xmlsoap.org/soap/encoding/"
	xmlns:SOAP-ENV="http://schemas.xmlsoap.org/soap/envelope/"
	xmlns:SOAP-ENC="http://schemas.xmlsoap.org/soap/encoding/"
	xmlns:xsd="http://www.w3.org/2001/XMLSchema"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
	<SOAP-ENV:Body>
		<m:getLastName xmlns:m="urn:SoapRequest">
			<id xsi:type="xsd:int">{}</id>
		</m:getLastName>
	</SOAP-ENV:Body>
	</SOAP-ENV:Envelope>
	""".format(user_id)

	return request_body

def main():
	send_soap()

main()
{% endhighlight %}

And this returns:

<img src="{{ site.baseurl }}/images/webgoat/2017-02-01-webgoat_part_21/request-results.jpg">

Now I didn't deserialize the responses, because I was lazy, but we can see clearly that user 101 goes by the full name "Joe Snow". And when we refresh the challenge page:

<img src="{{ site.baseurl }}/images/webgoat/2017-02-01-webgoat_part_21/request-success.jpg">

Nice! We have successfully built a SOAP client and interacted with the challenge server interface! Another lengthy process of learning a new protocol and implementing an interacting client, but once again learned a lot about how the bits and bytes work.

### References

[1. SOAP Protocol][soap-user]

[2. Reading WSDL][reading-wsdl]

[3. W3 SOAP][w3-soap]

[soap-user]:http://www.soapuser.com/basics1.html
[reading-wsdl]:http://www.predic8.com/wsdl-reading.htm
[w3-soap]:Zhttp://www.w3schools.com/xml/xml_soap.asp