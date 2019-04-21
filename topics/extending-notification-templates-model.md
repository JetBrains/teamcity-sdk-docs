[//]: # (title: Extending Notification Templates Model)
[//]: # (auxiliary-id: Extending+Notification+Templates+Model.html)

You can extend data model passed into [Customizing Notifications](https://www.jetbrains.com/help/teamcity/?customizing-notifications) when evaluating.



In your plugin, implement `[jetbrains.buildServer.notification.TemplateProcessor](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/notification/TemplateProcessor.html)` interface. The following example can be found in our sample plugin:



```

public class SampleTemplateProcessor implements TemplateProcessor \{
  public SampleTemplateProcessor() \{
  \}

  @NotNull
  public Map<String, Object> fillModel(@NotNull NotificationContext context) \{
    Map<String, Object> model = new HashMap<String, Object>();
    model.put("users", context.getUsers());
    model.put("event", context.getEventType());
    return model;
  \}
\}

```


