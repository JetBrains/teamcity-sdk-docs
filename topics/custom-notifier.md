[//]: # (title: Custom Notifier)
[//]: # (auxiliary-id: Custom+Notifier.html)

Custom notifier must implement [`jetbrains.buildServer.notification.Notificator`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/notification/Notificator.html) interface and register implementation in the [`jetbrains.buildServer.notification.NotificatorRegistry`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/notification/NotificatorRegistry.html).

When a notifier is registered, it can provide information about additional properties that must be filled in by the user. To obtain values of these properties, use the following code:


```java
String value = user.getPropertyValue(new NotificatorPropertyKey(<notifier type>, <property name>));

```



Notifier can also provide custom UI for __Notifier rules__ and __My Settings&amp;Tools__ pages. See __PlaceId.NOTIFIER\_SETTINGS\_FRAGMENT__ and __PlaceId.MY\_SETTINGS\_NOTIFIER\_SECTION__.

Notifications are only delivered if there is at least one subscribed user for given event.

<tip>

Use source code of the existing plugins as a reference:
* [Create Teamcity Notifier](http://code.google.com/p/buildbunny/wiki/CreateTeamcityNotifier) \- instructions, source code at [GitHub](https://github.com/mendhak/buildbunny)
* [tcgrowl](https://github.com/ndrake/tcgrowl)
</tip>

  __See also:__

__Concepts__: [Notifier](https://www.jetbrains.com/help/teamcity/?notifier)

__User's Guide__: [Subscribing to Notifications](https://www.jetbrains.com/help/teamcity/?subscribing-to-notifications.md)

__Administrator's Guide__: [Customizing Notifications](https://www.jetbrains.com/help/teamcity/?customizing-notifications)
