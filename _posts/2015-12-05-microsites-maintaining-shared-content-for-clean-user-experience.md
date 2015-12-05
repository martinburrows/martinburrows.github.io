---
layout: post
title:  "Microsites: Maintaining shared content for a clean user experience"
date:   2015-12-05 21:40:00
categories: architecture microsites
image:  /images/posts/microsites-maintaining-shared-content-for-clean-user-experience-0.png
description: "Managing updates to common headers, footers and sidebars can be a challenge when working across microsite deployments. How can a change be made without re-releasing every microsite with it?"
---

Microsites. We all know how good they are for coping with the business needs of several projects all operating under one brand, for developers pushing changes without conflicts and for making small, low-risk, frequent releases. Changes involving one product can made without affecting another, meanwhile the average site visitor is blissfully unaware as they flow seamlessly across your site. 

That is - until we need to update anything common across all our sites and find our deployments go dreadfully out of sync. 

[![Illustration](/images/posts/microsites-maintaining-shared-content-for-clean-user-experience-0.png)][image0]

Our seamless experience transforms into a multi-branded mess, at best looking unprofessional and at worst confusing the user and driving them away. Whilst this is an extreme example, even small changes such as a simple fix to a broken link might be pushed to some sites and missed in others, not to mention the time taken to go through this process with every change and dealing with any sites which have work in progress and are not production-ready.

Managing updates to common headers, footers and sidebars can be a challenge when working across microsite deployments. An update to your site header in a monolithic site deployment typically requires one change to a layout file - updates made here apply to all other pages. The popularity of microsites introduces a new challenge - how can a similar change be made to a common layout without re-releasing every microsite too? 

This post summarises the solution I introduced at [JustGiving][justgiving] when faced with this challenge during its first microsite rollout last year.

<!--more-->

Flexible layouts
---

The site had a variety of page layout requirements - some pages contained standard headers and footers, some had infinite scrolling with no footer, and some used custom headers which still needed to be shared with other microsites. We could have used some kind of shared layout portal or wrapper through which other content could render inside, but the variety of different layout requirements would have complicated this. We wanted flexibility for sites in a layout they could prescribe rather than being contained in a particular way.   

A JavaScript library making an ajax call to pull in shared content was also ruled out. We wanted to avoid page content "jumping down" as the header got inserted above the page after its initial rendering, and we didn't want to wait for several extra HTTP requests to be made before showing anything. Looking to a server-side solution we also wanted to maintain every page's performance and minimise any extra calls needed to pull in the shared layout. Any server-side solution would need to be platform-agnostic to allow any type of web server to use it whether running ASP.NET, Node or anything else.


Server-side rendering
---

The eventual solution involved using server-side includes to pull back common assets from a shared service with several layers of caching to optimise site performance. A simplified form of this is represented here: 

[![Illustration](/images/posts/microsites-maintaining-shared-content-for-clean-user-experience-1.png)][image1]

* A user's browser requests a web page. 
* Our web server receives the request...  
  * The page is generated as normal using whichever layout is required for that microsite. 
  * Placeholders are used whenever shared content is required (one for the header, another for the footer, etc).
  * The web server replaces everything marked by these placeholders with content from our internal common content server.
* The complete page response can then be sent back to the user's browser with microsite and shared content all together.
* Any images, styles and scripts referenced in the header, footer or sidebar can be requested directly from a CDN.

This solution means that whenever a change to our header is released to the common content service (CS) the update appears across our estate immediately without intervention on any microsites since they always pull the latest header when generating each page. Having the most recent header version means we also reference the most recent css, images and scripts from our CDN which we can update whenever our CS gets updated.

### Diving into detail 
     
So how exactly is the content sent from our CS to each web server, and how can it be shown with any layout the microsite chooses?

The layout file of a site might look like this example which uses server-side includes in a Razor template:
{% highlight aspx-cs  %}
<html>
  <head>
    <!-- other head content skipped -->
    @CommonHelper.Styles()
  </head>
  <body>
    @CommonHelper.Header()
    
    <!-- page content here -->
    
    @CommonHelper.Footer()
    
    <!-- page scripts here -->
    @CommonHelper.Scripts()
  </body>
</html>
{% endhighlight %}
 
This simplified example shows we've split our common content into four parts. 

* Each common part is independent, since they all belong on different areas of the rendered page.
* Some parts can also be optional - we might only want the header with no footer.

We've written the helper methods shown here to each fetch a chunk of HTML from our CS to display in their place. Each part can be accessed via its own endpoint from the CS. For example, the request for the styles and header sections might result in these requests and responses from our web server to our CS:


GET `http://content.internal.mydomain.com/styles`
`<link rel="stylesheet" type="text/css" href="//cdn.mydomain.com/header.css"></link>`


GET `http://content.internal.mydomain.com/header`
`<header>`
  `<h1>My amazing header</h1>`
`</header>`


The final generated web page sent back to the web browser would therefore look something like this:
 
{% highlight html %}
<html>
  <head>
    <!-- other head content skipped -->
    <link rel="stylesheet" type="text/css" href="//cdn.mydomain.com/header.css"></link>
  </head>
  <body>
    <header>
      <h1>My amazing header</h1>
    </header>
    
    <!-- remaining content here -->
  </body>
</html>
{% endhighlight %}
 
So common content can be placed anywhere on each microsite whilst always showing the latest header and footer with no further release necessary. 

The server-side includes used can be bundled into a package manager (eg. NuGet) and installed on every microsite to avoid the need to write them multiple times.
 
As I mentioned earlier we have several different header variations in use across our sites which still needed to be shared. We solved this by allowing microsites to request header/footer types from the CS if we wanted anything other than the default. All variations are still maintained by the CS and keep the other benefits of using this system.  


Optimising for speed
---

Obviously there are drawbacks to making several further requests from each microsite web server to our CS every time a page needs to be rendered and sent back to a waiting client. For this reason we utilised many layers of caching.

[![Illustration](/images/posts/microsites-maintaining-shared-content-for-clean-user-experience-2.png)][image2]

As mentioned already, server-side includes are used to pull down the content from the CS. These are packaged up and used on each web server. Also contained in the logic which downloads content from the CS is an in-memory caching implementation. Subsequent retrievals of the same content can be found in the cache instead of making new requests to the CS, so there is no impact on response time for anyone except the first visitor during the lifetime of the cache. An additional caching layer is used between the microsites and the CS to keep all response times at a minimum. 

These two caching layers result in very low traffic to the machines hosting the CS even though practically all web requests are using content provided by them. As all traffic shown above is within our network each response takes a few milliseconds, so even when content is not cached the impact for the user is negligible.  

The maximum age of caches such as this always depends on business needs, but in our case we tend to cache for around 30 seconds. This strikes the balance between the freshness of the shared content and the speed at which your site needs to respond to each user.   

One scenario which this caching configuration does not allow for is server-side generated user-specific content. For example, if the header is personalised to show a user's name and picture this cannot be cached and re-used for another visitor for obvious reasons. Our solution to this is to personalise our headers on the client side. We can do this since we're pulling scripts from the CS together with the header, so we can add this once we get to the browser.

Conclusion
---

We've been using this content sharing method successfully for well over a year now across several microsites. If the model fits with your site I can only recommend it.

As with any solution to microsite content sharing, one thing to mention is the importance of keeping the shared content lite and self-contained. You don't want your styles affecting something in a microsite and vice versa. A small number of css resets/namespacing is the key here, as is ensuring your scripts don't leak anything onto the global scope. If at all possible avoid using any JavaScript libraries to keep conflicts with different versions of the same library and file sizes to a minimum.

Obviously there is far more detail not mentioned here... as ever questions, criticisms and ideas are welcome in the comments below!



[justgiving]:      http://justgiving.com
[follow]:      http://martinburrows.net/feed.xml

[image0]:	/images/posts/microsites-maintaining-shared-content-for-clean-user-experience-0.png
[image1]:	/images/posts/microsites-maintaining-shared-content-for-clean-user-experience-1.png
[image2]:	/images/posts/microsites-maintaining-shared-content-for-clean-user-experience-2.png