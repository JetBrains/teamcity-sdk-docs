[//]: # (title: SPA UI Plugins)
[//]: # (auxiliary-id: SPA+UI+Plugins.html)

This guide explains how to create a Single-page Application (SPA) UI plugin based on the new [front-end extensions](front-end-extensions.md) paradigm.

__Source branch with an example project: [example/react-plugin](https://github.com/JetBrains/teamcity-sakura-ui-plugins/tree/example/react-plugin)__.

Nowadays, modern JavaScript frameworks like Angular or libraries like React are dominating in the web development, especially if you write rich applications or Single Page Applications. They help developers to concentrate on logic – not on the code itself; they take care of the performance and, generally, make the development easier.

In TeamCity, we use React. Every component in the Sakura UI is a React component. Moreover, many components in the classic UI are the same React components but aged visually for UI/UX consistency. React makes us reuse components.   
Similarly to a building process, we build the UI using small panels and bricks called components. Our components are based on the [Ring UI](https://jetbrains.github.io/ring-ui/master/index.html) library. We compose library components to build every piece of the Sakura UI.

Starting from TeamCity 2020.2 EAP1, we have decided to share our "bricks". We expose some of our internal components and give an opportunity to reuse the Ring UI in plugins. Therefore, if you decide to develop your UI plugin with React, there is no needing to stylize your own buttons or dialogs – you can reuse the components we use ourselves. Using them requires some knowledge on how React works, but it's easier than you might expect.

Before we start, please prepare your environment. Usually, modern Web development requires you to install [Node JS](https://nodejs.org/) and Node Package Manager (NPM). It might be not very convenient if you are a Java Developer and make the very first step with the modern Web. For this case, we prepared the `Docker-compose.yml` which will take care of compiling/transpiling and bundling in the JS.

To build your plugin, use

```shell script
docker-compose run build
```

To run a plugin React application in the Webpack Development server mode, use

```shell script
docker-compose run dev
```

If you prefer to run scripts locally using your environment, simply use the Intellij Idea Run Configurations or use CLI:

```shell script
npm run build
npm run start
```

The building process for React components is different from other components. First, you have to launch webpack bundling - this step will generate a minified JavaScript bundle and put it to the `/demoPlugin-server/src/main/resources/buildServerResources` directory. Then you have to launch the Maven Package command:

```shell script
mvn package
```

Let's prepare a plugin and see the result. If you open the TeamCity with the latest plugin enabled, you will see the addition in the sidebar.

<img src="spa-plugins-1.png"/>

As in the previous sections, we start with `SakuraUIPluginController.java`. Here we use the basic plugin with `PluginUIContext`.

```java

public class SakuraUIPluginController extends BaseController {
    private static final String PLUGIN_NAME = "SakuraUI-Plugin";
    private static final String BUNDLE_DEV_URL = "http://localhost:8080"; // 1

    private final PluginDescriptor myPluginDescriptor;

    public SakuraUIPluginController(
            @NotNull PluginDescriptor descriptor,
            @NotNull PagePlaces places,
            @NotNull WebControllerManager controllerManager,
            @NotNull final ContentSecurityPolicyConfig contentSecurityPolicyConfig // 2
    ) {
        myPluginDescriptor = descriptor;

        String url = "/reactPlugin.html";
        final SimplePageExtension pageExtension = new SimplePageExtension(places);
        pageExtension.setPluginName(PLUGIN_NAME);
        pageExtension.setPlaceId(PlaceId.ALL_PAGES_FOOTER); // 3
        pageExtension.setIncludeUrl(url);
        pageExtension.register();

        controllerManager.registerController(url, this);
        contentSecurityPolicyConfig.addDirectiveItems("script-src", BUNDLE_DEV_URL); // 4
    }

    @Nullable
    @Override
    protected ModelAndView doHandle(@NotNull HttpServletRequest request, @NotNull HttpServletResponse response) throws Exception {
        final ModelAndView mv = new ModelAndView(myPluginDescriptor.getPluginResourcesPath("react-plugin.jsp"));
        mv.getModel().put("BUNDLE_DEV_URL", BUNDLE_DEV_URL); // 5
        return mv;
    }
}
```

There are some changes:

1. `BUNDLE_DEV_URL` is the URL where Webpack hosts your JavaScript React App. It will be used later for live updates. This allows you to recompile the React App without reloading the plugin to update JS sources.
2. Inject the `ContentSecurity` policy configurator.
3. Change `PlaceID` to `PlaceID.ALL_PAGES_FOOTER`. By doing this, we make the plugin code loading directly in HTML, skipping the Plugin Wrapper handling. See the note below.
4. Tell the TeamCity core that `http://localhost:8080` is a trusted domain, so it allows Cross Origin Requests from TeamCity to the Webpack server.
5. Pass `BUNDLE_DEV_URL` to JSP.

>Remember to establish a dedicated build process for the production version. End users usually do not serve JS files separately. If you do not need to `BUNDLE_DEV_URL` at all, you can remove all this code. Then it will be a single line controller, as a basic plugin.
>
{type="note"}


<tip>

__What is the difference between `PlaceID.ALL_PAGES_FOOTER` and `PlaceID.SAKURA_*`?__

To understand it, let us explain the Plugin Wrapper workflow. In TeamCity 2020 EAP1, at the beginning of an HTML parsing, we send a request to [`[TEAMCITY_BASE_URL]/app/placeId/__ALL__`](http://localhost:8111/bs/app/placeId/__ALL__). The server responds with the mapping `PlaceID` → `Array<Plugins>`. Each entry in this map contains metadata about the plugin: name, controller URL, list of attached CSS, and JS files. That is how Plugin Wrapper understands that there are some plugins attached to its `PlaceID`. Then, Plugin Wrapper starts requesting every plugin. When the HTML is loaded, Plugin Wrapper creates a plugin using the HTML: `new Plugin(PlaceID, {content: HTML, ..., _onCreate: { loadScripts..., loadStyles...}_})`.   
In other words, the plugin creation consists of multiple steps: request HTML, parse HTML, create a plugin with parsed HTML, add subscription to the `ON_CREATE` event, where the styles and scripts are loaded asynchronously.

It is possible to explain it as a dialog between Frontend App (F), Plugin Wrapper (PW), and Server (S):

F: Server, give me the full list of plugins for the new SAKURA `PlaceID`.
S: Here you go (gives mapping `PlaceID` → `Array<Plugins>`).
F: Thank you, Server. 
...
F: PW, I have the plugin collection for you. Take it (gives the plugin).
PW: Hi, F! Thank you, I'll handle them from now on.
PW: Hey, S! I know that you have Plugin X for me in this URL. Please, give me the HTML content.
S: Here you go (gives HTML).
PW: Ok. Now I parse the HTML, extract JavaScript files, and create the plugin. Whenever the plugin is created, I'd like to load CSS and JS files.

As you can see, there is a few intermediate steps between the plugin HTML loading and actual JS/CSS applying to the page. You can avoid them by simply adding the plugin to `ALL_PAGES_FOOTER`. This is an old `PlaceID`. It is not processed by the Plugin Wrapper. But, in this case, you must manage the plugin in the JavaScript file.

</tip>


Content of `React-plugin.jsp`:

```jsp

<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core"
%><%@ taglib prefix="intprop" uri="/WEB-INF/functions/intprop"
%><%@ include file="/include.jsp" %>

<c:choose>
  <c:when test="${BUNDLE_DEV_URL ne null}">
    <bs:linkScript>${BUNDLE_DEV_URL}/bundle.js</bs:linkScript>
  </c:when>
  <c:otherwise>
    <bs:linkScript>${teamcityPluginResourcesPath}bundle.js</bs:linkScript>
  </c:otherwise>
</c:choose>
```

Using the `Choose` directive, we tell the TeamCity Core whether the plugin should load a JS file from the Webpack Server or use a bundled one. This is the last Java-related code we will see in this guide.

From now, we are ready to the pure frontend. But before that, here's a brief description of React basics.   
Every operation in a browser which involves DOM manipulation (like inserting or updating the content, removing and finding nodes) should be considered as the most expensive operation. React found its way to dramatically increase the performance: instead of directly updating the DOM, React *creates a simplified copy of this DOM* called VDOM ([Virtual Dom](https://reactjs.org/docs/faq-internals.html)). Whenever you update the UI, React checks which part of VDOM should be changed (this is the render phase) and, if something changed in VDOM, React reflects this change to the real DOM (this is the commit phase). Checking the VDOM is faster than manipulating the DOM because it is a plain JavaScript object. That is what made React so popular. 

In some cases, there could be multiple VDOMs on one page. It could happen if you use different React instances or if two parts have no common ancestor. Those VDOMs have no connection to each other. If you write your React plugin using the separate React instance, this plugin will have no access to some features like a common vDOM or common React contexts. The React application crashes when it faces two different React instances.

<img src="spa-plugins-1.png"/>

For plugin compatibility, we expose our internal React and ReactDOM instance via the Teamcity React API. The right usage of the TeamCity React instance is the key to writing a performant and safe plugin.

As you can see, there is a new folder named `Frontend`. In this folder, we have a React application which uses [FlowJS](https://flow.org/en/docs/types/classes/). If you prefer static typed way of programming, you will definitely like FlowJS, but you are free to use any language (we use Flow JS, our Space team loves KotlinJS, most people all over the globe like TypeScript).

There are two mandatory actions you have to do:

1. Install the npm module `@teamcity/react-api` (see in `package.json`).

2. Configure the Webpack (`webpack.config.js`).

In `webpack.config.js`, we point to the following lines:

```js

...
entry: './src/index.js',
module: {
    rules: [
        ...ringUiConfig.config.module.rules, // 1
        {
            test: /\.js$/,
            include: [srcPath],
            exclude: [/node_modules/],
            use: [babelLoader],
        },
    ]
},
...
externals: {
    'react': 'TeamcityReactAPI.React', // 2
    'react-dom': 'TeamcityReactAPI.ReactDom',
},
```

1. To use the Ring UI library, we let the Webpack know how to process the specific Ring UI rules.

2. We expose React and ReactDom directly from `TeamcityReactAPI.React`, so whenever you import React in your JavaScript/TypeScript, it is imported from TeamCity.

As you see in `webpack.config.js`, there is an "entry" item which points to a certain file from where we start our React journey.

`src/index.js`:

```js

// @flow strict - let the Flow compiler and Intellij Idea know, that there is a Flow typed file
import {Plugin, React} from "@teamcity/react-api" // import npm-package to use the same React, Sakura UI uses
import App from './App/App' // import the main component

new Plugin(Plugin.placeIds.SAKURA_SIDEBAR_TOP, { // register a UI Plugin for the Sidebar
    name: "Sakura UI Plugin", // specify a plugin name
    content: App, // pass the component to a Plugin
});
```

`src/App/App.js`:

```js

import {H2, H3} from '@jetbrains/ring-ui/components/heading/heading' // 1

import {React} from "@teamcity/react-api"
import type {PluginContext} from "@teamcity/react-api"; // 2

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
2. Import type definitions from `@teamcity/react-api` to add typings to the App component.
3. Importthe  CSS file. Depending on your Webpack config, styles could be included in JavaScript or added to a separate file. Here styles are added in JS.
4. Example of using the Ring UI H2 component.
5. Our application receives only `PluginContext` as a parameter. Whenever the location updates, Plugin Wrapper lets the plugin know about the new PluginUI context. React starts checking what DOM nodes should be changed and then applies changes to the DOM. You can click on the name in the sidebar to expand the container and make sure that the React plugin has subscribed to the plugin context updates.

Another plugin which would work with TeamCity UI component is called _All Builds.

Add the following code to `src/index.js`:

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
import {React} from '@teamcity/react-api'
import {AllBuilds as SakuraUIAllBuilds} from '@teamcity/react-api/components'

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

After the initialization, this component renders a Ring UI loader which will be shown for the next 3 second. After 3 seconds, it starts rendering the Sakura UI AllBuilds components which we imported from `@teamcity/react-api/components`.

This was an example of how to compose Ring UI components with TeamCity Sakura UI components and add your own code. At the moment, we have one exposed component (AllBuilds) but the list of exposed components will grow soon.