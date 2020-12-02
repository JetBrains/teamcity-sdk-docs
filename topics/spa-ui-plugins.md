[//]: # (title: SPA UI Plugins)
[//]: # (auxiliary-id: SPA+UI+Plugins.html)

This guide explains how to create a Single-page Application (SPA) React plugin based on the new [front-end extensions](front-end-extensions.md) paradigm.

__Source branch with an example project (FlowJS): [example/react-plugin](https://github.com/JetBrains/teamcity-sakura-ui-plugins/tree/example/react-plugin)__.

__Source branch with an example project (TypeScript): [example/react-plugin-typescript](https://github.com/JetBrains/teamcity-sakura-ui-plugins/tree/example/react-plugin-typescript)__.

Nowadays, modern JavaScript frameworks like Angular or libraries like React are dominating in the web development, especially if you write rich applications or Single Page Applications. They help developers to concentrate on a business logic – not on the code itself; they take care of the performance and, generally, make the development easier.

In TeamCity, we use React. Every component in the Sakura UI is a React component. Moreover, many components in the classic UI are the same React components but aged visually for UI/UX consistency.   

Similarly to a building process, we build the UI using small panels and bricks called components. Our components are based on the [Ring UI](https://jetbrains.github.io/ring-ui/master/index.html) library. We compose library components in order to build our UI.

Starting from TeamCity 2020.2 we share our "bricks". We expose some of our internal components and give an opportunity to use the Ring UI Library in plugins, as well, as other React UI Kits. Therefore, if you decide to develop your UI plugin with React, there is no needing to stylize your own buttons or dialogs – you can reuse the components we use ourselves.

## Composing SPA UI plugin

Before we start, please prepare your environment. Usually, modern Web development requires you to install [Node JS](https://nodejs.org/) and Node Package Manager (NPM). It might be not very convenient if you are a Java Developer and make the very first step with the modern Web. For this case, we prepared the `Docker-compose.yml` which will take care of compiling/transpiling and bundling in the JS.

To build your plugin within the Docker container, use

```shell script
docker-compose run build
mvn package
```

To run a plugin React application in the Webpack Development server mode, use

```shell script
mvn package
docker-compose run dev
```

If you prefer to run scripts locally using your environment, simply use the Intellij Idea Run Configurations or use CLI:

```shell script
cd frontend
npm install
npm run build
npm run start
```

The production building process for React components is different from other components. First, you have to launch webpack bundling - this step will generate a minified JavaScript bundle and put it to the `/demoPlugin-server/src/main/resources/buildServerResources` directory. Then you have to launch the Maven Package command:

```shell script
mvn package
```

Let's build the Demo plugin and see the result and apply it to the TeamCity. You will see the addition in the sidebar.

<img src="spa-plugins-1.png" thumbnail-same-file="true" thumbnail="true" alt="Simple sidebar plugin"/>

As in the previous sections, we start with `SakuraUIPluginController.java`.

```java
public class SakuraUIPluginController extends BaseController {
    private static final String PLUGIN_NAME = "SakuraUI-Plugin";

    private final PluginDescriptor myPluginDescriptor;

    public SakuraUIPluginController(
            @NotNull PluginDescriptor descriptor,
            @NotNull PagePlaces places,
            @NotNull WebControllerManager controllerManager
    ) {
        myPluginDescriptor = descriptor;

        String url = "/reactPlugin.html";
        final SimplePageExtension pageExtension = new SimplePageExtension(places);
        pageExtension.setPluginName(PLUGIN_NAME);
        pageExtension.setPlaceId(PlaceId.ALL_PAGES_FOOTER_PLUGIN_CONTAINER);
        pageExtension.setIncludeUrl(url);
        pageExtension.register();

        controllerManager.registerController(url, this);
    }

    @Nullable
    @Override
    protected ModelAndView doHandle(@NotNull HttpServletRequest request, @NotNull HttpServletResponse response) throws Exception {
        final ModelAndView mv = new ModelAndView(myPluginDescriptor.getPluginResourcesPath("react-plugin.jsp"));
        mv.getModel().put("PLUGIN_NAME", PLUGIN_NAME);
        return mv;
    }
}
```

### How Plugins are loaded?
<tip>

What is the difference between `PlaceID.ALL_PAGES_FOOTER_PLUGIN_CONTAINER` and `PlaceID.SAKURA_*`?

To understand it, let us explain the Plugin Wrapper workflow. In TeamCity 2020, at the time of DOMContentLoaded, we send a request to [`[TEAMCITY_BASE_URL]/app/placeId/__ALL__`](http://localhost:8111/bs/app/placeId/__ALL__). The server responds with the mapping `PlaceID` → `Array<Plugins>`. Each entry in this map contains metadata about the plugin: name, controller URL, list of attached CSS, and JS files.

Using this mapping, Plugin Wrappers understand, that there are some plugins attached to theirs `PlaceID`. Then, for each plugin, Plugin Wrapper starts requesting content from the Entrypoint. The content is the text/html - when it's loaded, Plugin Wrapper passes the given HTML to the JavaScript Plugin Constructor: 
```javascript
new Plugin(PlaceID, {
    content: withoutScripts(HTML),
    name: plugin.name,
    options: {debug: pluginDevelopmentMode != null},
    onCreate: () => {
        loadCssFiles(plugin.css)
        // some non-relevant to topic code
        js.map(script => scriptsContainer && scriptsContainer.appendChild(script))
        document.body && document.body.appendChild(scriptsContainer)
    },
})
```

During the constructing, the Plugin Constructor checks some minimal requirements (such as PlaceID is specified, and the plugin with the same ID doesn't exist) and, if all tests are passed, invokes the onCreate. As you see, the onCreate handler loads the CSS and JavaScript files. 

In other words, the plugin creation consists of multiple steps: request list of Plugins, request plugin HTML, parse HTML, create a plugin with parsed HTML, add subscription to the `ON_CREATE` event, where the styles and scripts are loaded asynchronously.

```
It is possible to explain it as a dialog between Frontend App (F), Plugin Wrapper (PW), and Server (S):

F: Server, give me the full list of plugins for the SAKURA `PlaceID`.
S: Here you are (gives mapping `PlaceID` → `Array<Plugins>`).
F: Thank you, Server.
...

F: PW, I have the plugin collection for you. Take it (gives the plugin).
PW: Hi, F! Thank you, I'll handle them from now on.
PW: Hey, S! I know that you have Plugin X for me in this URL. Please, give me the HTML content.
S: Here you are (gives HTML).
PW: Ok. Now I parse the HTML, extract JavaScript files, and create the plugin. Whenever the plugin is created, I'd like to load CSS and JS files.

As you can see, there is a few intermediate steps between the plugin HTML loading and actual JS/CSS applying to the page. You can avoid them by simply adding the plugin to `ALL_PAGES_FOOTER_PLUGIN_CONTAINER`. This is a `PlaceIDs` which works in both UIs. It is not processed by the Plugin Wrapper. 
```
</tip>


And the JSP File:
```jsp
<%@ page import="jetbrains.buildServer.util.StringUtil"
%><%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core"
%><%@ taglib prefix="intprop" uri="/WEB-INF/functions/intprop"
%><%@ include file="/include.jsp" %>

<c:set var="overrideBundleUrl" value="${intprop:getProperty('teamcity.plugins.SakuraUI-Plugin.bundleUrl', '')}" />

<c:choose>
  <c:when test="${!StringUtil.isEmpty(overrideBundleUrl)}">
    <script src="<c:out value="${overrideBundleUrl}" />"></script>
  </c:when>
  <c:otherwise>
    <bs:linkScript>${teamcityPluginResourcesPath}bundle.js</bs:linkScript>
  </c:otherwise>
</c:choose>
```
As you see, we use internal property `teamcity.plugins.SakuraUI-Plugin.bundleUrl` to understand, if we should load the JavaScript from the TeamCity itself, or somewhere else. It's required for incremental Webpack compilation and livetime updates we will touch later. This is the last Java-related code we will see in this guide.

From now, we are ready to the pure frontend. Here's a brief description of React basics.
   
Every operation in a browser which involves DOM manipulation (like inserting or updating the content, removing and finding nodes) should be considered as the most expensive operation. The React team found its way to dramatically increase the performance: instead of directly updating the DOM, React *creates a simplified copy of this DOM* called VDOM ([Virtual Dom](https://reactjs.org/docs/faq-internals.html)). Whenever you update the UI, React checks which part of VDOM should be changed (this is the render phase) and, if something changed in VDOM, React reflects this change to the real DOM (this is the commit phase). Checking the VDOM is faster than manipulating the 'real' DOM because it is a plain JavaScript object, which are way easier to manipulate. That is what made React so popular. 

In some cases, there could be multiple VDOMs on one page. It could happen if you use different React instances or if two parts have no common ancestor. Those VDOMs have no connection to each other. If you write your React plugin using the separate React instance, this plugin will have no access to some features like a common vDOM or common React contexts. The React application crashes when it faces two different React instances.

<img src="spa-plugins-2.png" thumbnail-same-file="true" thumbnail="true" alt="Application crash"/>

For plugin compatibility, we expose our internal React and ReactDOM instances via the Teamcity React API. The right usage of the TeamCity React instance is the key to writing a performant and safe plugin.

> Speaking of [upcoming React v.17](https://reactjs.org/blog/2020/08/10/react-v17-rc.html): there is one important change we will see soon: it will be not required anymore to use the only one React version. And yet, even the React Team explains, that the purpose of this update is to facilitate migration between different React versions, not to compose different React Versions permanently. 
{type="tip"}

As you can see, there is a new folder named `Frontend`. In this folder, we have a React application which uses [Flow JS](https://flow.org/en/docs/types/classes/). If you prefer static typed way of programming, you will definitely like Flow JS, but you are free to use any language (we use Flow JS, our Space team loves [KotlinJS](https://play.kotlinlang.org/hands-on/Building%20Web%20Applications%20with%20React%20and%20Kotlin%20JS/01_Introduction), most people all over the globe like [TypeScript](https://www.typescriptlang.org/)).

Unless you've decided to use Docker container, there are two mandatory actions you have to do:

1. Install the npm module `@jetbrains/teamcity-api` (see in `package.json` and in [NPM](https://www.npmjs.com/package/@jetbrains/teamcity-api)).

2. Configure the Webpack (`webpack.config.js`).

We tried to make the Webpack config as simple as we can do, so it simply extends the predefined config:

```js
const path = require('path')
const getWebpackConfig = require('@jetbrains/teamcity-api/getWebpackConfig')

module.exports = getWebpackConfig({
    srcPath: path.join(__dirname, './src'),
    outputPath: path.resolve(__dirname, '../demoPlugin-server/src/main/resources/buildServerResources'),
    entry: './src/index.js',
    useFlow: true,
})
```

As you see in `webpack.config.js`, there is an "entry" item which points to a certain file from where we start our React journey.

`src/index.js`:

```js

// @flow strict - let the Flow compiler and Intellij Idea know, that there is a Flow typed file
import {Plugin, React} from "@jetbrains/teamcity-api"
import App from './App/App'

new Plugin([Plugin.placeIds.SAKURA_SIDEBAR_TOP, Plugin.placeIds.BEFORE_CONTENT], {
    name: "Sakura UI Plugin",
    content: App,
});

```

`src/App/App.js`:

```js

import {H2, H3} from '@jetbrains/ring-ui/components/heading/heading' // 1

import {React} from "@jetbrains/teamcity-api"
import type {PluginContext} from "@jetbrains/teamcity-api"; // 2

import styles from './App.css' // 3

const defaultProfile = {
    firstName: "Elvis",
}

const ProfileInfo = ({onNameClick, firstName, lastName}) => 
  <H2 className={styles.name} onClick={onNameClick}>{`Hello, ${firstName} ${lastName ?? ''}`}</H2> // 4

function App({location}: {| location: PluginContext |}) { // 5
    const [expanded, setExpanded] = React.useState(false)
    const toggleExpanded = React.useCallback(() => setExpanded(state => !state), [])

    return (
        <div className={styles.wrapper}>
            <ProfileInfo onNameClick={toggleExpanded} name={defaultProfile.name} />
            {expanded && <div>
                {Object.entries(location).map(([key, value]) => value ? <p key={key}>{`${key}:${value}`}</p>: null)}
            </div>}
        </div>
    )
}

export default App
```

1. Import the Ring UI components. It could be buttons, dialogs, whatever you would find suitable your goal. For this tutorial, we selected the heading.
2. Import type definitions from `@jetbrains/teamcity-api` to add typings to the App component.
3. Import the  CSS file. Depending on your Webpack config, styles could be included in JavaScript or added to a separate file. Here styles are added in JS.
4. Example of using the Ring UI H2 component.
5. Our application receives only the `PluginContext` as a parameter. Whenever the location updates, Plugin Wrapper lets the plugin know about the new PluginUI context. React starts checking what DOM nodes should be changed and then applies changes to the DOM. You can click on the name in the sidebar to expand the container and make sure that the React plugin has subscribed to the plugin context updates.

We also should highlight, that there is an opportunity to reuse not a Public Library Components, but the internal TeamCity components. For now, we expose only the _All Builds_ component. In the next releases we will add Contexts and a few more components. To reuse internal TeamCity components, add the following code to `src/index.js`:

```js
import AllBuilds from "./AllBuilds/AllBuilds";
...
...
...
new Plugin(Plugin.placeIds.SAKURA_PROJECT_BEFORE_CONTENT, {
    name: "Sakura UI Plugin",
    content: AllBuilds,
});
```

Create `./AllBuilds/AllBuilds.js`:

```js

// @flow strict
import Loader from '@jetbrains/ring-ui/components/loader/loader'
import {H1} from '@jetbrains/ring-ui/components/heading/heading'
import {Content}  from '@jetbrains/ring-ui/components/island/island'
import {React} from '@jetbrains/teamcity-api'
import {AllBuilds as SakuraUIAllBuilds} from '@jetbrains/teamcity-api/components'

const AllBuilds = () => {
    const [count, setCount] = React.useState(3);

    React.useEffect(() => {
        if (count > 0) {
            setTimeout(() => setCount(count => count - 1), 1000)
        }
    }, [count])

    return <Content>
        {count === 0 ? <SakuraUIAllBuilds /> : <div><Loader /><H1>{`The React Plugin content will be shown in ${count} second`}</H1></div>}
    </Content>
};

export default React.memo(AllBuilds)
```

After the initialization, this component renders a Ring UI loader which will be shown for the next 3 second. After 3 seconds, it starts rendering the Sakura UI AllBuilds components which we imported from `@jetbrains/teamcity-api/components`.

This was an example of how to compose Ring UI components with TeamCity Sakura UI components and add your own code. At the moment, we have one exposed component (AllBuilds) but the list of exposed components will grow soon.
