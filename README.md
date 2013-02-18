ResponsivePlugin
================

Manages the buildup and teardown of jQuery plugins at different media queries.

Relies on enquire.js - http://wicky.nillia.ms/enquire.js/

___

## Why use this plugin?
Ever have a situation on a responsive website where you wanted a plugin to change it's behavior at different media query breakpoints?  Maybe a slider shouldn't exist on screens smaller than 600px wide, or a photo gallery should switch to a more vertical-friendly layout on smaller devices.  An even simpler use case to consider is when a plugin needs to behave differently for tablet-horizontal vs. tablet-vertical.  Unfortunately, a lot of plugin authors do not provide a way to destroy an instance of a plugin.  And even worse, the plugins which can destroy themselves very rarely restore the original DOM which existed before the plugin went and inserted its slimy greasy nodes into your clean semantic markup.  And holy macaroni if the these self-destroying plugins actually unbind *all* events which were registerd during its lifetime.

### Enter Responsive Plugin
See if you can follow what is going on:

```javascript
(function($) {
    var breakpoints = {
	    	MAX_WIDTH_48EM: "not screen and (min-width: 48em)",
			MIN_WIDTH_48EM: "screen and (min-width: 48em)" 
	    },
	    breakpointOptions = {};
	    
    breakpointOptions[ breakpoints.MAX_WIDTH_48EM ] = {
    	options: {
    		selector: ".slides > li",
    		direction: "vertical"
    	}
    };
    breakpointOptions[ breakpoints.MIN_WIDTH_48EM ] = {
    	options: {
    		selector: ".slides > li",
    		direction: "horizontal"
    	},
    	callbackBefore: function() {
    		console.log("Getting ready to destroy any existing flexslider, restore the original document fragment, and create a new instance of flexslider at for media query: " + breakpoints.MIN_WIDTH_48);
    	},
    	callbackAfter: function() {
    		console.log("Just instantiated a flexslider for media query: " + breakpoints.MIN_WIDTH_48);
    	}
    };
    
	$('.slider').responsivePlugin({
	    pluginName: "flexslider",
	    breakpointOptions: breakpointOptions
	});
})(jQuery);

enquire.listen();
```

Yup, you got it!  A vertical slider on small screens, and a horizontal slider on larger screens.  If the user happens to cross over into a different media query, the original slider will be completely torn down, all events unbound, original DOM restored, and a new slider instantiated.

## Basic Usage
### Default options:

```javascript
$('.some-element').responsivePlugin({
    pluginName: "",					// The name of the plugin as you would normally use it in code: $('.some-element').pluginName();
    destroyMethodName: "destroy",	// The name of the destroy method (see below for detailed description)
    breakpointOptions: {			// A list of options and callbacks for each media query breakpoint
	    "only screen and (min-width: 48em)": {
	    	options: {},			// plugin options
	    	callbackBefore: null, 	// Called before any destroying, unbinding, DOM restoration, etc
	    	callbackAfter: null, 	// After the new plugin has been instantiated
	    	restoreDOM: true 		// whether or not the original DOM should be restored.  Anything other than FALSE will pass as TRUE... even undefined
	    }
	}
});

// We take care of calling enquire.fire(), but you must remember to call enquire.listen().
// You should only do this once in your application after all enquire-related stuff is done
enquire.listen(); 
```

### How to use the `destroyMethodName` option

##### As a string
- Some plugin authors store a reference to the plugin instance in the node data.  We try to find this reference by doing `$instance = $node.data( options.pluginName );`.   If we find something there, we try to call the destroy method on the instance itself: `$instance[ options.destroyMethodName ]();`.
- Next, we pass the `destroyMethodName` to the plugin method... similar to the way jQuery UI `destroy` works.  This should gracefully fail if the plugin author did not account for this.

##### As a function
- If you pass a function reference, we will call that function.

### NOTES

- All callback functions and destroy functions are scoped to the jQuery object on which the plugin was instantiated.

```javascript
$('.some-element').responsivePlugin({
    pluginName: "flexslider",
    breakpointOptions: {
	    "only screen and (min-width: 48em)": {
	    	options: {},			
	    	callbackAfter: function() {
	    		// this === $('.some-element')
	    	}
	    }
	}
});
```


## Advanced Usage

- Pass "destroy" to one of the breakpoints OR options to simply restore the document fragment to its original pre-plugin state.

```javascript
	breakpointOptions: {
	    "only screen and (max-width: 48em)": "destroy"
	}
	
	breakpointOptions: {
	    "only screen and (max-width: 48em)": {
	    	options: "destroy"
	    }
	}
```

- Pass "hide" to one of the breakpoints OR options to hide the entire document fragment for a particular breakpoint.  
NOTE: hiding takes place **after** all events have been unbound and the original document fragment restored.

```javascript
	breakpointOptions: {
	    "only screen and (max-width: 48em)": "hide"
	}
	
	breakpointOptions: {
	    "only screen and (max-width: 48em)": {
	    	options: "hide"
	    }
	}
```

- Pass FALSE to the `restoreDOM` option if you **do not** want the original document fragment restored for a particular breakpoint.

```javascript
	breakpointOptions: {
		"only screen and (max-width: 48em)": {
			options: "destroy",
			restoreDOM: FALSE // Must be a boolean false.  Anything else will pass as TRUE, even undefined
		}
	}
```

## How do you know that all events are properly unbound?
We temporarily hijack the jQuery `$.fn.on` method while the plugin is creating itself.  As the plugin adds hundreds of event handlers to your page, we append our own custom namespace to each event and keep a reference to the node on which the event is bound.  When it's time to tear down the plugin, we loop over each event and `.off()` it.  The reason this works is because all calls to `bind()`, `delegate()`, `one()`, `click()`, etc get routed through the `on()` method.  It doesn't matter how the event was bound... we will catch it and remember to unbind it.  **NOTE:** all calls to `responsivePlugin` will be given their own unique event namespace so that we do not accidentally unbind any other events when the plugin is destroyed.