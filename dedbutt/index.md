# Dead Button Syndrome

There is a frustrating design issue I've been seeing more and more often that I think should have a name.  Navigate to the [issues page for this repository](https://github.com/em-tg/em-tg.github.io/issues).  If you already have a slow connection, you might be able to reproduce this easily, but if not you can simply enable network throttling using the developer tools available in most web browsers.

Near the top of the page is a search box for filtering visible issues.  As soon as the box appears, try typing in it or clicking the magnifying glass to trigger a search.  If you're fast enough, the search button will do absolutely nothing.  Even worse, text you type into the box will be invisible and is deleted after a seemingly random interval.

I don't know what the name for this is, so I'm calling it "dead button syndrome," because when it happens to buttons, the control is visible on screen and interactive, but does nothing.  I believe the blame for this falls largely on React and Next.js, but I've seen it happen even in native programs, so it seems the trend is not limited to the web.

That's all I really wanted to say, but if you're interested, here's a brief history of what I think happened:

## The REST Era

In the beginning, web applications worked using a paradigm called "REST."  The browser requested a URL, and the server replied with a document that described to both the user (via text) and the browser (via links and forms) how they should go about making further requests.  A search engine's website might reply to a homepage request with something like this:

```html
<h1>InteRESTing Websites</h1>
<form action="results">
	<input type="text" name="q" placeholder="enter query">
	<button type="submit">Search</button>
</form>
<p>Have a nice day!
```

...which to the user appears as a text field and button that they can use to perform a search.  The markup tells the browser that when the user clicks the button, it should navigate to a URL built from the string `results?q=` and the text that the user entered.

Web pages can be slow to load, so the browser will display this form on the screen as soon as it becomes available.  Importantly, the button and text input will both be interactive as soon as they appear on screen, even if the "Have a nice day" text hasn't downloaded yet.

Javascript changed this:

```html
<h1>InteRESTing Websites</h1>
<input type="text" name="q" id="thetextbox" placeholder="enter query">
<button type="button" id="thebutton">Search</button>
<p>We're excited about Web 2.0!
<div id="output"></div>
<script src="./jquery.js"></script>
<script>
	$('#thebutton').on('click', function(){
		var params = { q: $('#thetextbox').val() };
		var results = $.get('results', params, function(data){
			$('#output').html(data);
		});
	});
</script>
```

Here instead of letting the browser make the request, a snippet of Javascript detects the submit button press, makes a request in the background, and then pastes the results into the page.  When the browser loads this page, it will still show the text input and button, even though neither are functional until the Javascript files are later downloaded.  The search button is a dead button!

The common advice of the time was to design a web page that worked correctly with Javascript disabled, then only use Javascript to make non-critical improvements to the user experience, a practice called [progressive enhancement](https://en.wikipedia.org/wiki/Progressive_enhancement):

```html
<h1>InteRESTing Websites</h1>
<form action="results" id="theform">
	<input type="text" name="q" id="thetextbox" placeholder="enter query">
	<button type="submit">Search</button>
</form>
<p>Note: fixed the "dead button" issue users were reporting
<div id="output"></div>
<script src="./jquery.js"></script>
<script>
	$('#theform').on('submit', function(e){
		e.preventDefault();
		var params = { q: $('#thetextbox').val(), ajax: true };
		var results = $.get('results', params, function(data){
			$('#output').html(data);
		});
	});
</script>
```

This version works like the first example when Javascript isn't available, and like the second example when it is.  If the user is fast enough to submit a query by the time the Javascript loads then the website will still behave as expected.

## The SPA Era

Web developers didn't like limiting themselves to the browser's built-in functionality, and so staged a revolt.  Instead of following "REST" and twisting their designs to fit the model of forms and links, programmers invented a new architecture: the SPA.

```html
<div id="content"></div>
<script src="basically-everything.js"></script>
```

There's no more HTML.  Instead, the UI is constructed dynamically at run-time by Javascript code.  In fact, Javascript controls everything: UI, network requests, styles... even scrolling!  With the added control comes the opportunity for improved user experiences: clicking a button doesn't need to re-request the whole page, and instead can request a tiny snippet of data (JSON-encoded, of course) and update the part of the page that needs changing.  In exchange, users need only tolerate a longer initial load time, and the accompanying loading animation.

The SPA paradigm is still the dominant one today, and has been around for so long that an entire generation of programmers has learned it as the only way to do web development.  The term "REST" [was re-defined to refer to those tiny snippets of JSON](https://htmx.org/essays/how-did-rest-come-to-mean-the-opposite-of-rest/), so that there is no longer even a name for the old way of doing things.  Websites as simple as blogs and forums now entirely fail to load when Javascript is disabled.

## React and SSR

Out of many contenders, React emerged as the de facto default tool for building SPAs, possibly because it was unopinionated, supported gradual adoption within existing projects, and was developed by Facebook.

In 2016, developers at Vercel (then Zeit) reviewed the state of affairs and decided to do something about the initial load time that plagued React-based SPAs.  [They released Next.js](https://vercel.com/blog/next), a web application development framework which allows applications to pre-render react components to HTML, so that they could be sent to the browser and displayed before the Javascript bundle finishes downloading.

Some time near the end of 2020, the React team [silently abandoned](https://github.com/react/create-react-app/graphs/code-frequency) their in-house SPA framework in favor of [improving React's built-in support for Next.js](https://news.ycombinator.com/item?id=25499171) via a feature called "React Server Components."  Eventually, frameworks with support for Server-Side Rendering (SSR) became the default mode of React development, [as recommended by React's own documentation](https://react.dev/learn/creating-a-react-app).

In a way, the SSR trend in the React world is a return to the old paradigm: users can once again see the content of the page without waiting for a bunch of Javascript to download.  Unfortunately, this has brought back dead buttons: none of the widgets sent by the server are interactive until the Javascript that powers them is ready.  In fact, the situation is much worse than in the dead button example I presented earlier, since the size of modern Javascript bundles is much larger than they were in the past and institutional knowledge about progressive enhancement techniques has been lost.

## Dead Buttons Make Everything Feel Slower

By the standard objective measures, these SSR web pages are faster than their traditional SPA counterparts, but some of those measures are flawed.  Metrics like [time to first contentful paint](https://developer.mozilla.org/en-US/docs/Glossary/First_contentful_paint) only measure the time until a page *looks like* it has finished loading, not the time until it *actually* finishes loading.

In effect, a website that throws up non-functioning HTML while the Javascript downloads is lying, both to benchmarking tools and to the end user, who expects the stuff they see to be clickable.  Eventually, enough time dealing with dead buttons causes users to internalize that they must wait for several seconds after an application has apparently finished loading before trying to interact with it.  This lesson effectively makes the *entire* computing experience feel slower, since a user can't tell in advance which applications require waiting and which don't.
## Solutions

I don't have any solutions to this problem other than to raise awareness that it exists and bothers me.  The techniques required to avoid it are all very old and well-documented; programmers seem to have simply stopped using them.







