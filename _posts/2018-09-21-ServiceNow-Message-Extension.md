---
layout: post
title: Extending the duration of ServiceNow Messages on the Service Portal (Pre London)
---
How to extend the duration of Error and Info messages on the ServiceNow Service Portal to better the customer expierence. This solution provides an easy way to remove functionality for ServiceNow's enhancement in London.

## Problem
Users are unable to see the message once they fill out an item on the Service Catalog. We are not 100% sure how, but we assume they must be clicking away to another tab, or glancing at another screen. Instead they recieve the expierence of just a bunch of grayed out fields (read-only after submission).


## Quick Solution OverView
##### There is a better [solution](https://docs.servicenow.com/bundle/london-servicenow-platform/page/build/service-portal/concept/properties-service-portal.html) offered in the London Release.  
(_This is a temporary workaround that is very easy to remove via turning a script to inactive._)  
Create an angular decorator for the spNotifications directive and have it load on every portal that utilizes a specific theme. The decorator should override the `getMilliSeconds` function to return the values you want instead.


## In-Depth Solution OverView
I did begin to work on this myself and then found this [community article](https://community.servicenow.com/community?id=community_question&sys_id=823e87eddb9cdbc01dcaf3231f961931). However I don't think it does the best in a walkthrough for folks who do not have expierence in this area so I wanted to add a little extra documentation.


### Createing the decorator.
1. Naviage to UI Scripts module under the System UI application.
2. Create a new UI Script and name it spNotificationsDirective  
3. In the script field place the following  
4. Save the `spNotificationsDirective` UI Script

``` javascript
// Override spNotificationsDirective controller
angular.module('sn.$sp').config(function($provide) {
$provide.decorator('spNotificationsDirective', function($delegate, $controller) {
var directive = $delegate[0];
var controllerName = directive.controller;
directive.controller = function($scope) {
var controller = $controller(controllerName, {$scope: $scope});
// Override functions getMilliSeconds
controller.getMilliSeconds = function() {
//The first value is how many miliseconds trivial messages last, the second is how long Info and Error messages last.
var seconds = (areTrivial(controller.notifications)) ? 10 : 3000;
return seconds * 1000;
};
}
return controller;
};
return $delegate;
});
});
``` 


### Link the decorator to a theme
1. Create a new JS Include for this UI Script (Related list at the bottom of the UI Script Form).
2. Name the JS Include `spNotificationsDirectiveJSInclude` and leave the rest of the fields as is.
3. Navigate to the Themes module under the Service Portal application in the Filter Navigator.
4. Choose the theme you utilize for your Service Portal.  
(_If you don't know what theme you use for your portal you can also navigate to the portal module under Service Portal application and select your portal, then click on your theme._)  
5. In the JS Includes related list, click edit and search for `spNotificationsDirectiveJSInclude`, pull it into the right side of the slush bucket and save the record.

### How do I see this code on the client side?
1. Navigate to your Service Portal.
2. Open your DevTools (F12 for chrome).
3. Navigate to the sources tab(chrome).

If you expand the folder for your instance you should see a scripts folder and a `spNotificationsDirective.jsdbx`.  
If you expand the scripts folder you will see a `js_includes_sp.jsx?...`  
The `spNotificationsDirective.jsdbx` is your decorator file, you can click on it to see the code you saved in the UI Script.  
The `js_includes_sp.jsx?...` has the original code that your decorator is overriding. You can see this if you click on the `js_includes_sp.jsx?...` file and search for spNotifications, then scroll down to the `c.getMilliSeconds` function.  

Ideally any directives you see in the `js_includes_sp.jsx?...`, you can modify by using decorators. They are a great way to modify functionality, with the ability to quickly revert to out of box if/when ServiceNow adds functionality of fixes a bug.

You can read more on [angularJs Decorators](https://code.angularjs.org/1.6.10/docs/guide/decorators) via [angualrJS's documentation](https://code.angularjs.org/1.6.10/docs/api).



