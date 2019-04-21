[//]: # (title: Custom Authentication Module)
[//]: # (auxiliary-id: Custom+Authentication+Module.html)





There are two types of custom authentication modules, which can be provided by plugins: credentials authentication modules and HTTP authentication modules. The first ones are used to check the credentials user typed in login form on the login page. The second ones are used to authenticate a user by HTTP request without showing login page at all.





## Credentials Authentication Module



Credentials authentication modules API is based on Sun JAAS API. To provide your own credentials authentication module you should provide a login module class which must implement the interface __javax.security.auth.spi.LoginModule__ and register it in the __jetbrains.buildServer.serverSide.auth.LoginConfiguration__.



To make the authentication module active its type name can then be used during [Configuring Authentication Settings](https://www.jetbrains.com/help/teamcity/?configuring-authentication-settings).



For example:



```java
public class CustomLoginModule implements javax.security.auth.spi.LoginModule {
  private Subject mySubject;
  private CallbackHandler myCallbackHandler;
  private Callback[] myCallbacks;
  private NameCallback myNameCallback;
  private PasswordCallback myPasswordCallback;

  public void initialize(Subject subject, CallbackHandler callbackHandler, Map<String, ?> sharedState, Map<String, ?> options) {
    // We should remember callback handler and create our own callbacks.
    // TeamCity authorization scheme supports two callbacks only: NameCallback and PasswordCallback.
    // From these callbacks you will receive username and password entered on the login page.
    myCallbackHandler = callbackHandler;
    myNameCallback = new NameCallback("login:");
    myPasswordCallback = new PasswordCallback("password:", false);
    // remember references to newly created callbacks
    myCallbacks = new Callback[]{myNameCallback, myPasswordCallback};

    // Subject is a place where authorized entity credentials are stored.
    // When user is successfully authorized, the jetbrains.buildServer.serverSide.auth.ServerPrincipal
    // instance should be added to the subject. Based on this information the principal server will know a real name of
    // the authorized entity and realm where this entity was authorized.
    mySubject = subject;
  }

  public boolean login() throws LoginException {
    // invoke callback handler so that username and password were added
    // to the name and password callbacks
    try {
      myCallbackHandler.handle(myCallbacks);
    }
    catch (Throwable t) {
      throw new jetbrains.buildServer.serverSide.auth.TeamCityLoginException(t);
    }

    // retrieve login and password
    final String login = myNameCallback.getName();
    final String password = new String(myPasswordCallback.getPassword());

    // perform authentication
    if (checkPassword(login, password)) {
    // create ServerPrincipal and put it in the subject
      mySubject.getPrincipals().add(new ServerPrincipal(null, login));
      return true;
    }

    throw new jetbrains.buildServer.serverSide.auth.TeamCityFailedLoginException();
  }

  private boolean checkPassword(final String login, final String password) {
    return true;
  }

  public boolean commit() throws LoginException {
    // simply return true
    return true;
  }

  public boolean abort() throws LoginException {
    return true;
  }

  public boolean logout() throws LoginException {
    return true;
  }
}

```




Now we should register this module in the server. To do so, we create a login module descriptor:



```java
public class CustomLoginModuleDescriptor extends jetbrains.buildServer.serverSide.auth.LoginModuleDescriptorAdapter {
  // Spring framework will provide reference to LoginConfiguration when
  // the CustomLoginModuleDescriptor is instantiated
  public CustomLoginModuleDescriptor(LoginConfiguration loginConfiguration) {
    // register this descriptor in the login configuration
    loginConfiguration.registerLoginModule(this);
  }

  public String getName() {
    // return unique name of this module type. e.g. a derivative id will then be used in "auth-config.xml" file
    return "custom";
  }

  @Override
  public String getDisplayName() {
    // return presentable name of this plugin
    return "My custom authentication plugin";
  }

  @Override
  public String getDescription() {
    // return human-readable description of this module type
    return "Authentication via custom login module";
  }

  public Class<? extends LoginModule> getLoginModuleClass() {
    // return our custom login module class
    return CustomLoginModule.class;
  }

  @Override
  public Map<String, ?> getJAASOptions(final Map<String, String> properties) {
    // return options which can be passed to our custom login module
    // for example, we could store reference to SBuildServer instance here
    return null;
  }
}

```




Finally we should create `build-server-plugin-ourUserAuth.xml` and zip archive with plugin classes as it is described [Plugins Packaging](plugins-packaging.md) and write there `CustomLoginModuleDescriptor` bean:



```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE beans PUBLIC "-//SPRING//DTD BEAN//EN" "http://www.springframework.org/dtd/spring-beans.dtd">
<beans default-autowire="constructor">
  <bean id="customLoginModuleDescriptor" class="some.package.CustomLoginModuleDescriptor"/>
</beans>

```





Now you should be able to [Configuring Authentication Settings](https://www.jetbrains.com/help/teamcity/?configuring-authentication-settings) to your authentication module.



## HTTP Authentication Module



To provide your own HTTP authentication module you should provide a class which must implement the interface __jetbrains.buildServer.controllers.interceptors.auth.HttpAuthenticationScheme__ and register it in the __jetbrains.buildServer.serverSide.auth.LoginConfiguration__.



To make the authentication module active its type name can then be used during [Configuring Authentication Settings](https://www.jetbrains.com/help/teamcity/?configuring-authentication-settings).



For example:



```java
public class CustomHttpAuthenticationScheme extends jetbrains.buildServer.controllers.interceptors.auth.HttpAuthenticationSchemeAdapter {
  // Spring framework will provide reference to LoginConfiguration when
  // the CustomLoginModuleDescriptor is instantiated
  public CustomHttpAuthenticationScheme(final LoginConfiguration loginConfiguration) {
    // register this scheme in the login configuration
    loginConfiguration.registerAuthModuleType(this);
  }

  @Override
  protected String doGetName() {
    // return unique name of this module type. e.g. a derivative id will then be used in "auth-config.xml" file
    return "custom";
  }

  @Override
  public String getDisplayName() {
    // return presentable name of this plugin
    return "My custom HTTP authentication plugin";
  }

  @Override
  public String getDescription() {
    // return human-readable description of this module type
    return "Authentication via custom HTTP scheme";
  }

  // main method to process HTTP authentication
  @Override
  public HttpAuthenticationResult processAuthenticationRequest(final HttpServletRequest request,
                                                               final HttpServletResponse response,
                                                               final Map<String, String> properties) throws IOException {
    if (!isAttemptToAuthenticateViaThisHTTPScheme(request)) {
      return HttpAuthenticationResult.notApplicable();
    }

    // perform authentication
    final String username = authenticate(request);
    if (username == null) {
      return HttpAuthUtil.sendUnauthorized(request, response, "Failed to authenticate user", this, properties);
    }

    return HttpAuthenticationResult.authenticated(new ServerPrincipal(null, username), true);
  }
}

```





Finally we should create `build-server-plugin-ourUserAuth.xml` and zip archive with plugin classes as it is described [Plugins Packaging](plugins-packaging.md) and write there `CustomHttpAuthenticationScheme` bean:



```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE beans PUBLIC "-//SPRING//DTD BEAN//EN" "http://www.springframework.org/dtd/spring-beans.dtd">
<beans default-autowire="constructor">
  <bean id="customHttpAuthenticationScheme" class="some.package.CustomHttpAuthenticationScheme"/>
</beans>

```





Now you should be able to [Configuring Authentication Settings](https://www.jetbrains.com/help/teamcity/?configuring-authentication-settings) to your authentication module.
