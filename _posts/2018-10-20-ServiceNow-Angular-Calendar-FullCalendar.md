---
layout: post
title: A Service Portal Angular Calendar Widget Using Full Calendar
---
A widget to show a calendar similar to those on [http://angular-ui.github.io/ui-calendar/](http://angular-ui.github.io/ui-calendar/), with dynamic updating based on record updates.

## Problem
My current company had a use case where they wanted to see Upcomming Changes on a calendar view (They were unhappy with the OOB conflict calendar and wanted one that looks like the Cab WorkBench Calendar).  
(Someone asked in the ServiceNow Slack and I figuered it would be a great time to document how I did this.)  
**Update Set Below!!**


## Quick Solution OverView
I took the ui-calendar directive off of [http://angular-ui.github.io/ui-calendar/](http://angular-ui.github.io/ui-calendar/) and implemented it in ServiceNow. I then added functionality so that on clicking of an event it will display the record on a form.

## In-Depth Solution OverView

### Moving a Directive Into ServiceNow.
1. Search for Service Portal in the filter Navigator
2. Click on the Angular Providers Module.
3. Click New
4. Name the Angular Provider uiCalendar, set the type to Directive.
5. We then move the code around to work with how ServiceNow implements these Angualr Providers.

Original Code from [https://github.com/angular-ui/ui-calendar/blob/master/src/calendar.js](https://github.com/angular-ui/ui-calendar/blob/master/src/calendar.js)
``` javascript
/*
*  AngularJs Fullcalendar Wrapper for the JQuery FullCalendar
*  API @ http://arshaw.com/fullcalendar/
*
*  Angular Calendar Directive that takes in the [eventSources] nested array object as the ng-model and watches it deeply changes.
*       Can also take in multiple event urls as a source object(s) and feed the events per view.
*       The calendar will watch any eventSource array and update itself when a change is made.
*
*/

angular.module('ui.calendar', [])

    .constant('uiCalendarConfig', {
        calendars : {}
    })
    .controller('uiCalendarCtrl', ['$scope', '$locale',
        function ($scope, $locale) {

            var sources = $scope.eventSources;
            var extraEventSignature = $scope.calendarWatchEvent ? $scope.calendarWatchEvent : angular.noop;

            var wrapFunctionWithScopeApply = function (functionToWrap) {
                return function () {
                    // This may happen outside of angular context, so create one if outside.
                    if ($scope.$root.$$phase) {
                        return functionToWrap.apply(this, arguments);
                    }

                    var args = arguments;
                    var that = this;
                    return $scope.$root.$apply(
                        function () {
                            return functionToWrap.apply(that, args);
                        }
                    );
                };
            };

            var eventSerialId = 1;
            // @return {String} fingerprint of the event object and its properties
            this.eventFingerprint = function (e) {
                if (!e._id) {
                    e._id = eventSerialId++;
                }

                var extraSignature = extraEventSignature({
                    event : e
                }) || '';
                var start = moment.isMoment(e.start) ? e.start.unix() : (e.start ? moment(e.start).unix() : '');
                var end = moment.isMoment(e.end) ? e.end.unix() : (e.end ? moment(e.end).unix() : '');

                // This extracts all the information we need from the event. http://jsperf.com/angular-calendar-events-fingerprint/3
                return [e._id, e.id || '', e.title || '', e.url || '', start, end, e.allDay || '', e.className || '', extraSignature].join('');
            };

            var sourceSerialId = 1;
            var sourceEventsSerialId = 1;
            // @return {String} fingerprint of the source object and its events array
            this.sourceFingerprint = function (source) {
                var fp = '' + (source.__id || (source.__id = sourceSerialId++));
                var events = angular.isObject(source) && source.events;

                if (events) {
                    fp = fp + '-' + (events.__id || (events.__id = sourceEventsSerialId++));
                }
                return fp;
            };

            // @return {Array} all events from all sources
            this.allEvents = function () {
                return Array.prototype.concat.apply(
                    [],
                    (sources || []).reduce(
                        function (previous, source) {
                            if (angular.isArray(source)) {
                                previous.push(source);
                            } else if (angular.isObject(source) && angular.isArray(source.events)) {
                                var extEvent = Object.keys(source).filter(
                                    function (key) {
                                        return (key !== '_id' && key !== 'events');
                                    }
                                );

                                source.events.forEach(
                                    function (event) {
                                        angular.extend(event, extEvent);
                                    }
                                );

                                previous.push(source.events);
                            }
                            return previous;
                        },
                        []
                    )
                );
            };

            // Track changes in array of objects by assigning id tokens to each element and watching the scope for changes in the tokens
            // @param {Array|Function} arraySource array of objects to watch
            // @param tokenFn {Function} that returns the token for a given object
            // @return {Object}
            //  subscribe: function(scope, function(newTokens, oldTokens))
            //    called when source has changed. return false to prevent individual callbacks from firing
            //  onAdded/Removed/Changed:
            //    when set to a callback, called each item where a respective change is detected
            this.changeWatcher = function (arraySource, tokenFn) {
                var self;

                var getTokens = function () {
                    return ((angular.isFunction(arraySource) ? arraySource() : arraySource) || []).reduce(
                        function (rslt, el) {
                            var token = tokenFn(el);
                            map[token] = el;
                            rslt.push(token);
                            return rslt;
                        },
                        []
                    );
                };

                // @param {Array} a
                // @param {Array} b
                // @return {Array} elements in that are in a but not in b
                // @example
                //  subtractAsSets([6, 100, 4, 5], [4, 5, 7]) // [6, 100]
                var subtractAsSets = function (a, b) {
                    var obj = (b || []).reduce(
                        function (rslt, val) {
                            rslt[val] = true;
                            return rslt;
                        },
                        Object.create(null)
                    );
                    return (a || []).filter(
                        function (val) {
                            return !obj[val];
                        }
                    );
                };

                // Map objects to tokens and vice-versa
                var map = {};

                // Compare newTokens to oldTokens and call onAdded, onRemoved, and onChanged handlers for each affected event respectively.
                var applyChanges = function (newTokens, oldTokens) {
                    var i;
                    var token;
                    var replacedTokens = {};
                    var removedTokens = subtractAsSets(oldTokens, newTokens);
                    for (i = 0; i < removedTokens.length; i++) {
                        var removedToken = removedTokens[i];
                        var el = map[removedToken];
                        delete map[removedToken];
                        var newToken = tokenFn(el);
                        // if the element wasn't removed but simply got a new token, its old token will be different from the current one
                        if (newToken === removedToken) {
                            self.onRemoved(el);
                        } else {
                            replacedTokens[newToken] = removedToken;
                            self.onChanged(el);
                        }
                    }

                    var addedTokens = subtractAsSets(newTokens, oldTokens);
                    for (i = 0; i < addedTokens.length; i++) {
                        token = addedTokens[i];
                        if (!replacedTokens[token]) {
                            self.onAdded(map[token]);
                        }
                    }
                };

                self = {
                    subscribe : function (scope, onArrayChanged) {
                        scope.$watch(getTokens, function (newTokens, oldTokens) {
                            var notify = !(onArrayChanged && onArrayChanged(newTokens, oldTokens) === false);
                            if (notify) {
                                applyChanges(newTokens, oldTokens);
                            }
                        }, true);
                    },
                    onAdded : angular.noop,
                    onChanged : angular.noop,
                    onRemoved : angular.noop
                };
                return self;
            };

            this.getFullCalendarConfig = function (calendarSettings, uiCalendarConfig) {
                var config = {};

                angular.extend(config, uiCalendarConfig);
                angular.extend(config, calendarSettings);

                angular.forEach(config, function (value, key) {
                    if (typeof value === 'function') {
                        config[key] = wrapFunctionWithScopeApply(config[key]);
                    }
                });

                return config;
            };

            this.getLocaleConfig = function (fullCalendarConfig) {
                if (!fullCalendarConfig.lang && !fullCalendarConfig.locale || fullCalendarConfig.useNgLocale) {
                    // Configure to use locale names by default
                    var tValues = function (data) {
                        // convert {0: "Jan", 1: "Feb", ...} to ["Jan", "Feb", ...]
                        return (Object.keys(data) || []).reduce(
                            function (rslt, el) {
                                rslt.push(data[el]);
                                return rslt;
                            },
                            []
                        );
                    };

                    var dtf = $locale.DATETIME_FORMATS;
                    return {
                        monthNames : tValues(dtf.MONTH),
                        monthNamesShort : tValues(dtf.SHORTMONTH),
                        dayNames : tValues(dtf.DAY),
                        dayNamesShort : tValues(dtf.SHORTDAY)
                    };
                }

                return {};
            };
        }
    ])
    .directive('uiCalendar', ['uiCalendarConfig',
        function (uiCalendarConfig) {

            return {
                restrict : 'A',
                scope : {
                    eventSources : '=ngModel',
                    calendarWatchEvent : '&'
                },
                controller : 'uiCalendarCtrl',
                link : function (scope, elm, attrs, controller) {
                    var sources = scope.eventSources;
                    var sourcesChanged = false;
                    var calendar;
                    var eventSourcesWatcher = controller.changeWatcher(sources, controller.sourceFingerprint);
                    var eventsWatcher = controller.changeWatcher(controller.allEvents, controller.eventFingerprint);
                    var options = null;

                    function getOptions () {
                        var calendarSettings = attrs.uiCalendar ? scope.$parent.$eval(attrs.uiCalendar) : {};
                        var fullCalendarConfig = controller.getFullCalendarConfig(calendarSettings, uiCalendarConfig);
                        var localeFullCalendarConfig = controller.getLocaleConfig(fullCalendarConfig);
                        angular.extend(localeFullCalendarConfig, fullCalendarConfig);
                        options = {
                            eventSources : sources
                        };
                        angular.extend(options, localeFullCalendarConfig);
                        //remove calendars from options
                        options.calendars = null;

                        var options2 = {};
                        for (var o in options) {
                            if (o !== 'eventSources') {
                                options2[o] = options[o];
                            }
                        }
                        return JSON.stringify(options2);
                    }

                    scope.destroyCalendar = function () {
                        if (calendar && calendar.fullCalendar) {
                            calendar.fullCalendar('destroy');
                        }
                        if (attrs.calendar) {
                            calendar = uiCalendarConfig.calendars[attrs.calendar] = angular.element(elm).html('');
                        } else {
                            calendar = angular.element(elm).html('');
                        }
                    };

                    scope.initCalendar = function () {
                        if (!calendar) {
                            calendar = $(elm).html('');
                        }
                        calendar.fullCalendar(options);
                        if (attrs.calendar) {
                            uiCalendarConfig.calendars[attrs.calendar] = calendar;
                        }
                    };

                    scope.$on('$destroy', function () {
                        scope.destroyCalendar();
                    });

                    eventSourcesWatcher.onAdded = function (source) {
                        if (calendar && calendar.fullCalendar) {
                            calendar.fullCalendar(options);
                            if (attrs.calendar) {
                                uiCalendarConfig.calendars[attrs.calendar] = calendar;
                            }
                            calendar.fullCalendar('addEventSource', source);
                            sourcesChanged = true;
                        }
                    };

                    eventSourcesWatcher.onRemoved = function (source) {
                        if (calendar && calendar.fullCalendar) {
                            calendar.fullCalendar('removeEventSource', source);
                            sourcesChanged = true;
                        }
                    };

                    eventSourcesWatcher.onChanged = function () {
                        if (calendar && calendar.fullCalendar) {
                            calendar.fullCalendar('refetchEvents');
                            sourcesChanged = true;
                        }
                    };

                    eventsWatcher.onAdded = function (event) {
                        if (calendar && calendar.fullCalendar) {
                            calendar.fullCalendar('renderEvent', event, !!event.stick);
                        }
                    };

                    eventsWatcher.onRemoved = function (event) {
                        if (calendar && calendar.fullCalendar) {
                            calendar.fullCalendar('removeEvents', event._id);
                        }
                    };

                    eventsWatcher.onChanged = function (event) {
                        if (calendar && calendar.fullCalendar) {
                            var clientEvents = calendar.fullCalendar('clientEvents', event._id);
                            for (var i = 0; i < clientEvents.length; i++) {
                                var clientEvent = clientEvents[i];
                                clientEvent = angular.extend(clientEvent, event);
                                calendar.fullCalendar('updateEvent', clientEvent);
                            }
                        }
                    };

                    eventSourcesWatcher.subscribe(scope);
                    eventsWatcher.subscribe(scope, function () {
                        if (sourcesChanged === true) {
                            sourcesChanged = false;
                            // return false to prevent onAdded/Removed/Changed handlers from firing in this case
                            return false;
                        }
                    });

                    scope.$watch(getOptions, function (newValue, oldValue) {
                        if (newValue !== oldValue) {
                            scope.destroyCalendar();
                            scope.initCalendar();
                        } else if ((newValue && angular.isUndefined(calendar))) {
                            scope.initCalendar();
                        }
                    });
                }
            };
        }
    ]
);
```

Modificaiton of directive to get it working in ServiceNow (This goes in the Client Script) on the Angualr Provider

``` javascript
function () {
	var uiCalendarConfig = {};
	return {
		restrict : 'A',
		scope : {
			eventSources : '=ngModel',
			calendarWatchEvent : '&'
		},
		controller:function ($scope, $locale) {

			var sources = $scope.eventSources;
			var extraEventSignature = $scope.calendarWatchEvent ? $scope.calendarWatchEvent : angular.noop;

			var wrapFunctionWithScopeApply = function (functionToWrap) {
				return function () {
					// This may happen outside of angular context, so create one if outside.
					if ($scope.$root.$$phase) {
						return functionToWrap.apply(this, arguments);
					}

					var args = arguments;
					var that = this;
					return $scope.$root.$apply(
						function () {
							return functionToWrap.apply(that, args);
						}
					);
				};
			};

			var eventSerialId = 1;
			// @return {String} fingerprint of the event object and its properties
			this.eventFingerprint = function (e) {
				if (!e._id) {
					e._id = eventSerialId++;
				}

				var extraSignature = extraEventSignature({
					event : e
				}) || '';
				var start = moment.isMoment(e.start) ? e.start.unix() : (e.start ? moment(e.start).unix() : '');
				var end = moment.isMoment(e.end) ? e.end.unix() : (e.end ? moment(e.end).unix() : '');

				// This extracts all the information we need from the event. http://jsperf.com/angular-calendar-events-fingerprint/3
				return [e._id, e.id || '', e.title || '', e.url || '', start, end, e.allDay || '', e.className || '', extraSignature].join('');
			};

			var sourceSerialId = 1;
			var sourceEventsSerialId = 1;
			// @return {String} fingerprint of the source object and its events array
			this.sourceFingerprint = function (source) {
				var fp = '' + (source.__id || (source.__id = sourceSerialId++));
				var events = angular.isObject(source) && source.events;

				if (events) {
					fp = fp + '-' + (events.__id || (events.__id = sourceEventsSerialId++));
				}
				return fp;
			};

			// @return {Array} all events from all sources
			this.allEvents = function () {
				return Array.prototype.concat.apply(
					[],
					(sources || []).reduce(
						function (previous, source) {
							if (angular.isArray(source)) {
								previous.push(source);
							} else if (angular.isObject(source) && angular.isArray(source.events)) {
								var extEvent = Object.keys(source).filter(
									function (key) {
										return (key !== '_id' && key !== 'events');
									}
								);

								source.events.forEach(
									function (event) {
										angular.extend(event, extEvent);
									}
								);

								previous.push(source.events);
							}
							return previous;
						},
						[]
					)
				);
			};

			// Track changes in array of objects by assigning id tokens to each element and watching the scope for changes in the tokens
			// @param {Array|Function} arraySource array of objects to watch
			// @param tokenFn {Function} that returns the token for a given object
			// @return {Object}
			//  subscribe: function(scope, function(newTokens, oldTokens))
			//    called when source has changed. return false to prevent individual callbacks from firing
			//  onAdded/Removed/Changed:
			//    when set to a callback, called each item where a respective change is detected
			this.changeWatcher = function (arraySource, tokenFn) {
				var self;

				var getTokens = function () {
					return ((angular.isFunction(arraySource) ? arraySource() : arraySource) || []).reduce(
						function (rslt, el) {
							var token = tokenFn(el);
							map[token] = el;
							rslt.push(token);
							return rslt;
						},
						[]
					);
				};

				// @param {Array} a
				// @param {Array} b
				// @return {Array} elements in that are in a but not in b
				// @example
				//  subtractAsSets([6, 100, 4, 5], [4, 5, 7]) // [6, 100]
				var subtractAsSets = function (a, b) {
					var obj = (b || []).reduce(
						function (rslt, val) {
							rslt[val] = true;
							return rslt;
						},
						Object.create(null)
					);
					return (a || []).filter(
						function (val) {
							return !obj[val];
						}
					);
				};

				// Map objects to tokens and vice-versa
				var map = {};

				// Compare newTokens to oldTokens and call onAdded, onRemoved, and onChanged handlers for each affected event respectively.
				var applyChanges = function (newTokens, oldTokens) {
					var i;
					var token;
					var replacedTokens = {};
					var removedTokens = subtractAsSets(oldTokens, newTokens);
					for (i = 0; i < removedTokens.length; i++) {
						var removedToken = removedTokens[i];
						var el = map[removedToken];
						delete map[removedToken];
						var newToken = tokenFn(el);
						// if the element wasn't removed but simply got a new token, its old token will be different from the current one
						if (newToken === removedToken) {
							self.onRemoved(el);
						} else {
							replacedTokens[newToken] = removedToken;
							self.onChanged(el);
						}
					}

					var addedTokens = subtractAsSets(newTokens, oldTokens);
					for (i = 0; i < addedTokens.length; i++) {
						token = addedTokens[i];
						if (!replacedTokens[token]) {
							self.onAdded(map[token]);
						}
					}
				};

				self = {
					subscribe : function (scope, onArrayChanged) {
						scope.$watch(getTokens, function (newTokens, oldTokens) {
							var notify = !(onArrayChanged && onArrayChanged(newTokens, oldTokens) === false);
							if (notify) {
								applyChanges(newTokens, oldTokens);
							}
						}, true);
					},
					onAdded : angular.noop,
					onChanged : angular.noop,
					onRemoved : angular.noop
				};
				return self;
			};

			this.getFullCalendarConfig = function (calendarSettings, uiCalendarConfig) {
				var config = {};

				angular.extend(config, uiCalendarConfig);
				angular.extend(config, calendarSettings);

				angular.forEach(config, function (value, key) {
					if (typeof value === 'function') {
						config[key] = wrapFunctionWithScopeApply(config[key]);
					}
				});

				return config;
			};

			this.getLocaleConfig = function (fullCalendarConfig) {
				if (!fullCalendarConfig.lang && !fullCalendarConfig.locale || fullCalendarConfig.useNgLocale) {
					// Configure to use locale names by default
					var tValues = function (data) {
						// convert {0: "Jan", 1: "Feb", ...} to ["Jan", "Feb", ...]
						return (Object.keys(data) || []).reduce(
							function (rslt, el) {
								rslt.push(data[el]);
								return rslt;
							},
							[]
						);
					};

					var dtf = $locale.DATETIME_FORMATS;
					return {
						monthNames : tValues(dtf.MONTH),
						monthNamesShort : tValues(dtf.SHORTMONTH),
						dayNames : tValues(dtf.DAY),
						dayNamesShort : tValues(dtf.SHORTDAY)
					};
				}

				return {};
			};
		},
		link : function (scope, elm, attrs, controller) {
			var sources = scope.eventSources;
			var sourcesChanged = false;
			var calendar;
			var eventSourcesWatcher = controller.changeWatcher(sources, controller.sourceFingerprint);
			var eventsWatcher = controller.changeWatcher(controller.allEvents, controller.eventFingerprint);
			var options = null;

			function getOptions () {
				var calendarSettings = attrs.uiCalendar ? scope.$parent.$eval(attrs.uiCalendar) : {}
				var fullCalendarConfig = controller.getFullCalendarConfig(calendarSettings, uiCalendarConfig);
				var localeFullCalendarConfig = controller.getLocaleConfig(fullCalendarConfig);
				angular.extend(localeFullCalendarConfig, fullCalendarConfig);
				options = {
					eventSources : sources
				};
				angular.extend(options, localeFullCalendarConfig);
				//remove calendars from options
				options.calendars = null;

				var options2 = {};
				for (var o in options) {
					if (o !== 'eventSources') {
						options2[o] = options[o];
					}
				}
				return JSON.stringify(options2);
			}

			scope.destroyCalendar = function () {
				if (calendar && calendar.fullCalendar) {
					calendar.fullCalendar('destroy');
				}
				if (attrs.calendar) {
					calendar = uiCalendarConfig.calendars[attrs.calendar] = angular.element(elm).html('');
				} else {
					calendar = angular.element(elm).html('');
				}
			};

			scope.initCalendar = function () {
				if (!calendar) {
					calendar = $(elm).html('');
				}
				calendar.fullCalendar(options);
				if (attrs.calendar) {
					uiCalendarConfig.calendars[attrs.calendar] = calendar;
				}
			};

			scope.$on('$destroy', function () {
				scope.destroyCalendar();
			});

			eventSourcesWatcher.onAdded = function (source) {
				if (calendar && calendar.fullCalendar) {
					calendar.fullCalendar(options);
					if (attrs.calendar) {
						uiCalendarConfig.calendars[attrs.calendar] = calendar;
					}
					calendar.fullCalendar('addEventSource', source);
					sourcesChanged = true;
				}
			};

			eventSourcesWatcher.onRemoved = function (source) {
				if (calendar && calendar.fullCalendar) {
					calendar.fullCalendar('removeEventSource', source);
					sourcesChanged = true;
				}
			};

			eventSourcesWatcher.onChanged = function () {
				if (calendar && calendar.fullCalendar) {
					calendar.fullCalendar('refetchEvents');
					sourcesChanged = true;
				}
			};

			eventsWatcher.onAdded = function (event) {
				if (calendar && calendar.fullCalendar) {
					calendar.fullCalendar('renderEvent', event, !!event.stick);
				}
			};

			eventsWatcher.onRemoved = function (event) {
				if (calendar && calendar.fullCalendar) {
					calendar.fullCalendar('removeEvents', event._id);
				}
			};

			eventsWatcher.onChanged = function (event) {
				if (calendar && calendar.fullCalendar) {
					var clientEvents = calendar.fullCalendar('clientEvents', event._id);
					for (var i = 0; i < clientEvents.length; i++) {
						var clientEvent = clientEvents[i];
						clientEvent = angular.extend(clientEvent, event);
						calendar.fullCalendar('updateEvent', clientEvent);
					}
				}
			};

			eventSourcesWatcher.subscribe(scope);
			eventsWatcher.subscribe(scope, function () {
				if (sourcesChanged === true) {
					sourcesChanged = false;
					// return false to prevent onAdded/Removed/Changed handlers from firing in this case
					return false;
				}
			});

			scope.$watch(getOptions, function (newValue, oldValue) {
				if (newValue !== oldValue) {
					scope.destroyCalendar();
					scope.initCalendar();
				} else if ((newValue && angular.isUndefined(calendar))) {
					scope.initCalendar();
				}
			});
		}
	};
}
```
___As you can see, we moved all the code from the directive into a function with no parameters, and changed the controller property to the function in that was in the controller, instead of the name of the controller. We also had to uiCalendarConfig parameter(the constant) into the function and default it to an empty object.___

 6.Save the New Angular Provider.


### Creating the Widget
We will create the new widget, realte the angular provider and the necesarry dependencies and write the code.

1. Search for Service Portal in the filter Navigator
2. Click on the widgets module
3. Click New
4. Name the Widget "Angular Calendar"
5. Save the new Widget
6. Scroll down to the related lists, select the Angular Providers and select edit...
7. Move the uiCalendar Angular Provider(directive) from the left slush bucket to the right and save.
8. Select the Dependencies reated list and select new
9. Name the New Dependency "FullCalendar v3.8.0" and save.
10. Select the JS Includes Related list and select new
11. Set the display name to "FullCalendar 3.8 JS", source to URL and JS file url to "https://cdnjs.cloudflare.com/ajax/libs/fullcalendar/3.8.0/fullcalendar.min.js" and save.
12. Select the CSS Includes Related list and select new
13. Set the display name to "FullCalendar 3.8 CSS", source to URL and JS file url to "https://cdnjs.cloudflare.com/ajax/libs/fullcalendar/3.8.0/fullcalendar.min.css" and save.
14. Set the HTML, Client Script and Server Script as follows below:

HTML

``` html
<div>
  <div class="calendar" ng-model="eventSources" ui-calendar="calendar"></div>
</div>
```

Client Script

``` javascript
function($controller,$scope,spUtil,spModal){
	var c = this;
	/*
	* @description - Removes the event from the $scope.events
	* @author - Sean Boyer
	* @param {Object} obj - The object from the recordWatcher.
	*/
	$scope.deleteEvent = function(obj){
		var eventToBeDeleted = _.find($scope.events,function(listObj){
			return listObj.id == obj.data.sys_id;
		});
		var deleteAtIndex = $scope.events.indexOf(eventToBeDeleted);
		$scope.events.splice(deleteAtIndex,1);
	}
	/*
	* @description - Creates a new Event and adds it to $scope.events
	* @author - Sean Boyer
	* @param {Object} obj - The object from the recordWatcher.
	* @param {Object} filterObj - The filterObj that was used to initilzie the recordWatcher.
	*/
	$scope.createEvent = function(obj,filterObj){
		var event = {"title":"",
								 "id":"",
								 "start":"",
								 "end":"",
								 "table":filterObj.table,
								 "view":filterObj.view || "angularCalendar"}
		event.id = obj.data.sys_id;
		event.title = obj.data.record[filterObj.fields.display].display_value;
		event.start = obj.data.record[filterObj.fields.start].value;
		event.end = obj.data.record[filterObj.fields.end].value;
		$scope.events.push(event);
	}
	/*
	* @description - Removes the event from the $scope.events
	* @author - Sean Boyer
	* @param {Object} obj - The object from the recordWatcher.
	* @param {Object} filterObj - The filterObj that was used to initilzie the recordWatcher.
	*/
	$scope.updateEvent = function(obj,filterObj){
		var oldEvent = _.find($scope.events,function(listObj){
			return listObj.id == obj.data.sys_id;
		});
		if(obj.data.record[filterObj.fields.display].display_value){
			oldEvent.title = obj.data.record[filterObj.fields.display].display_value;
		}
		if(obj.data.record[filterObj.fields.start]){
			oldEvent.start = obj.data.record[filterObj.fields.start].value;
		}
		if(obj.data.record[filterObj.fields.end]){
			oldEvent.end = obj.data.record[filterObj.fields.end].value;	
		}
		$scope.events.push(oldEvent);

	}
	/*
	* @description - Adds a recordWatcher for each filter in $scope.filters
	* @author - Sean Boyer
	*/
	$scope.watchEventsFromFilters = function(){
		for(var i in $scope.filters){
			var filterObj = $scope.filters[i];
			spUtil.recordWatch($scope, filterObj.table, filterObj.encodedQuery, function(obj){

				switch(obj.data.operation){
					case "insert":
						$scope.createEvent(obj,filterObj);
						break;
					case "update":
						$scope.updateEvent(obj,filterObj);
						break;
					case "delete":
						$scope.deleteEvent(obj);
						break;
					default:
						break;
				}
			})
		}
	}
	/*
	* @description - Setup the Calendar so that on click it will open a modal with the form view of the record.
	* @author - Sean Boyer
	*/
	$scope.setupCalendarFunctionality = function(){
		$scope.calendar.eventClick = function(calEvent, jsEvent, view){
			var widgetInput = {
				'table':calEvent.table,
				'sys_id':calEvent.id,
				'view':calEvent.view
			};

			var widgetObj = {
				"title":calEvent.title,
				"widget":'widget-form',
				"widgetInput":widgetInput
			}
			spModal.open(widgetObj);
		}
	}

	$scope.formModel = {};
	$scope.calendar = c.data.calendar;
	$scope.setupCalendarFunctionality();
	//Intialize Values or default to empty.
	$scope.events = c.data.events || [];
	$scope.filters = c.data.filters || {};
	/*
	* @description - If filters are provided, set up record watcher to display the changes.
	* @author - Sean Boyer
	*/
	if($scope.filters.length > 0){
		$scope.watchEventsFromFilters();
	}

	$scope.eventSources = [$scope.events];


}
```

Server Script

``` javascript
(function() {
	var defaultCalendar = {};
	var defaultEvents = [];
	var defaultFilters = [];
	var parsedOptions = {};
	parseOptions();

	data.events = input.events || parsedOptions.events || defaultEvents;
	data.filters = input.filters || parsedOptions.filters || defaultFilters;
	/*
	* @description -If filters are provided, get the inital data.
	* @author - Sean Boyer
	*/
	if(data.filters.length > 0){
		generateEventsFromFilters();
	}
	data.calendar =  input.calendar || parsedOptions.calendar || defaultCalendar;
	/*
	* @description - If the option is populated, parse it into a JSON object to be used later.
	* @author - Sean Boyer
	*/
	function parseOptions(){
		if(options.events){
			parsedOptions.events = JSON.parse(options.events);
		}
		if(options.filters){
			parsedOptions.filters = JSON.parse(options.filters);
		}
		if(options.calendar){
			parsedOptions.calendar = JSON.parse(options.calendar);
		}
	}
	/*
	* @description - Creates events to pass to the client script based on the filters.
	* @author - Sean Boyer
	*/
	function generateEventsFromFilters(){
		for(var i in data.filters){
			var obj = data.filters[i];
			var gr = new GlideRecord(obj.table);
			gr.addEncodedQuery(obj.encodedQuery);
			gr.query();
			while(gr.next()){
				var event = {"title":"",
										 "id":"",
										 "start":"",
										 "end":"",
										 "table":"",
										 "view":""}
				//Set the Values.
				event.title = gr.getDisplayValue(obj.fields.display);
				event.id = gr.getUniqueValue();
				event.start = gr.getValue(obj.fields.start);
				event.end = gr.getValue(obj.fields.end);
				event.table = gr.getTableName();
				event.view = obj.view || "angularCalendar"
				//Add the event to the array of events.
				data.events.push(event);
			}
		}
	}
})();
```


### The Update Set!
[Download](https://raw.githubusercontent.com/SeanABoyer/seanaboyer.github.io/master/files/Angular-calendar-widget.xml)


Once you commit the update set you can navigate to [https://{instance-name}.service-now.com/sp?id=angular_calendar_example](https://{instance-name}.service-now.com/sp?id=angular_calendar_example) to see examples of using the widget with instance options and input options (Embedded widgets). These allow us to not have to manipulate the base widget.
