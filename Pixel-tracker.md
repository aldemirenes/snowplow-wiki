[**HOME**](Home) » [**SNOWPLOW TECHNICAL DOCUMENTATION**](Snowplow-technical-documentation) » [**Trackers**](trackers) » Pixel tracker

This guide covers:

1. [What is the Pixel tracker?](#what)
2. [Anatomy of a Pixel tracking tag](#anatomy)
3. [Using the Pixel tracker with the Clojure collector](#clojure)
4. [Click tracking](#click-tracking)

<a name="what" />

## 1. What is the Pixel tracker?

The Pixel tracker is an HTML-only tracking tag (no JavaScript) to track opens / views of HTML content that does not support JavaScript. Examples of use cases include HTML emails.

In a normal JavaScript tag, the name-value pairs of data that are sent through to the Snowplow collector via the querystring are calculated on the fly by the JavaScript. (Examples of data points that are calculated on the fly include `user_id`, or `browser_features`.)

In an environment where JavaScript is not permitted, these values need to be set in advance and hardcoded into the tracking tag. As a result, if you want to record a different `page_title`, for example, for several different HTML-only web pages using the tracking code, you will need to generate a different tracking tag for each of those different web pages, with the right `page_title` set for each.

<a name="anatomy" />

## 2. Anatomy of a Pixel tracking tag

An example tag is shown below:

```html
<!--Snowplow start plowing-->
<img src="http://collector.acme.com/i?&e=pv&page=Root%20README&url=http%3A%2F%2Fgithub.com%2Fsnowplow%2Fsnowplow&aid=snowplow&p=web&tv=no-js-0.1.0" />
<!--Snowplow stop plowing-->
```

Some things to note about the tag:

1. It is a straightforward `<img ...>` tag, that results in the GET request to the Snowplow tracking pixel
2. The endpoint is set to a Clojure collector that we are running at `collector.snplow.com`.
3. Five data points are passed on the query string: the event type (`pageview`), the page name (`Root README`), the URL (`http://github.com/snowplow/snowplow`), the application id (`snowplow`), the platform (`web`) and the tracker version (`no-js-0.1.0`)

<a name="clojure" />

## 3. Using the Pixel tracker with the Clojure collector

When using the Pixel tracker with the Clojure collector, the Clojure collector sets a `user_id` and drops this on a browser cookie.

Care must therefore be exercised when using the Pixel tracker on domains that you do not own. **It is your responsibility to abide by the terms and conditions of any domain owner for domains where you post content including uploading Pixel tracking tags.** Some domain owners forbid 3rd parties from dropping cookies on their domains. It is your responsibility to ensure you do not violate the terms and conditions of any domain owners that you work with.

<a name="click-tracking" />

## 4. Click tracking

**This feature requires Snowplow R72+ and the Clojure Collector 1.1.0+**

You can use the Pixel tracker for click tracking aka URI redirects:

* Set your collector path to `{{collector-domain}}/r/tp2?{{name-value-pairs}}` - the `/r/tp2` tells Snowplow that you are attempting a URI redirect
* Add a `&u={{uri}}` argument to your collector URI, where `{{uri}}` is the URL-encoded URI that you want to redirect to
* On clicking this link, the collector will register the link and then do a 302 redirect to the supplied `{{uri}}`
* As well as the `&u={{uri}}` parameter, you can populate the collector URI with any other fields from the [[Snowplow Tracker Protocol]]

Example:

```html
Check out <a href="http://collector.acme.com/r/tp2?u=https%3A%2F%2Fgithub.com%2Fsnowplow%2Fhuskimo">Huskimo</a>
```

Snowplow converts the `&u={{uri}}` argument into a `com.snowplowanalytics.snowplow/uri_direct` self-describing JSON.

How Snowplow attaches the `uri_redirect` to the event depends on what other Tracker Protocol fields you attached to the event:

1. If you attached an `&e={{event type}}` to your event, then the `uri_redirect` will be added to the contexts array of your event
2. If you did not attach an `&e={{event type}}` to your event, then this event will be treated as an unstructured event and the `uri_redirect` will be attached as the event itself

[pixel-repo]: https://github.com/snowplow/snowplow/tree/master/1-trackers/no-js-tracker
