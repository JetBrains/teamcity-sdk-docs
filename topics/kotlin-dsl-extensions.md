[//]: # (title: Kotlin DSL Extensions)
[//]: # (auxiliary-id: Kotlin+DSL+Extensions.html)

TeamCity allows writing custom Kotlin DSL extensions for plugins. Extensions specify functionality blocks of a plugin (for example, a build runner or feature) in a DSL format. This provides the following benefits for [DSL-based projects](https://www.jetbrains.com/help/teamcity/kotlin-dsl.html):
* typed parameters for each specific functionality;
* autocompletion of parameters in IDE;
* controlled validation and proper compilation;
* DSL extensions are natively mapped onto the UI settings and displayed in the "View DSL" mode;
* changes made in the UI automatically apply to the DSL code.

With extensions, you can create a custom class library for your plugin, and TeamCity will handle the DSL generation and conversion when the plugin functionality is used in DSL-based projects.

To add an extension to a plugin:
1. Inside the plugin's [ZIP package](getting-started-with-plugin-development.md#Step+5.+Build+your+project+with+Maven), create the `kotlin-dsl` directory.
2. Inside the `kotlin-dsl` directory, create an `*.xml` file and describe a specific extension. Refer to the sections below for more information on expected syntax.

>Each extension must be described in its own `*.xml` file inside the `kotlin-dsl` directory.

>
{type="tip"}

This is a recommended approach which covers most use cases and can be properly processed by TeamCity.

If your plugin implements a major addition to the TeamCity functionality and cannot fully rely on common TeamCity objects, you have an option to write a completely custom extension. For this, add a `*.jar` file with your code to the same `kotlin-dsl` directory.   
Note that TeamCity will not be able to parse such a code using the extension syntax; it will generate a standard Kotlin DSL instead. Use this method only if you lack flexibility of the recommended approach.

## Declaring DSL Extension

To declare an extension in an XML file, use the following general syntax:

```XML

<dsl-extension kind="<kind_value>" type="<type_name>" generateDslJar="true">

    <class name="<class_name>">
        <description>
            A description of a class.
        </description>
    </class>

    <function name="<function_name>">
        <description>
            A description of a called function.
        </description>
    </function>

    <params>
        <param name="<parameter_name>">
         <description>
            A parameter's description.
        </description>
        </param>
    </params>

</dsl-extension>

```

where `<kind_value>` describes what kind of functionality is introduced by the extension. Supported values are:
* `vcs`
* `buildStep`
* `buildFeature`
* `failureCondition`
* `projectFeature`
* `trigger`
* `project`

### Extension Parameters

The `params` block can include as many parameters as needed:

```XML
...

<params>
    <param name="paramOne">
        <description>
            First parameter
        </description>
    </param>
    <param name="paramTwo">
        <description>
            Second parameter
        </description>
    </param>
</params>

...
```

To create a composite parameter that includes other nested parameters, use the `type="compound"` attribute:

```XML
...

<params>
    <param name="paramParent" type="compound">
        <description>
            Parent parameter
        </description>
            <param name="paramChildOne">
                <description>
                    First child parameter
                </description>
            </param>
            <param name="paramChildTwo">
                <description>
                    Second child parameter
                </description>
            </param>
    </param>
</params>

...
```

Supported parameters' attributes are:

<table>

<tr>
<td>

Attribute

</td>
<td>

Values

</td>
<td>

Description

</td>
</tr>

<tr>
<td>

`mandatory`

</td>
<td>

* `true`
* `false`

</td>

<td>

Sets a parameter as required or optional.

</td>

</tr>

<tr>

<td>

`dslName`

</td>

<td>

</td>

<td>

A parameter name to be used in DSL.

</td>

</tr>

<tr>

<td>

`type`

</td>

<td>

* `string`
* `boolean`
* `int`
* `compound`
* [custom](#Custom+Types)

</td>

<td>

For a boolean parameter, specify the `trueValue` and `falseValue` attributes.

</td>

<tr>

<td>

`ref`

</td>

<td>

* `VCS_ROOT`
* `BUILD_TYPE`

</td>

<td>

Refer the VCS root or build configuration object.

</td>

</tr>

</table>

Parameters can have nested options, and these options can have nested parameters. Options are represented in the UI as values of the parameter's drop-down menu, and their nested parameters are displayed only if the parent option is selected.

In general, options of a parameter are declared as follows:

```XML
...


<param name="paramOne">

    <option name="<optionOne>" value="<optionOne_value>">
        <description>
            A description of Option One.
        </description>
    </option>
    <option name="<optionTwo>" value="<optionTwo_value>">
        <description>
            A description of Option Two.
        </description>
    </option>

</param>

...
```

To add nested parameters to an option, use the following syntax:

```XML
...

<option name="<option_name>" value="<option_value>">
    <description>
        A description of an option.
    </description>
    <param name="paramOne">
        <description>
            A description of Parameter One.
        </description>
    </param>
    <param name="paramTwo">
        <description>
            A description of Parameter Two.
        </description>
    </param>   
</option>

...
```

### Custom Types

You can introduce custom enum types with your extension. Types are usually specified after the parameters' declaration.

The following example code will add multiple enum types to your extension:

```XML

<dsl-extension kind="<kind_value>" type="<type_name>" generateDslJar="true">
    
    ...
    
    <types>
        <enum name="<FileEncoding>">
            <option name="AUTODETECT" value="autodetect"/>
            <option name="ASCII" value="US-ASCII"/>
            <option name="UTF_8" value="UTF-8"/>
            <option name="UTF_16BE" value="UTF-16BE"/>
            <option name="UTF_16LE" value="UTF-16LE"/>
            <option name="CUSTOM" value="custom"/>
        </enum>
    </types>    

</dsl-extension>

```

## Extension Add-ons

If you want to create a DSL extension that adds functionality over an existing extension of another plugin, use the `dsl-add-on` functionality. Add-on files are stored in `.xml` files in the `kotlin-dsl` directory.

Add-on syntax:

```XML

<dsl-add-on kind="<kind_value>">
  <extends>
    <dsl type="<referenced_extension_type_name>"/>
  </extends>

  <params>
    ...
  </params>

</dsl-add-on>
```