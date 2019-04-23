[//]: # (title: Issue Tracker Integration Plugin)
[//]: # (auxiliary-id: Issue+Tracker+Integration+Plugin.html)

TeamCity [Integrating TeamCity with Issue Tracker](https://www.jetbrains.com/help/teamcity/?integrating-teamcity-with-issue-tracker) with several issue trackers and a custom plugin can provide support for other systems.



### Issue tracker integration



To create a TeamCity plugin for custom issue tracking system (ITS), you have to implement the following interfaces (all from `jetbrains.buildServer.issueTracker` package):


	
* [jetbrains.buildServer.issueTracker.SIssueProvider`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/notification/TemplateProcessor.html): represents a single provider
	
* [jetbrains.buildServer.issueTracker.IssueProviderFactory`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/issueTracker/IssueProviderFactory.html): API for instantiation of issue tracker providers




The main entity is a _provider_ (i.e. connection to the ITS), responsible for parsing, extracting and fetching issues from the ITS.



Here is a brief description of the strategy used in TeamCity in respect to ITS integration:
When the server is going to render the user comment (VCS commit, or build comment), it invokes all registered providers to parse the comment. This operation is performed by the `IssueProvider.getRelatedIssues()` method, which analyzes the comment and returns the list of the issue mentions ([`jetbrains.buildServer.issueTracker.IssueMention`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/issueTracker/IssueMention.html)).
`IssueMention` just holds the information that is enough to render a popup arrow near the issue id. When the user points the mouse cursor on the arrow, the server requests the full data for this issue calling `IssueProvider.findIssueById()` method, and then displays the data in a popup. The data can be taken from the provider's cache.



The provider has a number of parameters, configured from admin UI. These parameters are passed using the properties map (a map string \-&gt; string). Commonly used properties include provider name, credentials to communicate with ITS, or regular expression to parse issue ids. You don't have to worry about storing the properties in XML files, server does that.



Provider registration is done by the TeamCity administrator in the web UI, and the responsibility for it lies mostly on TeamCity server. The plugin must only provide a JSP used for creation/editing of the provider (see details below).



### Plugin development overview



A brief summary of steps to be done to create and add a plugin to TeamCity.


	
* Implement factory and provider interfaces ([`jetbrains.buildServer.issueTracker.SIssueProvider`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/notification/TemplateProcessor.html) and [`jetbrains.buildServer.issueTracker.IssueProviderFactory`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/issueTracker/IssueProviderFactory.html))
	
* Create a JSP page for admin UI
	
* [Install](https://www.jetbrains.com/help/teamcity/?installing-additional-plugins) the plugin (to `.BuildServer/plugins`)




### Reusing default implementation



Common code of Jira, Bugzilla and YouTrack plugins can be found in `Abstract*` classes in the same package:


	
* [`jetbrains.buildServer.issueTracker.AbstractIssueProviderFactory`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/issueTracker/AbstractIssueProviderFactory.html)
	
* [`jetbrains.buildServer.issueTracker.AbstractIssueProvider`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/issueTracker/AbstractIssueProvider.html)
	
* [`jetbrains.buildServer.issueTracker.AbstractIssueFetcher`](http://javadoc.jetbrains.net/teamcity/openapi/current/jetbrains/buildServer/issueTracker/AbstractIssueFetcher.html): a helper entity which encapsulates fetch\-related logic




`AbstractIssueProvider` implements a simple caching provider able to extract the issues from the string based on a regexp. In most cases you just need to derive from it and override few methods. A simple derived provider class can look like this:



```java
public class MyIssueProvider extends AbstractIssueProvider {
  // Let's name the provider simple: "myName". The plugin name should be the same.
  public MyIssueProvider(@NotNull IssueFetcher fetcher) {
    super("myName", fetcher);
  }

  // Means that issues are in format "PREFIX-123', like in Jira or YouTrack.
  // The prefix is configured via properties, regexp is invisible for users.
  protected boolean useIdPrefix() {
    return true;
  }
}

```




Providers like Bugzilla might need to override `extractId` method, because the mention of issue id (in comment) and the id itself can differ. For instance, suppose the issues are referenced by a hash with a number, e.g. #1234; the regexp is `"#(\d{4})"` (configurable); but the issues in ITS are represented as plain integers. Then the provider must extract the substrings matching `"#(\d{4})"` and return the first groups only. You should implement it in `extractId` method:



```java
@NotNull
protected String extractId(@NotNull String match) {
  Matcher matcher = myPattern.matcher(match);
  matcher.find();
  return matcher.group(1);
}

```




The factory code is very simple as well, for example:



```java
public class MyIssueProviderFactory extends AbstractIssueProviderFactory {
  public MyIssueProviderFactory(@NotNull IssueFetcher fetcher) {
    // Type name usually starts with uppercase character because it is displayed in UI, but not necessarily.
    super(fetcher, "MyName");
  }

  @NotNull
  public IssueProvider createProvider() {
    return new MyIssueProvider(myFetcher);
  }
}

```




IssueFetcher is usually the central class performing plugin\-specific logic. You have to implement `getIssue` method, which connects to the ITS remotely (via HTTP, XML\-RPC, etc), passes authentication, retrieves the issue data and returns it, or reports an error. Example:



```java
public IssueData getIssue(@NotNull String host, @NotNull String id,
                          @Nullable final Credentials credentials) throws Exception {
  final String url = getUrl(host, id);
  return getFromCacheOrFetch(url, new FetchFunction() {
    @NotNull
    public IssueData fetch() throws Exception {
      InputStream body = fetchHttpFile(url, credentials);
      IssueData result = null;
      if (body != null) {
        result = parseXml(body, url);
      }
      if (result == null) {
        throw new RuntimeException("Failed to fetch issue from \"" + url + "\"");
      }
      return result;
    }
  });
}

```




You need to implement how to compose the server URL and how do you parse the data out of XML (HTML). `AbstractIssueFetcher` will take care about caching, errors reporting and everything else.



### Plugin UI



The only mandatory JSP required by TeamCity is `editIssueProvider.jsp` (the full path must be `/plugins/myName/admin/editIssueProvider.jsp`, that is, the plugin should have the jsp available `/admin/editIssueProvider.jsp` of [its resources](plugins-packaging.md#PluginsPackaging-WebResourcesPackaging)). This JSP is requested when the user opens the dialog for editing (or creating) the issue provider. In most cases it just renders the provider properties, or returns the form for filling them.



You can see the example in `/plugins/youtrack/admin/editIssueProvider.jsp`.
