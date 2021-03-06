---
layout: post
title:  "Authenticating with Cookies from a Client-Side Script"
author: "K. Adam White"
author_web: https://bocoup.com/about/bocouper/kadam-white
date:   2016-08-12 16:12:00 -0400
categories: guides
---

Don't be fooled by the name, `node-wpapi` runs great in the browser&mdash;and unless you're already using backbone, we think it's _the_ simplest way to get up and running quickly with live site data in your WordPress plugin's JavaScript.

If you're using [webpack](https://webpack.github.io) or [browserify](http://browserify.org/) you can install this script with npm (`npm install --save wpapi`), then `var WPAPI = require( 'wpapi' )` in your code and you're off to the races.

But you don't have to have a build system at all to use `node-wpapi`; just [download the pre-built script bundle][script-download], then use [`wp_enqueue_script`][wp_enqueue_script] to pull it in from your plugin. The library will export itself as the global variable `WPAPI`, which you can reference in your own scripts.

Out of the box you can use the script to access all public content, but eventually you're likely to want more. In this guide, we're going to learn how to use the normal WordPress login system along with a "nonce", a unique number generated by WordPress, to authenticate your script with the API.

## Creating The Plugin

If you don't have a plugin already, let's make a quick one together. First, we'll need the [WordPress REST API v2 plugin](https://wordpress.org/plugins/rest-api/) to be installed and activated. Then in your `wp-content/plugins` directory, create the folder `wpapi-demo`, then make an empty `wpapi-demo.php` file inside that folder.

Next, download and copy over the script files from the [wpapi.js download bundle](https://wp-api.github.io/node-wpapi/wpapi.zip) into your plugin directory. At a minimum, you should now have this:

- `wp-content/`
    + `plugins/`
        * `wpapi-demo/`
            - `wpapi-demo.php`
            - `wpapi.min.js` _(the script you copied over from the download)_
            - _(Probably the rest of the files from the download are here too)_
        * `rest-api/` _(the REST API plugin itself)_

The first thing we have to do is to tell our new WordPress plugin where to find the `wpapi.js` script (the example code assumes we're using the minified version, but either will work). To do that, we need to call [`wp_register_script`][wp_register_script] within our plugin; `wp_register_script` is meant to be called within the `wp_enqueue_scripts` action, as shown below

{% highlight php %}
<?php
/**
 * Plugin Name: wpapi.js demo
 * Description: Demo of authenticating with wpapi.js in a plugin
 */

add_action( 'wp_enqueue_scripts', function() {
    // Tell WordPress where to find the web-ready wpapi script,
    // and identify it as "wpapi"
    wp_register_script(
        'wpapi',
        // This expects the script file to be next to this PHP file
        plugin_dir_url( __FILE__ ) . 'wpapi.min.js
    );

    // ...more code will go here
});
{% endhighlight %}

The library on its own doesn't do much for us unless we write a script that uses it, so let's add a `wpapi-demo.js` script to our plugin folder as well. (You can leave it empty for now.) We'll tell our WordPress plugin where to find that one, too, by adding this code:

{% highlight php %}
    // Register our own first-party script, which will use wpapi as a dependency
    wp_register_script(
        'wpapi-demo',
        // This expects the file wpapi-demo.js to be in this same plugin directory
        plugin_dir_url( __FILE__ ) . 'wpapi-demo.js',
        array( 'wpapi' )
    );
{% endhighlight %}

The final piece of the puzzle is to use [wp_localize_script][wp_localize_script] to inject a _nonce_ into our code; this unique number will let our script authenticate itself with WordPress whenever you are logged in to your site as normal.

In our JavaScript, let's prepare to create the site client object for our WordPress API; we'll use the variable `WP_API_Settings` to stand in for the values that WordPress will inject.

In that empty `wpapi-demo.js` script, add this code:

{% highlight js %}
// Access the wpapi.js functionality through the WPAPI global, which
// is created by the wpapi.js script (upon which this depends)
(function( WPAPI ) {
  'use strict';

  // These files will be "localized" by WordPress when the script is
  // enqueued. Localization in this case has nothing to do with human
  // language, but instead is used to inject a NONCE (unique number)
  // into the script, which we use to authenticate with WordPress using
  // your normal login cookie. If you are logged in to WordPress, you
  // will be logged in within this script!
  var wp = new WPAPI({
    endpoint: WP_API_Settings.root,
    nonce: WP_API_Settings.nonce
  });

  // You can now authenticate, and read/write private data!
  wp.users().me().then(function( me ) {
    console.log( 'I am ' + me.name + '!' );
  });
})( window.WPAPI );

{% endhighlight %}

And in our PHP plugin file, we'll instruct WordPress how to populate that `WP_API_Settings` object:

{% highlight php %}
    // Localize our script to inject a NONCE that can be used to authenticate
    wp_localize_script(
        'wpapi-demo',
        'WP_API_Settings',
        array(
            'root' => esc_url_raw( rest_url() ),
            'nonce' => wp_create_nonce( 'wp_rest' )
        )
    );
{% endhighlight %}

With all the preparations complete, you can add the line

{% highlight php %}
    // Enqueue our script
    wp_enqueue_script( 'wpapi-demo' );
{% endhighlight %}

to the bottom of the `wp_enqueue_script` action in your PHP code.

If you now activate our demo plugin and log in to the WordPress site, you should see "I am [your name or username]" printed out to the console: you are officially authenticated with the WordPress REST API!

## Full Sample Code

### wpapi-demo.php:
{% highlight php %}
<?php
/**
 * Plugin Name: wpapi.js demo
 * Description: Demo of authenticating with wpapi.js in a plugin
 */

add_action( 'wp_enqueue_scripts', function() {
    // Tell WordPress where to find the web-ready wpapi script
    wp_register_script(
        'wpapi',
        // wpapi.min.js should be in the same directory as this PHP file
        plugin_dir_url( __FILE__ ) . 'wpapi.min.js',
        array(),
        false,
        true // enqueue in footer
    );

    // Register our own first-party script, which depends on "wpapi"
    wp_register_script(
        'wpapi-demo',
        // wpapi-demo.js should be in the same directory as this PHP file
        plugin_dir_url( __FILE__ ) . 'wpapi-demo.js',
        array( 'wpapi' ),
        false,
        true // enqueue in footer
    );

    // Localize our script to inject a NONCE that can be used to auth
    wp_localize_script(
        'wpapi-demo',
        'WP_API_Settings',
        array(
            'root' => esc_url_raw( rest_url() ),
            'nonce' => wp_create_nonce( 'wp_rest' )
        )
    );

    // Enqueue our script
    wp_enqueue_script( 'wpapi-demo' );
});

{% endhighlight %}

### wpapi-demo.js:

{% highlight js %}
// Access the wpapi.js functionality through the WPAPI global, which
// is created by the wpapi.js script (upon which this depends)
(function( WPAPI ) {
  'use strict';

  // These files will be "localized" by WordPress when the script is
  // enqueued. Localization in this case has nothing to do with human
  // language, but instead is used to inject a NONCE (unique number)
  // into the script, which we use to authenticate with WordPress using
  // your normal login cookie. If you are logged in to WordPress, you
  // will be logged in within this script!
  var wp = new WPAPI({
    endpoint: WP_API_Settings.root,
    nonce: WP_API_Settings.nonce
  });

  // You can now authenticate, and read/write private data!
  wp.users().me().then(function( me ) {
    console.log( 'I am ' + me.name + '!' );
  });
})( window.WPAPI );
{% endhighlight %}

[script-download]: https://wp-api.github.io/node-wpapi/wpapi.zip
[wp_register_script]: https://developer.wordpress.org/reference/functions/wp_register_script/
[wp_enqueue_script]: https://developer.wordpress.org/reference/functions/wp_enqueue_script/
[wp_localize_script]: https://pippinsplugins.com/use-wp_localize_script-it-is-awesome/
