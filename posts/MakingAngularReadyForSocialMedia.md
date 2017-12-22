## Making a SPA usable for social media previews

I think we all have seen the previews created by twitter or facebook when you post a link

[Add Picture here]

I never about how those things were created. Yes ... sure there will be a request to the page, greping the title some images and some of the text as description ... can't be that hard.
But this request as i learned yesterday is only a plain html request without any javascript interpretation.
And here comes the problem for a SPA (Single Page Application) like those created with Angular. In Angular all requests that are not a valid file like style informations or pictures get redirected to the index.html and the then the main part of this page that is written in javascript can do it's magic and fill the page with the content we want. But a page that mainly relies on javascript and a request that ignores javascript gave me a serious problem.

On my index.html page the crawler from facebook could only find informations about the title of my index.html, definitly not enough information. So i googled a bit and found a nice suggestion

https://www.michaelbromley.co.uk/blog/enable-rich-social-sharing-in-your-angularjs-app/

With this page i found out that pages like facebook also can be feed with special meta tags
``` html
<head>
    <meta property="og:title" content="My Page" />
    <meta property="og:description" content="A description of my page." />
    <meta property="og:image" content="http://www.mysite.com/images/my_lovely_face.jpg" />
    <!-- etc. -->
</head>
```

And after a while i first found out
-	facebook does cache those crawl requests 
-	and they would like to get some more metadata than mentioned on that page

First i tried to debug those request with different URLs and multiple tools to try to see if my redirecting works. And then i found one of the most important informations (far too late). A list of links to debug this preview creation and to reset the cache ...
- https://developers.facebook.com/tools/debug/
- https://dev.twitter.com/docs/cards/validation/validator
- https://developers.pinterest.com/rich_pins/validator/
- http://www.google.com/webmasters/tools/richsnippets

With all this information let me show you how i solved this problem even if i use most solutions from this website.

### Solution
The main idea is to recognize the request made by the social media crawler and redirect it to a special webpage that generates all the crawler relevant information.

#### Creating the crawler page
First we have to write a webpage that gives the social media crawler the information it needs, i will use the example from the page with somne slight alterations. The page used in this example is written in php and my altered version looks like this
```php
<?php
$SITE_ROOT = "http://www.mysite.com/";

$jsonData = getData($SITE_ROOT);
makePage($jsonData, $SITE_ROOT);
function getData($siteRoot) {
    $id = ctype_digit($_GET['id']) ? $_GET['id'] : 1;
    $rawData = file_get_contents($siteRoot.'api/'.$id);
    return json_decode($rawData);
}

function makePage($data, $siteRoot) {
    ?>
    <!DOCTYPE html>
    <html>
        <head>
        	<title><?php echo $data->title; ?></title>
            <meta property="og:title" content="<?php echo $data->title; ?>" />
            <meta property="og:description" content="<?php echo $data->description; ?>" />
            <meta property="og:image" content="<?php echo $data->image; ?>" />
            <meta property="og:type" content="website" />
            <!-- etc. -->
        </head>
        <body>
            <p><?php echo $data->description; ?></p>
            <img src="<?php echo $data->image; ?>">
        </body>
    </html>
    <?php
}
?>
```
This page expects an `id` as query parameter, sends a request to an api and sets the gathered information to a crawler readable webpage.

Now you already got a page to test with those debuging links above. But since you don't want to link this php page all the time we have to add some redirecting

#### Redirecting crawlers to the crawler page

When i set up an Angular site i always add a `.htaccess` file to redirect everything to the index.html so Angular can do his routing magic. Normally a `.htaccess` file for this looks like this
```
RewriteEngine on

RewriteBase /

RewriteCond %{REQUEST_FILENAME} -f [OR]
RewriteCond %{REQUEST_FILENAME} -d
RewriteRule ^ - [L]

RewriteRule ^ /index.html [L]
```

So i redirect everything that is not a valid file or directory to my index.html (You need `mod_rewrite` for this)

But now we also need a handling for those social media crawlers. The crawlers can be identified by the `User-Agent` used.

Here is the list from the page that inspired me (i will have a look if all are still valid, by now i only needed the facebook and twitter and they worked)

- Facebook: `facebookexternalhit/1.1 (+http(s)://www.facebook.com/externalhit_uatext.php)`
- Twitter: `Twitterbot/{version}`
- Pinterest: `Pinterest/0.1 +http://pinterest.com/`
- Google Plus: `Google (+https://developers.google.com/+/web/snippet/)`
- Google Structured Data Testing tool: `Google-StructuredDataTestingTool; +http://www.google.com/webmasters/tools/richsnippets`

So as i said, first we have to identify those crawlers, because we only want those crawlers send to our php page

```
RewriteCond %{HTTP_USER_AGENT} (facebookexternalhit/[0-9]|Twitterbot|Pinterest|Google.*snippet)
```

And then we have to extract the data to identify the page from the URL called and send it to out php script.

Example from the website
```
RewriteRule album/(\d*)$ http://www.michaelbromley.co.uk/experiments/angular-social-demo/server/static-page.php?id=$1 [P]
```

but i don't like to redirect to an URI with http/https and domain, maybe i want to change that ... so my solution currently looks like this (i still have to adapt it so it can also handle the other possible URLs in the Angular site)
```
RewriteRule "^EinfachMalReingucken/Geschichten/([a-zA-Z0-9% ]+)$" "/assets/crawler.php?title=$1" [PT]
```
The obvious changes i did were
- change the target php script to a script i place in my assets
- change the input parameter to title
- change the source URL

The littel more subtle changes
- i changed the flag from [P] to [PT] so i can use a relative target
- i added "" around the source and the target so whitespaces don't mess up everything (took me some time to find this when i got the error `bad flag delimiters`)

So just to show you how the `.htaccess` file looks after adding those lines 
```
RewriteEngine on

RewriteBase /

RewriteCond %{REQUEST_FILENAME} -f [OR]
RewriteCond %{REQUEST_FILENAME} -d
RewriteRule ^ - [L]

# allow social media crawlers to work by redirecting them to a server-rendered static version on the page
RewriteCond %{HTTP_USER_AGENT} (facebookexternalhit/[0-9]|Twitterbot|Pinterest|Google.*snippet)
RewriteRule "^EinfachMalReingucken/Geschichten/([a-zA-Z0-9% ]+)$" "/assets/crawler.php?title=$1" [PT]

RewriteRule ^ /index.html [L]
```

But this is only a part of the solution to make a SPA social media and SEO. Problems till remaining are
- you have to adapt the php script to generate a page for every URL in your SPA
- you will still not be very search engine friendly ...

I haven't really checked the following site, but it sounds promising to solve this problem
https://coursetro.com/posts/code/68/Make-your-Angular-App-SEO-Friendly-(Angular-4-+-Universal)
https://medium.com/@feloy/angular-v4-universal-demystified-f06a576d6ca2
