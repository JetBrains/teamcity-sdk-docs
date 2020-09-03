[//]: # (title: SPA UI Plugins)
[//]: # (auxiliary-id: SPA+UI+Plugins.html)

This guide explains how to create a basic UI plugin based on the new [front-end extensions](front-end-extensions.md) paradigm.

Source branch with an example project: [example/react-plugin](https://github.com/JetBrains/teamcity-sakura-ui-plugins/tree/example/react-plugin).

Nowadays modern JavaScript frameworks like Angular or libraries like React are dominating in web development, especially if you write rich application or Single Page Application. They help developers to concentrate on logics, not on the Code itself; they take care of performance and, generally, make the development easier. 

In TeamCity we use React. Every component in the Sakura UI - is a React Component. Even more - many components in the Main UI are the same React Components, but aged visually for UI / UX consistency. React makes us reuse Components. Consider it as a building process - we build UI using small panels and bricks called Components. Our components are based on Ring UI (https://jetbrains.github.io/ring-ui/master/index.html) library. We compose library components to build every piece of the Sakura UI. 

And this looks so natural, that we decided to share our “bricks”.Starting the 2020.2 EAP 1 we expose some our internal components and give an opportunity to reuse the Ring UI in plugins. Therefore, if you decide to develop your UI Plugin with React, there is no needing to stylise you own buttons or dialogs, because you can just reuse the same components, we use ourselves. It requires some knowledge on how React works, but it’s easier, that you expect.

Before you start, you have to prepare your environment. Usually, modern Web development requires you to install Node JS (https://nodejs.org/) and Node Package Manager (NPM). We think, that it’s not convenient if you are a Java Developer and make the very first step with modern Web. For this case we prepared the Docker-compose.yml, which will take care of compiling / transpiling and bundling in the JS. 

To build your plugin, use 

docker-compose run build

To run a Plugin React Application in a Webpack Development Server mode, use

docker-compose run dev

If you prefer to run scripts locally using your environment, simply use the Intellij Idea Run Configurations or use CLI:

npm run build
npm run start

The building process for React components is different. Firstly, you have to launch webpack bundling - this step will generate minified JavaScript bundle and put it to the /demoPlugin-server/src/main/resources/buildServerResources. Then you have launch the Maven Package command:

mvn package

Let’s prepare plugin and see the result. If you open the TeamCity with a latest plugin, it appeared something new in the Sidebar. 
[Image: image.png]As usual, we start with SakuraUIPluginController.java. Here we use Basic Plugin with PluginUIContext.


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

There are some changes:

1. BUNDLE_DEV_URL is the URL where Webpack hosts your JavaScript React App. It will be used later for live updates. This allows you to recompile React App without reloading plugin to update JS sources;
2. Inject ContentSecurity Policy Configurator;
3. Change PlaceID to PlaceID.ALL_PAGES_FOOTER. By doing this, we make plugin code loading directly in HTML, skipping the Plugin Wrapper handling. See the note below.
4. Tell TeamCity core, that http://localhost:8080 (http://localhost:8080/) is a trusted domain, so it allows Cross Origin Requests from TeamCity to the webpack server;
5. Pass the BUNDLE_DEV_URL to JSP

! Do not forget to establish a dedicated build process for a production version. End users usually do not serve JS files separately. If you do not need BUNDLE_DEV_URL at all, you can remove all this code. Then it will be a single line Controller, as the Basic Plugin.


*Deep dive: *PlaceID.ALL_PAGES_FOOTER vs PlaceID.SAKURA_* - what’s the difference? 

To understand the difference, let us explain the Plugin Wrapper workflow. In TeamCity 2020 EAP1, at the beginning of a HTML Parsing, we send a request to [TEAMCITY_BASE_URL]/app/placeId/__ALL__ (http://localhost:8111/bs/app/placeId/__ALL__). The Server responds with the mapping PlaceID → Array<Plugins>. Each entry in this map contains meta data about the plugin: name, controller URL, list of attached CSS and JS files. That is how the Plugin Wrapper understands, that there are some plugins, attached to its Place ID. Then Plugin Wrapper starts requests for every plugin. When the HTML is loaded, Plugin Wrapper creates a plugin using the HTML: new Plugin(PlaceID, {content: HTML, ..., _onCreate: { loadScripts..., loadStyles...}_}); In other words, the Plugin creation consists of a few steps: request HTML, parse HTML, create Plugin with parsed HTML, add subscription to ON_CREATE event, where the Styles and Scripts are loaded asynchronously.

As long as all this is about the network discussion between Backend and Frontend, let us explain it as a dialog between Frontend App (F), Plugin Wrapper (PW) and Server (S)

F: Server, please, give me the full list of plugins for the new SAKURA PlaceID
S: Here you are (gives mapping PlaceID → Array<Plugins>)
F: Thank you, S! 
...
F: PW, I have plugins collection for you. Take it (gives Plugin)
PW: Hi, F! Thank you, I’ll handle them from now on.
PW: Hey, S! I know, that you have Plugin X for me on this URL. Please, give me the HTML content
S: Here you are (gives HTML)
PW: Ok. Now I parse the HTML, extract JavaScript files and Create Plugin. And whenever plugin is created, I’d like to load CSS and JS files.

As you can see, there are a few intermediate steps between the Plugin HTML loading and actual JS / CSS applying to the page. You can avoid them, simply adding the Plugin to the ALL_PAGES_FOOTER. This is an old Place ID. It is not processed by the Plugin Wrapper. But in this case you must manage Plugin in JavaScript file.


Content of React-plugin.jsp

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

Using the Choose directive, we tell the TeamCity core, whether the Plugin should load a JS file from the Webpack Server or use a Bundled one. And this is the last Java-related code we will see in this guide.

From now we are ready to the Pure Frontend. Why did we create a separate section for the React Plugins? What’s that important about React in spite of TeamCity UI? There are a lot of guides all other the Web, which explains step by step every aspect of React development. But if you would just create a Plugin using regular guides, you will not get all the benefits of TeamCity UI environment. 

Let us make a brief React basics introduction. Every operation in Browser, which involves DOM manipulation (like insert or update content, remove Nodes, find nodes) should be considered as the most expensive operation. React found its way to dramatically increase the performance: instead of directly update the DOM, React *creates a simplified copy of this DOM* called vDOM (Virtual Dom) (https://reactjs.org/docs/faq-internals.html). Whenever you update the UI, React checks, which certain part of VDOM should be changed (this is a Render Phase) and, if something changed in VDOM, React reflects this change to a real DOM (this is a Commit phase). Checking the VDOM is faster, than manipulating the DOM, because it’s a plain JavaScript object. So that is what made React so popular. 

In some cases, there could be few a VDOMs on one page. It could happen if you use different React instances or if two parts have no common ancestor. Those VDOMs have no connection to each other. If you write your React plugin using the separate React instance, this Plugin will have no access to some benefits, like a common vDOM or access to common React Contexts. Actually, the React Application crashes, when it faces two different React instances.
[Image: image.png]So, for Plugin compatibility we expose our internal React and ReactDOM instance via the TeamcityReactAPI. The right usage of the TeamCity React instance is a key to write a performant and safe plugin.

As you can see, there is a new folder called “Frontend”. In this folder we have a React Application, which uses FlowJS (https://flow.org/en/docs/types/classes/) inside. If you prefer static typed way of programming, you will definitely like FlowJS. For sure, you are free to use any language (we use Flow JS, our Space Team loves KotlinJS, most people all over the Globe like TypeScript). 

In this folder there are two important things, which are *obligatory*, if you would like to write a React Plugin. 1. You have to install npm module @teamcity/react-api (see in the package.json) and 2. configure the Webpack (webpack.config.js). 

In webpack.config.js we point to the next lines:

....
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
....
externals: {
    'react': 'TeamcityReactAPI.React', // 2
    'react-dom': 'TeamcityReactAPI.ReactDom',
},

1. In order to use Ring UI Library we let the Webpack know how to process Ring UI specific rules
2. We expose react and react-dom directly from TeamcityReactAPI.React, so whenever you import react in your JavaScript / TypeScript, it is imported from the TeamCity

As you see in webpack.config.js, there is an “entry” item, which points to a certain file, from where we start our React journey.

src/index.js

// @flow strict - let the Flow compiler and Intellij Idea know, that there is a Flow typed file
import {Plugin, React} from "@teamcity/react-api" // import npm-package to use the same React, Sakura UI uses
import App from './App/App' // import the main component

new Plugin(Plugin.placeIds.SAKURA_SIDEBAR_TOP, { // register a UI Plugin for the Sidebar
    name: "Sakura UI Plugin", // specify a plugin name
    content: App, // pass the component to a Plugin
});

src/App/App.js

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

1. Import the Ring UI components. It could be buttons, dialogs, whatever you would find suitable your goal. But for this case we selected Heading;
2. Import type definitions from the @teamcity/react-api to add typings to App component;
3. Import CSS file. Depends on your webpack config, styles could be included in JavaScript or added to a separate file. Here styles added in JS;
4. Example of using Ring UI H2 component';
5. Our Application receives only the PluginContext as a parameters. Whenever location updates, Plugin Wrapper lets the Plugin know about the new PluginUI Context. At this time React starts check what certain DOM nodes should be changed and then applies changes to DOM. You can click on the name in Sidebar to expand the container and make sure, that React Plugin subscribed to Plugin Context Updates


And the another plugin, which would work with TeamCity UI component called All Builds:

Add to the src/index.js the next code:


import AllBuilds from "./AllBuilds/AllBuilds";
...
...
...
new Plugin(Plugin.placeIds.SAKURA_PROJECT_BEFORE_CONTENT, {
    name: "Sakura UI Plugin",
    content: AllBuilds,
});

Create the ./AllBuilds/AllBuilds.js

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

After the initialization, this component renders a Ring UI Loader, which will be shown for the next 3 second. After 3 seconds, it starts render the Sakura UI AllBuilds components, which we imported from “@teamcity/react-api/components”.

This is a good example of how compose Ring UI components with TeamCity Sakura UI components and add some you own code. Right now we have the only one exposed component (AllBuilds); but soon the list of exposed component will grow.