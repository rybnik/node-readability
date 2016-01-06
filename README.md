read-art [![NPM version](https://badge.fury.io/js/read-art.svg)](http://badge.fury.io/js/read-art) [![Build Status](https://travis-ci.org/Tjatse/node-readability.svg?branch=master)](https://travis-ci.org/Tjatse/node-readability)
=========
[![NPM](https://nodei.co/npm/read-art.png?downloads=true&downloadRank=true&stars=true)](https://nodei.co/npm/read-art/)

1. Readability reference to Arc90's.
2. Scrape article from any page (automatically).
3. Make any web page readable, no matter Chinese or English.

> *快速抓取网页文章标题和内容，适合node.js爬虫使用，服务于ElasticSearch。*

## Guide
- [Features](#features)
- [Performance](#perfs)
- [Installation](#ins)
- [Usage](#usage)
- [Score Rule](#score_rule)
- [Extract Selectors](#selectors)
- [Customize Settings](#cus_sets)
- [Output](#output)
- [Notes](#notes)

<a name="features" />
## Features
- Fast And Shoot Straight.
- High Performance - Less memory
- Automatic Read Title & Content
- Follow Redirects
- Automatic Decoding Content Encodings(Avoid Messy Codes, Especially Chinese)
- Gzip/Deflate Support
- Proxy Support
- Auto-generate User-Agent
- Free and extensible

<a name="perfs" />
## Performance
In my case, the speed of [spider](https://github.com/Tjatse/spider2) is about **700 thousands documents per day**, **22 million per month**, and the maximize crawling speed is **450 per minute**, **avg 80 per minute**, the memory cost are about **200 megabytes** on each spider kernel, and the accuracy is about 90%, the rest 10% can be fixed by customizing [Score Rules](#score_rule) or [Selectors](selectors). it's better than any other readability modules.

![image](screenshots/performance.jpg)

> Server infos: 
> * 20M bandwidth of fibre-optical
> * 8 Intel(R) Xeon(R) CPU E5-2650 v2 @ 2.60GHz cpus
> * 32G memory

<a name="ins" />
## Installation
```javascript
npm install read-art --production
```

<a name="usage" />
## Usage
```javascript
read(html/uri [, options], callback)
```

It supports the definitions such as:
  * **html/uri** Html or Uri string.
  * **options** An optional options object, including:
    - **output** The data type of article content, head over to [Output](#output) to get more information.
    - **killBreaks** A value indicating whether kill breaks, blanks, tab symbols(\r\t\n) into one `<br />` or not, `true` by default.
    - **minTextLength** If the content is less than `[minTextLength]` characters, don't even count it, `25` by default.
    - **tidyAttrs** Remove all the attributes on elements, `false` by default.
    - **dom** Will return the whole cheerio dom when this property is set to `true`, `false` by default, try to use `art.dom` to get the dom object in callback function.
    - **damping** The damping to calculate score of parent node, `1/2` by default. e.g.: the score of current document node is `20`, the score of parent will be `20 * damping`.
    - **options from [cheerio](https://github.com/cheeriojs/cheerio)**
    - **options from [req-fast](https://github.com/Tjatse/req-fast)**
    - **scoreRule** Customize the score rules of each node, one arguments will be passed into the callback function (head over to [Score Rule](#score_rule) to get more information):
      - **node** The [cheerio object](https://github.com/cheeriojs/cheerio#selectors).
    - **selectors** Customize the data extract [selectors](#selectors).
    - **imgFallback** Customize the way to get source of image, should be one of following types.
      - *Boolean* Fallback to `img.src = (node.data('src') || node.attr('data-src'))` when set to `true`.
      - *String* Customize the attribute name, it will take `node.attr([imgFallback])` as `src` of `img`.
      - *Function* Give users maximum customizability and scalability of source attribute on `img`, e.g.:

        ```javascript
        imgFallback: function(node){
          return node.attr('base') + '/' + node.attr('rel-path');
        }
        ```
  * **callback** The callback to run - `callback(error, article, options, response)`, arguments are:
    - **error** `Error` object when exception has been caught.
    - **article** The article object, including: `article.title`, `article.content` and `article.html`.
    - **options** The request options.
    - **response** The response of your request, including: `response.headers`, `response.redirects`, `response.cookies` and `response.statusCode`.

> Head over to test or examples directory for a complete example.

<a name="usage_eg" />
### Examples
```javascript
var read = require('read-art');
// read from google:
read('http://google.com', function(err, art, options, resp){
    if(err){
      throw err;
    }
    var title = art.title,      // title of article
        content = art.content,  // content of article
        html = art.html;        // whole original innerHTML

    console.log('[STATUS CODE]', resp && resp.statusCode);
});
// or:
read({
    uri: 'http://google.com',
    charset: 'utf8'
  }, function(err, art, options, resp){

});
// what about html?
read('<title>node-art</title><body><div><p>hello, read-art!</p></div></body>', function(err, art, options, resp){

});
// of course could be
read({
    uri: '<title>node-art</title><body><div><p>hello, read-art!</p></div></body>'
  }, function(err, art, options, resp){

});
```
**CAUTION:** Title must be wrapped in a `<title>` tag and content must be wrapped in a `<body>` tag.

**With High Availability: [spider2](https://github.com/Tjatse/spider2)**

<a name="score_rule" />
## Score Rule
In some situations, we need to customize score rules to crawl the correct content of article, such as BBS and QA forums.
There are two effective ways to do this:
- **minTextLength**
  It's useful to get rid of useless elements (`P` / `DIV`), e.g. `minTextLength: 100` will dump all the blocks that `node.text().length` is less than `100`.

- **scoreRule**
  You can customize the score rules manually, e.g.:
  ```javascript:
  scoreRule: function(node){
    if (node.hasClass('w740')) {
      return 100;
    }
  }
  ```

  The elements which have the `w740` className will get `100` bonus points, that will make the `node` to be the *topCandidate*, which means it's enough to make the `text` of `DIV/P.w740` to be the content of current article.

<a name="score_rule_eg" />
### Example
```javascript
read('http://club.autohome.com.cn/bbs/thread-c-66-37239726-1.html', {
  minTextLength: 0,
  scoreRule: function(node){
    if (node.hasClass('w740')) {
      return 100;
    }
  }
}, function(err, art){

});
```

<a name="selectors" />
## Extract Selectors
Some times we wanna extract article somehow, e.g.: pick the text of `.article>h3` as title, and pick `.article>.author` as the author data:

### Example
```javascript
read({
  html: '<title>read-art</title><body><div class="article"><h3 title="--read-art--">Who Am I</h3><p class="section1">hi, dude, i am <b>readability</b></p><p class="section2">aka read-art...</p><small class="author" data-author="Tjatse X">Tjatse</small></div></body>',
  selectors: {
    title: {
      selector: '.article>h3',
      extract: ['text', 'title']
    },
    content: '.article p.section1',
    author: {
      selector: '.article>small.author',
      extract: {
        shot_name: 'text',
        full_name: 'data-author'
      }
    }
  },
}, function (err, art) {
  // art.title === {text: 'Who Am I', title: '--read-art--'}
  // art.content === 'hi, dude, i am <b>readability</b>'
  // art.author === {shot_name: 'Tjatse', full_name: 'Tjatse X'}
});
```

Properties:
- **selector** the query selector, e.g.: `#article>.title`, `.articles:nth-child(3)`
- **extract** the data that you wanna extract, could be `String`, `Array` or `Object`.

**Notes** The binding data will be an object or array (object per item) if the `extract` option is an array object, `title` and `content` will override the default extracting methods, and the output of `content` depends on the `output` option.  

<a name="cus_sets" />
## Customize Settings
We're using different regexps to iterates over elements (cheerio objects), and removing undesirable nodes.
```javascript
read.use(function(){
  //[usage]
});

```

The `[usage]` could be one of following:
- `this.reset()`
  Reset the settings to default.
- `this.skipTags([tags], [override])`
  Remove useless elements by tagName, e.g. `this.skipTags('b,span')`, if `[override]` is set to `true`, `skiptags` will be `"b,span"`, otherwise it will be appended to the origin, i.e. :
  ```
  aside,footer,label,nav,noscript,script,link,meta,style,select,textarea,iframe,b,span
  ```

- `this.regexps.positive([re], [override])`
  If `positive` regexp test `id` + `className` of node success, it will be took as a candidate. `[re]` is a regexp, e.g. `/dv101|dv102/` will match the element likes `<div class="dv101">...` or `<div id="dv102">...`, if `[override]` is set to `true`, `positive` will be `/dv101|dv102/i`, otherwise it will be appended to the origin, i.e. :
  ```
  /article|blog|body|content|entry|main|news|pag(?:e|ination)|post|story|text|dv101|dv102/i
  ```

- `this.regexps.negative([re], [override])`
  If `negative` regexp test `id` + `className` of node success, it will not be took as a candidate. `[re]` is a regexp, e.g. `/dv101|dv102/` will match the element likes `<div class="dv101">...` or `<div id="dv102">...`, if `[override]` is set to `true`, `negative` will be `/dv101|dv102/i`, otherwise it will be appended to the origin, i.e. :
  ```
  /com(?:bx|ment|-)|contact|comment|captcha|foot(?:er|note)?|link|masthead|media|meta|outbrain|promo|related|scroll|shoutbox|sidebar|sponsor|util|shopping|tags|tool|widget|tip|dialog|copyright|bottom|dv101|dv102/i
  ```

- `this.regexps.unlikely([re], [override])`
  If `unlikely` regexp test `id` + `className` of node success, it probably will not be took as a candidate. `[re]` is a regexp, e.g. `/dv101|dv102/` will match the element likes `<div class="dv101">...` or `<div id="dv102">...`, if `[override]` is set to `true`, `unlikely` will be `/dv101|dv102/i`, otherwise it will be appended to the origin, i.e. :
  ```
  /agegate|auth?or|bookmark|cat|com(?:bx|ment|munity)|date|disqus|extra|foot|header|ignore|link|menu|nav|pag(?:er|ination)|popup|related|remark|rss|share|shoutbox|sidebar|similar|social|sponsor|teaserlist|time|tweet|twitter|\bad[\s_-]?\b|dv101|dv102/i
  ```

- `this.regexps.maybe([re], [override])`
  If `maybe` regexp test `id` + `className` of node success, it probably will be took as a candidate. `[re]` is a regexp, e.g. `/dv101|dv102/` will match the element likes `<div class="dv101">...` or `<div id="dv102">...`, if `[override]` is set to `true`, `maybe` will be `/dv101|dv102/i`, otherwise it will be appended to the origin, i.e. :
  ```
  /and|article|body|column|main|column|dv101|dv102/i
  ```

- `this.regexps.div2p([re], [override])`
  If `div2p` regexp test `id` + `className` of node success, all divs that don't have children block level elements will be turned into p's. `[re]` is a regexp, e.g. `/<(span|label)/` will match the element likes `<span>...` or `<label>...`, if `[override]` is set to `true`, `div2p` will be `/<(span|label)/i`, otherwise it will be appended to the origin, i.e. :
  ```
  /<(a|blockquote|dl|div|img|ol|p|pre|table|ul|span|label)/i
  ```

<a name="cus_sets_eg" />
### Example
```javascript
read.use(function(){
  this.reset();
  this.skipTags('b,span');
  this.regexps.div2p(/<(span|b)/, true);
});
```

<a name="output" />
## Output
You can wrap the content of article with different types, it supports `text`, `html` `json` and `cheerio`, the `output` option could be:
- **String**
  One of types, `html` by default.
- **Object**
  Key-value pairs including:
  - **type**
    One of types.
  - **stripSpaces**
    A value indicates whether strip the tab symbols (\r\n\t) or not, `false` by default.
  - **break**
    A value indicates whether split content into paragraphs by `<br />` (Only affects JSON output).

<a name="output_text" />
### text
Returns the inner text, e.g.:
```javascript
read('http://example.com', {
  output: 'text'
}, function(err, art){
  // art.content will be formatted as TEXT
});
// or
read('http://example.com', {
  output: {
    type: 'text',
    stripSpaces: true
  }
}, function(err, art){
  // art.content will be formatted as TEXT
});
```

<a name="output_html" />
### html
Returns the inner HTML, e.g.:
```javascript
read('http://example.com', {
  output: 'html'
}, function(err, art){
  // art.content will be formatted as HTML
});
// or
read('http://example.com', {
  output: {
    type: 'html',
    stripSpaces: true
  }
}, function(err, art){
  // art.content will be formatted as HTML
});
```

**Notes** Videos could be scraped now, the domains currently are supported: *youtube|vimeo|youku|tudou|56|letv|iqiyi|sohu|sina|163*.

<a name="output_json" />
### json
Returns the restful result, e.g.:
```javascript
read('http://example.com', {
  output: 'json'
}, function(err, art){
  // art.content will be formatted as JSON
});
// or
read('http://example.com', {
  output: {
    type: 'json',
    stripSpaces: true,
    break: true
  }
}, function(err, art){
  // art.content will be formatted as Array
});
```

The art.content will be an Array such as:
```json
[
  { "type": "img", "value": "http://example.com/jpg/site1/20140519/00188b1996f214e3a25417.jpg" },
  { "type": "text", "value": "TEXT goes here..." }
]
```

Util now there are only two types - *img* and *text*, the `src` of `img` element is absolute even if the original is a relative one.

<a name="output_cheerio" />
### cheerio
Returns the cheerio node, e.g.:
```javascript
read('http://example.com', {
  output: 'cheerio'
}, function(err, art){
  // art.content will be a cheerio node
  art.content.find('div.what>ul.you>li.need');
});
// or
read('http://example.com', {
  output: {
    type: 'cheerio',
    stripSpaces: true
  }
}, function(err, art){
  // art.content will be a cheerio node
  art.content.find('div.what>ul.you>li.need');
});
```

**Notes** The video sources of the sites are quite different, it's hard to fit all in a common way, I haven't find a good way to solve that, PRs are in demand.

<a name="notes" />
## Notes / Gotchas
**Pass the charset manually to refrain from the crazy messy codes**
```javascript
read('http://game.163.com/14/0506/10/9RI8M9AO00314SDA.html', {
  charset: 'gbk'
}, function(err, art){
  // ...
});
```

**Generate agent to simulate browsers**
```javascript
read('http://example.com', {
  agent: true // true as default
}, function(err, art){
  // ...
});
```

**Use proxy to avoid being blocked**
```javascript
read('http://example.com', {
  proxy: {
    host: 'http://myproxy.com/',
    port: 8081,
    proxyAuth: 'user:password'
  }
}, function(err, art){
  // ...
});
```

## Test
```
npm test
```

## License
Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.


