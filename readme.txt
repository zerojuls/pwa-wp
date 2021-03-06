=== PWA ===
Contributors:      xwp, google, automattic
Tags:              pwa, progressive web apps, service workers, web app manifest, https
Requires at least: 4.9
Tested up to:      4.9
Stable tag:        0.1.0
License:           GPLv2 or later
License URI:       http://www.gnu.org/licenses/gpl-2.0.html
Requires PHP:      5.2

WordPress feature plugin to bring Progressive Web App (PWA) capabilities to Core

== Description ==

<blockquote cite="https://developers.google.com/web/progressive-web-apps/">
Progressive Web Apps are user experiences that have the reach of the web, and are:

<ul>
<li><a href="https://developers.google.com/web/progressive-web-apps/#reliable">Reliable</a> - Load instantly and never show the downasaur, even in uncertain network conditions.</li>
<li><a href="https://developers.google.com/web/progressive-web-apps/#fast">Fast</a> - Respond quickly to user interactions with silky smooth animations and no janky scrolling.</li>
<li><a href="https://developers.google.com/web/progressive-web-apps/#engaging">Engaging</a> - Feel like a natural app on the device, with an immersive user experience.</li>
</ul>

This new level of quality allows Progressive Web Apps to earn a place on the user's home screen.
</blockquote>

Continue reading more about [Progressive Web Apps](https://developers.google.com/web/progressive-web-apps/) (PWA) from Google.

In general a PWA depends on the following technologies to be available:

* [Service Workers](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API)
* [Web App Manifest](https://developer.mozilla.org/en-US/docs/Web/Manifest)
* HTTPS

This plugin serves as a place to implement support for these in WordPress with the intention of being proposed for core merge, piece by piece.

**Development of this plugin is done [on GitHub](https://github.com/xwp/pwa-wp). Pull requests welcome. Please see [issues](https://github.com/xwp/pwa-wp/issues) reported there before going to the [plugin forum](https://wordpress.org/support/plugin/pwa).**

= Web App Manifest =

As noted in a [Google guide](https://developers.google.com/web/fundamentals/web-app-manifest/):

> The [web app manifest](https://developer.mozilla.org/en-US/docs/Web/Manifest) is a simple JSON file that tells the browser about your web application and how it should behave when 'installed' on the users mobile device or desktop.

The plugin exposes the web app manifest via the REST API at `/wp-json/app/v1/web-manifest`. A response looks like:

<pre lang=json>
{
    "name": "WordPress Develop",
    "short_name": "WordPress",
    "description": "Just another WordPress site",
    "lang": "en-US",
    "dir": "ltr",
    "start_url": "https://example.com",
    "theme_color": "#ffffff",
    "background_color": "#ffffff",
    "display": "minimal-ui",
    "icons": [
        {
            "sizes": "192x192",
            "src": "https://example.com/wp-content/uploads/2018/05/example-192x192.png",
            "type": "image/png"
        },
        {
            "sizes": "512x512",
            "src": "https://example.com/wp-content/uploads/2018/05/example.png",
            "type": "image/png"
        }
    ]
}
</pre>

A `rel=manifest` link to this endpoint is added at `wp_head`.

The manifest is populated with default values including:

* `name`: the site title from `get_option('blogname')`
* `short_name`: truncated site title
* `description`: the site tagline from `get_option('blogdescription')`
* `lang`: the site language from `get_bloginfo( 'language' )`
* `dir`: the site language direction from `is_rtl()`
* `start_url`: the home URL from `get_home_url()`
* `theme_color`: a theme's custom background via `get_background_color()`
* `background_color`: also populated with theme's custom background
* `display`: `minimal-ui` is used as the default.
* `icons`: the site icon via `get_site_icon_url()`

There is a `web_app_manifest` filter which is passed the above array so that plugins and themes can customize the manifest.

See [labeled GitHub issues](https://github.com/xwp/pwa-wp/issues?q=label%3Aweb-app-manifest) and see WordPress core tracking ticket [#43328](https://core.trac.wordpress.org/ticket/43328).

= Service Workers =

As noted in a [Google primer](https://developers.google.com/web/fundamentals/primers/service-workers/):

> Rich offline experiences, periodic background syncs, push notifications—functionality that would normally require a native application—are coming to the web. Service workers provide the technical foundation that all these features rely on.

Only one service worker can be controlling a page at a time. This has prevented themes and plugins from each introducing their own service workers because only one wins. So the first step at adding support for service workers in core is to provide an API for themes and plugins to register scripts and then have them concatenated into a script that is installed as the service worker. There are two such concatenated service worker scripts that are made available: one for the frontend and one for the admin. The frontend service worker is installed under the `home('/')` scope and the admin service worker is installed under the `admin_url('/')` scope.

The API is implemented using the same interface as WordPress uses for registering scripts; in fact `WP_Service_Workers` is a subclass of `WP_Scripts`. The instance of this class is accessible via `wp_service_workers()` in the same way as `wp_scripts()`. Instead of using `wp_register_script()` the service worker scripts are registered using `wp_register_service_worker()`. This function takes four arguments:

* `$handle`: The service worker script handle which can be used to mark the script as a dependency for other scripts.
* `$src`: The URL to the service worker _on the local filesystem_ or a callback function which returns the script to include in the service worker.
* `$deps`: An array of service worker script handles that a script depends on.
* `$scope`: Whether a script is included in the frontend service worker (`WP_Service_Workers::SCOPE_FRONT`), the wp-admin service worker (`WP_Service_Workers::SCOPE_ADMIN`), or both (`WP_Service_Workers::SCOPE_ALL`).

Note that there is no `$ver` (version) parameter because browsers do not cache service workers so there is no need to cache bust them.

Here are some examples:

<pre lang=php>
// Register script to only run on the frontend, with dependency on some other app-shell SW script.
wp_register_service_worker(
    'foo', // Handle.
    plugin_dir_url( __FILE__ ) . 'foo.js', // Source.
    array( 'app-shell' ), // Dependency.
    WP_Service_Workers::SCOPE_FRONT // Scope.
);

// Register script (here via render callback instead of URL) to only run only in the admin.
wp_register_service_worker(
    'bar',
    function() {
        return 'console.info( "Hello admin!" );';
    },
    array(), // No deps.
    WP_Service_Workers::SCOPE_ADMIN
);

// Register script with to run in both the frontend and wp-admin.
wp_register_service_worker( 'baz', plugin_dir_url( __FILE__ ) . 'baz.js', array(), WP_Service_Workers::SCOPE_ALL );

// The default values for $deps and $scope are array() and WP_Service_Workers::SCOPE_ALL respectively, so this is same as previous.
wp_register_service_worker( 'baz', plugin_dir_url( __FILE__ ) . 'baz.js' );
</pre>

The next step for service workers in the feature plugin is to explore the use of [Workbox](https://developers.google.com/web/tools/workbox/) to power a higher-level PHP abstraction for themes and plugins to indicate the routes and the caching strategies in a declarative way (with detection for conflicts).

See [labeled GitHub issues](https://github.com/xwp/pwa-wp/issues?q=label%3Aservice-workers) and see WordPress core tracking ticket [#36995](https://core.trac.wordpress.org/ticket/36995).

= HTTPS =

HTTPS is a prerequisite for progressive web apps. A service worker is only able to be installed on sites that are served as HTTPS. For this reason core's support for HTTPS needs to be further improved, continuing the great progress made over the past few years.

At the moment the plugin provides an API to detection of whether a site supports HTTPS. Building on that it's intended that this can then be used to present a user with an opt-in to switch over to HTTPS, which will also then need to include support for rewriting URLs from HTTP to HTTPS. See [labeled GitHub issues](https://github.com/xwp/pwa-wp/issues?q=label%3Ahttps) and see WordPress core tracking ticket [#28521](https://core.trac.wordpress.org/ticket/28521).

== Changelog ==

= 0.1.0 (2018-07-12) =

* Adds support for web app manifests which can be customized via a `web_app_manifest` filter.
* Adds initial support for service workers via `wp_register_service_worker()`.
* Adds an API for detecting whether HTTPS is available for a given site.

