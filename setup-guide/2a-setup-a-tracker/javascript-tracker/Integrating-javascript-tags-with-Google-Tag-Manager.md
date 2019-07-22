<a name="top" />

[**HOME**](Home) » [**SNOWPLOW SETUP GUIDE**](Setting-up-Snowplow) » [**Step 2: setup a Tracker**](Setting-up-a-Tracker) » [**JavaScript Tracker**](Javascript-tracker-setup) » Setting up the JavaScript Tracker with Google Tag Manager

This setup guide is divided into three sections:

1. [Setting up Google Tag Manager](#setup-gtm) (GTM)
2. [Integrating Snowplow JavaScript tracking tags with Google Tag Manager](#snowplow-setup)
3. [Integrating Snowplow JavaScript tracking tags with enhanced ecommerce](#enhanced-ecommerce)
4. [Next steps](#next-steps)

If you have already setup Google Tag Manager on your website, you can proceed directly to [section 2](#snowplow-setup). However, we recommend at least skimming the section on [setting up Google Tag Manager](#setup-gtm), as we've seen a number of companies implement this badly, with the end result that they cannot pass all the data they would like to Snowplow tags for analysis later.

<a name="setup-gtm" />

## 1. Setting up Google Tag Manager (GTM)

### Overview

There are six steps to setting up GTM to the point you can integrate Snowplow (or any other web analytics tool) with GTM:

1. [Create a Google Tag Manager account](#1.1)
2. [Integrate the container tag on your website](#1.2)
3. [Work out what data to pass to Google Tag Manager from your website, using the `dataLayer`](#1.3)
4. [Work out how to structure the data passed into `dataLayer`](#1.4)
5. [Integrate the `dataLayer` onto your website](#1.5)
6. [Create variables stored in the `dataLayer` in in the GTM UI](#1.6)

Typically, the steps people get wrong (or miss out alltogether) are steps 3-6. Getting them right is critical, however, because if you do not setup a robust mechanism for passing **all** the relevant data you want to report on in Snowplow (or any other web analytics program) into GTM, then GTM will not be able to pass that data into Snowplow, in turn. Either you will need to go back to your webmaster to add additional lines to your HTML / JavaScript to pass the missing data points later, or you'll be forced to perform analysis without them. Managing the data pipeline between your website(s) and GTM is key.

<a name="1.1" />

### 1.1 Create a Google Tag Manager account

Creating a Google Tag Manager account is very simple. Log on to [[http://www.google.com/tagmanager/]] and select the **SIGN IN** button. Once you have signed up, elect to create a **New Account**. Give the account a name. Then within it, create a **Container** and select your platform.

[[/setup-guide/images/gtm/1-create-account.png]]

In general, you'll only need to setup a single account, and most likely a single container for each domain you manage. However, that is not necessarily the case:

1. A container is a single GTM tag that you will install on your web pages instead of the individual tags from different web analytics and other vendors. If you use a common tag setup across many domains, then you can use the same container across multiple domains, rather than create a container for each domain. This might save administrative time in GTM, because you can then add and manage tags across all the domains in one go. (It is straightforward to add tags for a single domain within a multi-domain container setup, and to distinguish data collected from each domain, so there really is no disadvantage in using a single container across multiple domains.)
2. An account is a grouping of containers for administrative purposes. In general, one account is all you would need. If you work with different teams to manage different containers, then it makes sense to group the containers used by each team into a single account, with each team member being granted access to the account.

<a name="1.2" />

### 1.2 Integrating the container tag on your website

Once you have created a container, Google will provide you with the actual container tag. You will need to integrate this on every web page on your website(s), in place of any tags that are currently directly integrated on those web pages. (These will need to be migrated into GTM.)

[[/setup-guide/images/gtm/2-grab-embed-code.png]]

The embed code needs to be inserted on your web pages immediately after the opening `<body>` tag.

<a name="1.3" />

### 1.3 Work out what data to pass to Google Tag Manager from your website, using the `dataLayer`

The [`dataLayer`][datalayer] is a JSON that contains name value pairs of data points you wish to pass from your website into GTM. (And GTM can then, in turn, pass on to any tags that are managed in GTM, including Snowplow tags.)

Working out what data should be passed into the `dataLayer` is critical to ensuring that your GTM installation lasts. Getting it right, however, is not trivial. On the one hand, you need to be as comprehensive as possible: you need to identify every data point you might want to interrogate in your web analytics and make sure it is passed into the `dataLayer`. Otherwise it will not be possible to pass it on to Snowplow to use in analytics later.

On the other hand, you don't want to swamp GTM with lots of data that is never used: either because it is not passed onto any of the tags GTM manages, or because those points are never used as part of Snowplow analyses.

We recommend erring on the side of comprehensiveness: the cost of passing data into the `dataLayer` is small and Snowplow is built to house as much data as you can throw at it. As a process, we recommend:

1. Start off with the different objects that make up your product or service *universe*. If you're a video site, like Youtube, than videos will be the main object that make up your universe. But users, comments, likes and channels are all other objects that make up the Youtube experience. (Each one of these will be represented in the CMS behind Youtube.)
2. For each different object, think about what the key data points are that are interesting. For example, a video on Youtube will have an `id`, a `name`, an `author` / `producer` etc.
3. Now consider what actions / events happen on a user journey. Each will typically involve the user interacting with one or more objects e.g. a video on the site. If the user likes a video, comments on a video or uploads a video, it will involve creating a new object.
4. In general, each of the objects identified above should be pushed to the `dataLayer`, with each of the associated data points. It may not be necessary to push **all** the associated data points for each object, because it may be possible to infer some from another. (For example, if we know the `id` of a video, we might be able to lookup the `author` from the CMS at a later stage.) But we want to pass enough data points in with each object to support any analysis, that we might want to perform, down the line, on that object.
5. The list of objects, data points, and events on the user journeys, should be documented. Those documents will be used in the next step, working out how to [structure the data in the `dataLayer`](#1.4)

<a name="1.4" />

### 1.4 Work out how to structure the data passed into `dataLayer`

Broadly speaking, there are two occasions we can pass data points into the `dataLayer`:

1. When the page loads
2. When an event occurs. (Which might not correspond to a page load because it is AJAX.) This might include *watching a video*, *adding an item to the shopping basket*, *submitting a purchase*, *logging in*.

In general, if the data point is directly related to the webpage being loaded, it makes sense to push that data point into the data layer at page load time. For example, if on the product pages on an online shop, it would make sense to push key product details (`name`, `sku`, `price`, `category`) with the page load, by inserting the following code in the page **before** the container tags:

```html
<body>
	<script type="text/javascript">
		dataLayer = [{
      		'productSku': 'pbz00123',
      		'productName': 'The Original Rider Waite Tarot cards',
      		'productPrice': '9.99'
    	}];
	</script>
	 <!-- Google Tag Manager -->
  	...
  	<!-- End Google Tag Manager -->
</body>
```

GTM might then pass these data points into Snowplow as page level custom variables, for example.

We might equally want to record these data points on catalogue pages, where multiple products are listed. In this case, we could pass the three data points (`productSku`, `productName` and `productPrice`) in for each product as follows:

```html
<body>
	<script type="text/javascript">
		dataLayer = [{
      		'products': [
	            	{'productSku': 'pbz00123', 'productName': 'The Original Rider Waite Tarot cards', 'productPrice': '9.99' },
	            	{'productSku': 'pbz00124', 'productName': 'Aleicester Crowley Thoth Tarot', 'productPrice': '12.99' },
	            	{'productSku': ...},
	            	{'productSku': ...},
            		...
            ]
    	}];
	</script>
	 <!-- Google Tag Manager -->
  	...
  	<!-- End Google Tag Manager -->
</body>
```

We could use this data to perform an analysis of which product on catalogue pages are most likely to be clicked on, and whether that varies by user segment, for example. Analogous data points related to videos and listing the videos displayed on a selection page could be used by a Youtube-like site that wanted to analyse what drove users to select particular videos from a selection.

In many cases, however, we will want to capture data related to specific events that occur on a user's journey, like playing a video, when those events occur, rather than when a page loads. In these cases, we can use `dataLayer.push` function to add new data points to the layer when an AJAX event like watching a video occurs. For example, we might trigger the following with a video play:

```javascript
dataLayer.push(
	'event': 'playVideo',
	'videoId': 'vid-000-123',
	'videoName': 'Skateboarding dog',
	'videoAuthor': 'user-00121'
	'videoFormat': 'hd'
);
```

It is **critical** that whenever we want to capture data related to a specific event when that event occurs, the `dataLayer.push` function includes a field called `event` with a value set to the event type. That is because GTM uses the setting of value of the `event` field to trigger the firing of tags setup in GTM, and is hence essential to making Snowplow event tracking possible (because the Snowplow event tracking tag needs to fire following specific events) and the Snowplow ecommerce tracking possible (because that is, equally, triggered by specific purchase events.)

Now that we know *what* data we want to capture in the `dataLayer`, and *when* we want to capture each data point (either at page load time or with each event), we are in a position to finalise the documentation for the webmaster that makes it clear what variables to set in the dataLayer, and when. This will be the basis for [integrating the `dataLayer` onto the website](#1.5) below.

#### Note: GA suggestions for data points and field names in the `dataLayer`

Google has a number of suggestions for what fields should be captured in the `dataLayer`, and how they should be labelled. Those can be found [here][datalayer].

For the most part, these are just *suggestions*. However, some of them are more than that. For example, if you want to use either Google Analytics ecommerce tracking, or Snowplow's own ecommerce tracking (which is closely modelled on Google's approach), you will need to set *specific* variables in your `dataLayer`. Instructions on how to do this is covered below in the corresponding section on [ecommerce tracking](#ecommerce).


<a name="1.5" />

### 1.5 Integrate the `dataLayer` onto your website

Now that we've decided (and documented) what data to push to the `dataLayer`, and when, your webmaster should be in a position to integrate the `dataLayer` calls into your website.

Note - although we've separated this step out from step 1.2 [integrating your container tag on your website](#1.2), in practice you'd want to carry out both these steps simultaneously.

<a name="1.6" />

### 1.6 Create variables stored in the `dataLayer` in the GTM UI

Passing data into GTM via the `dataLayer` is great - but to get any value from that data, we need to be able to pass it on to the tags that GTM manages, including Snowplow.

Doing so is simple, if a little time consuming. For every top-level data field passed into GTM (e.g. `products`, `videoId` etc.), we need to create a `dataLayer` variable in GTM. The value of this variable will be set when the value is passed into the `dataLayer`, and we'll be able to pass the variable into any tags that are setup in GTM. (Instructions on how to do this for Snowplow tags will be given in the section on [integrating Snowplow](#snowplow-setup) below.)

> [**Variables**](https://support.google.com/tagmanager/answer/6106899) are name-value pairs for which the value is populated during runtime. For example, the predefined variable named "url" has been defined such that its value is the current page URL.

<p></p>

> [**Built-in (predefined) variables**](https://support.google.com/tagmanager/answer/6106965) are a special category of variables that are pre-created and non-customizable. They replace the variables that used to be generated when creating a new container.

To create a new custom variable in GTM, click on the **NEW** button at the bottom of the GTM screen (when you are logged into your Account » Container » Variables):

[[/setup-guide/images/gtm/create-macro-1.png]]

Give your variable an appropriate name. For simplicity, we always use the same name used in the `dataLayer`, although that is not a requirement. Select **Data Layer Variable** type. Then enter the name of the variable, exactly as used in the HTML / JavaScript. (This is case sensitive.)

[[/setup-guide/images/gtm/create-macro-2.png]]

Save the new variable. It will now be visible in the container setup under "User-Defined Variables" section:

[[/setup-guide/images/gtm/create-macro-3.png]]

#### Some notes on variables

GTM comes pre-configured with a few (built-in) variables already, in particular `event`, `referrer` and `url`. The `event` is especially important. You can set GTM up to trigger the firing of tags when specific event types are pushed to the `dataLayer`. We use this to trigger both Snowplow event tracking tags, and ecommerce tracking tags, as explained [below](#snowplow-setup).

[[/setup-guide/images/gtm/create-macro-4.png]]

[Back to top](#top)

<a name="snowplow-setup" />

## 2. Integrating Snowplow JavaScript tracking tags with Google Tag Manager

1. [Integrating Snowplow page tracking tags](#page)
2. [Integrating Snowplow structured event tracking tags](#events)
3. [Integrating Snowplow ecommerce tracking tags](#ecommerce)
4. [Tracking other events (campaigns, page pings)](#other-events)
5. [Publishing changes to GTM](#publish)

<a name="page" />

### 2.1 Integrating Snowplow page tracking tags

This is the simplest tag to integrate.

First, click on the **New** button within **Tags** section of the container.

[[/setup-guide/images/gtm/integrate-page-tracker-1.png]]

Give the tag a suitable name e.g. "Snowplow page tracker" and select **Custom HTML Tag** as your **product**. Then paste in the standard Snowplow page tracking code in the HTML box:

[[/setup-guide/images/gtm/integrate-page-tracker-2.png]]

<!---
[[/setup-guide/images/gtm/integrate-page-tracker-3.png]]
--->

The actual code you need to insert is:

```html
<!-- Snowplow starts plowing -->
<script type="text/javascript">
;(function(p,l,o,w,i,n,g){if(!p[i]){p.GlobalSnowplowNamespace=p.GlobalSnowplowNamespace||[];
p.GlobalSnowplowNamespace.push(i);p[i]=function(){(p[i].q=p[i].q||[]).push(arguments)
};p[i].q=p[i].q||[];n=l.createElement(o);g=l.getElementsByTagName(o)[0];n.async=1;
n.src=w;g.parentNode.insertBefore(n,g)}}(window,document,"script","//d1fc8wv8zag5ca.cloudfront.net/2.5.1/sp.js","snowplow"));

window.snowplow('newTracker', 'cf', '{{MY-COLLECTOR-URI}}', { // Initialise a tracker
  appId: '{{MY-SITE-ID}}',
  cookieDomain: '{{MY-COOKIE-DOMAIN}}'
});

window.snowplow('trackPageView');
</script>
<!-- Snowplow stops plowing -->
```

You will need to update {{MY-COLLECTOR-URI}} with the Cloudfront subdomain details you created as part of the [collector setup](https://github.com/snowplow/snowplow/wiki/setting-up-the-cloudfront-collector). (If you are using a version of Snowplow hosted by the Snowplow team, we will provide you with a Cloudfront domain to enter.) It will look something like `d3rkrsqld9gmqf.cloudfront.net`. If you are using a different collector the Cloudfront collector (e.g. the Clojure collector), use the URI of that collector instead. There is no need to include a protocol ("http" or "https") - the tracker does this automatically.

The `appId` and `cookieDomain` fields are optional: the first is used if you are running Snowplow across different applications and want to distinguish data for each easily. Setting a custom cookie domain is useful if you are tracking users across multiple subdomains e.g. 'blog.mysite.com', 'www.mysite.com', 'mysite.com'. Full instructions on the use of both these methods can be found in the [JavaScript Tracker technical documentation](Javascript-Tracker).

If you are hosting your own Snowplow JavaScript file (see the guide to [self-hosting snowplow.js](self hosting snowplow js)), then you need to update the tag above, swapping your own Cloudfront `{{SUBDOMAIN}}` (the one from which you serve `sp.js`) in for ours:

```javascript
;(function(p,l,o,w,i,n,g){if(!p[i]){p.GlobalSnowplowNamespace=p.GlobalSnowplowNamespace||[];
p.GlobalSnowplowNamespace.push(i);p[i]=function(){(p[i].q=p[i].q||[]).push(arguments)
};p[i].q=p[i].q||[];n=l.createElement(o);g=l.getElementsByTagName(o)[0];n.async=1;
n.src=w;g.parentNode.insertBefore(n,g)}}(window,document,"script","//{{SUBDOMAIN}}.cloudfront.net/sp.js.cloudfront.net/2.6.1/sp.js","snowplow"));
```
<!---
[[/setup-guide/images/gtm/integrate-page-tracker-4.png]]
--->

Once the tag is inserted, we need to instruct GTM to fire this tag on every page load. It is straightforward here: click on **Continue** button. This will apply the default tag firing option which is "Once per event".

[[/setup-guide/images/gtm/integrate-page-tracker-5.png]]

On the **Fire On** step select **All pages**. This will ensure that the tag is fired for every page load. (If you had tags on e.g. `www.mysite.com` and `test.mysite.com`, you could create an exception filter with a different URL regexp to only fire the tags on the production, rather than test site, and assign that rule to the tag.)

[[/setup-guide/images/gtm/integrate-page-tracker-6.png]]

Now you're done - click **Create Tag**. The new tag should be listed in the container summary screen.

[[/setup-guide/images/gtm/integrate-page-tracker-7.png]]

**Note**: although set up, the tag won't fire until this update is **published**. We cover how to publish the configurations made above in [section 2.4 below](#other-events).

<a name="events" />

### 2.2 Integrating Snowplow structured event tracking tags

This is best explained through an example. Let's assume, again, that we are implementing Snowplow on a Youtube-like video site. There will be a large number of possible events that occur on a user journey including searching for a video, playing a video, pausing a video, liking a video, uploading a video, sharing a video etc... As per steps [1.3](#1.3), [1.4](#1.4) and [1.5](#1.5), each time one of these events occurs, the event is logged in the `dataLayer` along with the associated data. For example, for a 'video play', the following JavaScript would execute:

```javascript
dataLayer.push(
	'event': 'playVideo',
	'videoId': 'vid-000-123',
	'videoFormat': 'hd'
);
```

We now need to set up the Snowplow event tracking tag to fire for each of the types of event identified, and use the five [event tracking fields][event-tracking] to pass the data into Snowplow. Those fields are documented [here][event-tracking], but for ease we list them below:

1. Event category
2. Event action
3. Event label
4. Event property
5. Event value

Let's work through the process for our `playVideo` event type. From the main Tag Manager screen, click the **New** button. Give the tag a suitable name e.g. 'Snowplow playVideo event' and select the **Custom HTML Tag** as we showed you earlier.

<!---
[[/setup-guide/images/gtm/event-tracking-2.JPG]]
--->

In the HTML box below, we need to paste the Snowplow event tracking code. The generic code looks as follows:

```html
<!-- Snowplow event tracking -->
<script type="text/javascript">
window.snowplow('trackStructEvent', {{CATEGORY}}, {{ACTION}}, {{LABEL}}, {{PROPERTY}}, {{VALUE}});
</script>
```

We need to insert the above code, but substitute in the data points captured when a video is played in the `dataLayer`. To do that, we need to map those fields i.e.:

1. `playVideo`
2. `videoId`
3. `videoFormat`

to the five structured event tracking fields. (Note - if 5 fields are not enough, we are in the process extending Snowplow to accommodate 10 additional event custom variables and an 11th arbitrary JSON, that can be stuffed with additional data.)

How we do the mapping is really a matter of preference: the following would work:

| **`playVideo` data point** | **Event tracking field** |
|:---------------------------|:-------------------------|
|                            | `category`               |
| `playVideo`                | `action`                 |
| `videoId`                  | `label`                  |
| `videoFormat`              | `property`               |
|                            | `value`                  |

It would make sense to hardcode the `category` to `video` or `media` (so we can group all video related actions in our analysis), and to leave the `value` field blank. (This field could be used if there's revenue that can be attributed to the `playVideo` event, for example.)

To implement the above mapping, we update the HTML in the box to the following:

```html
<!-- Snowplow structured event tracking -->
<script type="text/javascript">
window.snowplow('trackStructEvent', 'video', 'playVideo', {{videoId}}, {{videoFormat}}, '0.0');
</script>
```

The above code will dynamically insert the values in the `dataLayer` for `videoId` and `videoFormat` into the tag, which will pass the data into Snowplow. Note that this will only work if you have setup the corresponding variables in GTM for `videoId` and `videoFormat`, as documented in [section 1.6](#1.6).

[[/setup-guide/images/gtm/event-tracking-1.png]]

Now we need to tell GTM to fire it every time a `playVideo` event occurs. To do that, click on the **Continue** button, which will apply the default "Once per event" firing option. Under **Fire On** section, click on **More** button.

[[/setup-guide/images/gtm/event-tracking-2.png]]

This will bring up the list of existing triggers.

[[/setup-guide/images/gtm/event-tracking-3.png]]

Click on the **New** button and select **Custom Event** as the event type. Give the trigger and the event an appropriate name and set it to **event equals `playVideo`**:

[[/setup-guide/images/gtm/event-tracking-4.png]]

Hit **Create Trigger** button. It should now be added to list of triggers (we created only one) firing the tag.

[[/setup-guide/images/gtm/event-tracking-5.png]]

Click on **Create Tag** button. The newly created tag is added to the list of the existing tags.

[[/setup-guide/images/gtm/event-tracking-6.png]]

You now need to repeat the above process for each of the different types of events identified in your `dataLayer` i.e. `uploadVideo`, `shareVideo`, `likeVideo` etc. In each case, you will need to:

1. Create a new tag in GTM i.e. `Snowplow shareVideo event`, `Snowplow uploadVideo event`
2. Paste in the Snowplow event tracking code, but map the fields for the specific events into the 5 Snowplow event tracking fields
3. Trigger the tag to fire when e.g. `event: uploadVideo` is passed into the `dataLayer`

Once you have completed creating all the tags, you will need to [publish the updates](#publish) (as detailed [below](#publish)) in order to push the changes live.

#### An alternative, less time-consuming approach

The above approach is time consuming because it requires that you create a Snowplow tag in GTM for every different type of event identified in the `dataLayer`.

An alternative approach would be not to distinguish different types of event in the `dataLayer`. Instead, for every type of custom event, use the same set of fields in the `dataLayer` that correspond to the five Snowplow event tracking fields i.e.:

```javascript
dataLayer.push({
	'event': 'customEvent',
	'eventCategory': 'xxx',
	'eventAction': 'xxx',
	'eventLabel': 'xxx,
	'eventProperty': 'xxx',
	'eventValue': 'x.x'
});
```

Then we can create a single Snowplow event tag in GTM that is fired with every `customEvent` and maps the fields in the `dataLayer` directly into the Snowplow event tracking tag.

```html
<!-- Snowplow structured event tracking -->
<script type="text/javascript">
window.snowplow('trackStructEvent', '{{eventCategory}}', '{{eventAction}}', '{{eventLabel}}', '{{eventProperty}}', '{{eventValue}}');
</script>
```

There are several reasons we do not recommend this approach:

1. Although time consuming, the exercise of identifying all the relevant events that can occur on a user's journey, and all the data points that are associated with each, is incredibly useful. At analysis time, you would use the same mapping used to match those data points against the generic Snowplow event fields, to map them back into their 'idealised structure'.
2. We're working to improve Snowplow all the time. If the current event tracking isn't flexible enough to accommodate your needs, it's highly likely that a future version will be. If you're not capturing all the data that you ideally want to be reporting on, in GTM, then when we upgrade Snowplow, you won't be able to take advantage without updating your GTM implementation.
3. The whole point of using a tag management platform like GTM is to manage all your tags. The alternative approach outlined above is a Snowplow specific hack and will not necessarily work for other tags that are managed through GTM. In order to make it as easy as possible manage the widest variety of tags, you should pass all the data you might want to share with your tags into GTM in a sensible, comprehensible format, rather than one geared around a specific providers tag.

<a name="ecommerce" />

### 2.3 Integrating Snowplow ecommerce tracking tags

Snowplow's [ecommerce tracking][ecom-tracking] closely follows ecommerce tracking as implemented in Google Analytics. We have done this to make implementing Snowplow alongside GA as easy as possible.

Because of this, however, it is important that you follow Google's specified approach to pushing transaction related data into the `dataLayer`, using the same field names and data structures used by Google. These are listed on the [Google website][gtm-vars] and are repeated below for clarity:

| **Variable name**       | **Description**                 |
|:------------------------|---------------------------------|
| `transactionId`         | Unique transaction ID           |
| `transactionAffiliation`| Affiliation or store name       |
| `transactionTotal`      | Transaction value               |
| `transactionShipping`   | Shipping charge                 |
| `transactionTax`        | Tax (VAT) charge                |
| `transactionPaymentType` | Payment type (e.g. credit card)|
| `transactionCurrency`   | Currency of transaction         |
| `transactionShippingMethod` | Selected shipping method    |
| `transactionPromoCode`  | Discount or promotion codes used |
| `transactionProducts`   | List of items purchased in the transaction (provided to the `dataLayer` as an array) |

The following data is supported for each of the products in the `transactionProducts` array:

| **Variable name**       | **Description**                 |
|:------------------------|---------------------------------|
| `id`                    | Product ID                      |
| `name`                  | Product name                    |
| `sku`                   | Product SKU                     |
| `category`              | Product category                |
| `price`                 | Unit price                      |
| `quantity`              | Number of this item included in the order |

In addition, there are a number of additional data points that it is possible to capture using Snowplow ecommerce tracking through GTM, but we are unclear if Google supports. (GA ecommerce tracking supports the fields, but it is not clear if the GTM implementation of GA's ecommerce tracking supports them):

| **Variable name**       | **Description**                 |
|:------------------------|:--------------------------------|
| `transactionCity`       | City buyer is located in        |
| `transactionState`      | State or province buyer is located in |
| `transactionCountry`    | Country buyer is located in     |

In the `dataLayer`, a transaction needs to be recorded like the following example:

```javascript
dataLayer.push({
	'event': 'transaction',
	'transactionId': 'order-123',
	'transactionAffiliation': 'web',
	'transactionTotal': '107.98',
	'transactionShipping': '2.99',
	'transactionTax': '2',
	'transactionCurrency': 'GBP',
	'transactionProducts': [
		  {'name': 'Blue_t-shirt', 'sku': '1001', 'category': 'ts', 'price': '4.99', 'quantity': '1'},
		  {'name': 'Red_shoes', 'sku': '1002', 'category': 'shoes', 'price': '50.00', 'quantity': '2'}
	],
	'transactionCity': 'London',
	'transactionCountry': 'United Kingdom'
});
```

As described in [section 1.6](#1.6), we need to create variables in GTM for each of the fields listed in the above three tables i.e. `transactionId`, `transactionAffiliation`... `transactionCountry`. To recap on the process:

1. Head to **User-Defined Variables** section and click on the **New** button in the GTM interface
2. Name the new variable being created. (We recommend using exactly the same name used in the `dataLayer` e.g. `transactionId` for clarity)
3. Set the variable type to 'Data Layer Variable'
4. Set the value of the *Data Layer Variable Name* to the field name given in the `dataLayer` e.g. `transactionId`
5. Save the variable

We then need to create a Snowplow ecommerce tracking tag in GTM that passes the data onto Snowplow on a transaction event. In GTM, head to **Tags** and click on the **New** button. Give the tag a sensible name e.g. 'Snowplow ecommerce tracker' and select '*Custom HTML Tag*' in the Product. Now in the HTML box, paste the following code:

```html
<script type="text/javascript">
// 1st, fire 'addTrans' event for the new transaction
window.snowplow('addTrans',
    {{transactionId}},
    {{transactionAffiliation}},
    {{transactionTotal}},
    {{transactionTax}},
    {{transactionShipping}},
    {{transactionCity}},
    {{transactionState}},
    {{transactionCountry}}
);

// 2nd fire the 'addItem' event for each item included in the transaction
for(i=0; i<{{transactionProducts}}.length; i++) {
    window.snowplow('addItem',
        {{transactionId}},
        {{transactionProducts}}[i].sku,
        {{transactionProducts}}[i].name,
        {{transactionProducts}}[i].category,
        {{transactionProducts}}[i].price,
        {{transactionProducts}}[i].quantity
    );
}

// Finally fire the 'trackTrans' event to commit the transaction
window.snowplow('trackTrans');
</script>
```

Note: if you did not name each of the transaction variables with the same names as specified in the `dataLayer` e.g. `transactionId`, you will need to update the references to those variable names in the above tag accordingly.

Now that our tag is ready, we need to trigger it to fire. Assuming we identify when transactions occur in the `dataLayer` using `dataLayer.push({ 'event': 'transaction', ...});`, we'll want to fire the Snowplow tag every time **event equals `transactions`**. To do this, click on **More** under **Fire On** and create the triger using **Custom Event**.

[[/setup-guide/images/gtm/ecomm-tracking-1.png]]

The tag setup is now complete:

[[/setup-guide/images/gtm/ecomm-tracking-2.png]]

Save the tag. We are now ready to publish the changes. This is covered in the [next section](#publish).

<a name="other-events" />

### 2.4 Tracking other events (campaigns, page pings etc.)

As well as the page view, structured events and ecommerce event tracking tags, Snowplow has specific functionality to enable the capture of other event data including:

1. [Page pings](2-Specific-event-tracking-with-the-Javascript-tracker#wiki-pagepings). Use this to track how long visitors dwell on each page on your site, and how they scroll of pages over time.

Detailed documentation on how to capture the complete range of events possible with Snowplow can be found in the [[Javascript Tracker]] section of the [Technical Documentation](snowplow-technical-documentation).

Note: we recommend after you finish consulting the technical documentation related to the events supported by the JavaScript Tracker that you return to the setup guide to complete the setup.

<a name="publish" />

### 2.5 Publishing changes to GTM

Once you have setup all your tags, triggers and variables in GTM, you need to publish the changes before they will take effect on your website(s).

This is a three step process:

1. Preview and debug
2. Create a **version** with all the most recent changes / updates
3. When you are confident it is working as expected, publish it

Note: Your tags, triggers, and variables are validated on attempt to publish. You could start from any of the above processes as GTM will lead you in the above order anyway. That is if you start, say, from the last step ("Publish") GMT will check you work for validity first and it won't let you to proceed until you have debugged your variables and HTML code. If all is good, it will create a version automatically before publishing. Likewise, when trying to create a version first you will be forced to correct the errors (if any) in *debug* mode first.

Choosing any of the above steps is simple: Click on the button next to **Publish** to bring the dropdown menu up listing the steps.

[[/setup-guide/images/gtm/publish-1.png]]

Click on **Preview** to check for any problem. If GMT validation fails it will report it back with the screen which looks like the one below. You might need to scroll down to see the full list. You are in *debug* mode.

[[/setup-guide/images/gtm/publish-2.png]]

The location field contains the links to the problematic configuration. Follow the links to either check and correct your code or find out more about the problem. When all is fixed  preview the tags again. Since all the errors have been corrected you will be switched to *preview* (as opposed to *debug*) mode.

[[/setup-guide/images/gtm/publish-3.png]]

Now load a web page with the container in a tab in the same browser. You should see an additional GTM interface in the bottom half of the screen that indicates when different tags defined in the UI have been fired.

[[/setup-guide/images/gtm-debug.png]]

If you are happy that the setup works as expected, click the **Publish** button. The updates will be pushed to live.

In the event that it is not working as expected, you can go back and make the changes you require. You will need to create a new version, with the updates, before you can either preview them, or publish them.

[Back to top](#top)

<a name="enhanced-ecommerce" />

## 3. Integrating Snowplow JavaScript tracking tags with enhanced ecommerce

You can send [enhanced ecommerce][enhancedEcommerce] data to Snowplow as well as Google Analytics. For more information, see [Integrating JavaScript tags with enhanced ecommerce](https://github.com/snowplow/snowplow/wiki/Integrating-Javascript-tags-with-enhanced-ecommerce).

<a name="next-steps" />

## 4. Next steps

Now you have setup the JavaScript tracking tags, you are in a position to [test that they fire](testing the javascript tracker is firing).

[Return to setup guide](Setting-up-Snowplow).

[datalayer]: https://developers.google.com/tag-manager/reference
[event-tracking]: https://github.com/snowplow/snowplow/wiki/2-Specific-event-tracking-with-the-Javascript-tracker#tracking-specific-events
[ecomm-tracking]: https://github.com/snowplow/snowplow/wiki/javascript-tracker#wiki-ecommerce
[gtm-vars]: https://developers.google.com/tag-manager/reference#varnames
[enhancedEcommerce]: https://developers.google.com/analytics/devguides/collection/analyticsjs/enhanced-ecommerce
