[//]: # (title: Extending Highlighting for Web diff view)
[//]: # (auxiliary-id: Extending+Highlighting+for+Web+diff+view.html)

TeamCity uses [JHighlight](https://jhighlight.dev.java.net/) library to render the code on [Difference Viewer](https://www.jetbrains.com/help/teamcity/?difference-viewer) page. Essentially what JHighlight is doing is it takes plain source code, recognizes the language by extension, parses it, and in case of success renders the HTML output where the tokens are highlighted according to the specified settings. Unfortunately JHighlight supports relatively small subset of languages out\-of\-the\-box (major ones like Java, C\+\+, XML, and several more). Here we'd like to present you a HOWTO on adding the support for more languages.



<note>

Please note that in the further versions TeamCity may switch to another highlighting engine, so the changes you make will only work while JHighlight is used by TeamCity.
</note>



As an example we are implementing a highlighting for properties files, like this one:



```shell
# Comment on keys and values
key1=value1
foo = bar
x=y
a b c = foo bar baz
 
! another comment
! more complex cases:
a\=\fb : x\ty\n\x\uzzzz
 
 key = multiline value \
still value \
still value
the key

```




The implementation consists of the following steps:




### Step one: Writing a lexer using flex language



To understand this step, you might need to familiarize yourself with a [JFlex](http://jflex.de/) syntax.



There are several things you need to define in a flex file in order to generate a lexer. First of all, token types or, in our case, styles.



```java
public static final byte PLAIN_STYLE = 1;
public static final byte NAME_STYLE = 2;
public static final byte VALUE_STYLE = 3;
public static final byte COMMENT_STYLE = 4;

```




These constants will be mapped to the lexems in a source code and to the CSS classes, so at this moment you should decide which tokens are to be highlighted.
We will highlight names, values of properties, comments and plain text, which is just the '=' character.



Then you need to specify the states and actual parsing rules:



```java

WhiteSpace = [ \t\f]
 
%state IN_VALUE
 
%%
 
/* Rules for YYINITIAL state */
<YYINITIAL> {
  "\n"                           { return PLAIN_STYLE; }
 
  {WhiteSpace}                   { return PLAIN_STYLE; }
 
  [Extending Highlighting for Web diff view^=\n\t\f ]+                   { return NAME_STYLE; }
 
  "="                            { yybegin(IN_VALUE); return PLAIN_STYLE; }
 
  [#!] [Extending Highlighting for Web diff view^\n]* \n                 { return COMMENT_STYLE; }
}
 
/* Rules for IN_VALUE state */
<IN_VALUE> {
  "\\\n"                         { return VALUE_STYLE; }
 
  "\n"                           { yybegin(YYINITIAL); return PLAIN_STYLE; }
 
  [Extending Highlighting for Web diff view^\\\n]+                       { return VALUE_STYLE; }
}
 
/* error fallback */
.|\n                             { return PLAIN_STYLE; }

```




Our simple lexer has two states: initial (`YYINITIAL`, predefined) and `IN_VALUE`. In each of these states we try to handle the next character (or a group of characters) using regexp rules. 
The rules are applied from the top to the bottom, the first one that matches non\-empty string is used. Each rule is associated with the action to be performed on runtime. Here we have only simple actions that return the token constant and sometimes change the state.



To end the composition of a lexer add the common part to be inserted to the Java file. It's unlikely that you need to modify it.

Here's the full result code:



```java
package com.uwyn.jhighlight.highlighter;
 
import java.io.Reader;
import java.io.IOException;
 
%%
 
%class PropertiesHighlighter
%implements ExplicitStateHighlighter
 
%unicode
%pack
 
%buffer 128
 
%public
 
%int
 
%{
        /* styles */
 
        public static final byte PLAIN_STYLE = 1;
        public static final byte NAME_STYLE = 2;
        public static final byte VALUE_STYLE = 3;
        public static final byte COMMENT_STYLE = 4;
 
        /* Highlighter implementation */
 
        public byte getStartState() {
                return YYINITIAL+1;
        }
 
        public byte getCurrentState() {
                return (byte) (yystate()+1);
        }
 
        public void setState(byte newState) {
                yybegin(newState-1);
        }
 
        public byte getNextToken() throws IOException {
                return (byte) yylex();
        }
 
        public int getTokenLength() {
                return yylength();
        }
 
        public void setReader(Reader r) {
                this.zzReader = r;
        }
 
        public PropertiesHighlighter() {
        }
%}
 
WhiteSpace = [ \t\f]
 
%state IN_VALUE
 
%%
 
/* Rules for YYINITIAL state */
<YYINITIAL> {
  "\n"                           { yybegin(YYINITIAL); return PLAIN_STYLE; }
 
  {WhiteSpace}                   { return PLAIN_STYLE; }
 
  [Extending Highlighting for Web diff view^=\n\t\f ]+                   { return NAME_STYLE; }
 
  "="                            { yybegin(IN_VALUE);  return PLAIN_STYLE; }
 
  [#!] [Extending Highlighting for Web diff view^\n]* \n                 { return COMMENT_STYLE; }
}
 
/* Rules for IN_VALUE state */
<IN_VALUE> {
  "\\\n"                         { return VALUE_STYLE; }
 
  "\n"                           { yybegin(YYINITIAL); return PLAIN_STYLE; }
 
  [Extending Highlighting for Web diff view^\\\n]+                       { return VALUE_STYLE; }
}
 
/* error fallback */
.|\n                             { return PLAIN_STYLE; }

```




That's it: the lexer is ready. Download the latest JHighlighter sources from the [repository](https://jhighlight.dev.java.net/servlets/ProjectDocumentList) (version 1.0) and put this file to the `src/com/uwyn/jhighlight/highlighter` directory of JHighlight distribution.



### Step two: Generating a lexer on java



You can compile the code above using a [JFlex tool](http://jflex.de/), or amend the build.xml file adding the following task to the "flex" target:



```java
<jflex file="${src.dir}/com/uwyn/jhighlight/highlighter/PropertiesHighlighter.flex"
       destdir="${src.dir}"
       verbose="on"
       nobak="on"/>

```




After compilation we'll have the java class `PropertiesHighlighter` implementing the `ExplicitStateHighlighter` interface. If the previous steps are done right, you won't need to modify this file by hand.



### Step three: The renderer class.



The only JHighlight class left is the renderer corresponding to the generated lexer. This class should extend an `XhtmlRenderer` class and provide CSS classes correspondence along with default CSS map:



```java

package com.uwyn.jhighlight.renderer;
 
import com.uwyn.jhighlight.highlighter.ExplicitStateHighlighter;
import com.uwyn.jhighlight.highlighter.PropertiesHighlighter;
import com.uwyn.jhighlight.renderer.XhtmlRenderer;
import java.util.HashMap;
import java.util.Map;
 
public class PropertiesXhtmlRenderer extends XhtmlRenderer {
        // Contains the default CSS styles.
    public final static HashMap DEFAULT_CSS = new HashMap() {{
        put(".properties_plain",
            "color: rgb(0,0,0);");
 
        put(".properties_name",
            "color: rgb(0,0,128); " +
            "font-weight: bold;");
 
        put(".properties_value",
            "color: rgb(0,128,0); " +
            "font-weight: bold;");
 
        put(".properties_comment",
            "color: rgb(128,128,128); " +
            "background-color: rgb(247,247,247);");
    }};
 
    protected Map getDefaultCssStyles() {
        return DEFAULT_CSS;
    }
 
        // Maps the token type with the CSS class. E.g. each token of a 'PLAIN_STYLE' type will be rendered with 'properties_plain' style (see above).
    protected String getCssClass(int style) {
        switch (style) {
            case PropertiesHighlighter.PLAIN_STYLE:
                return "properties_plain";
            case PropertiesHighlighter.NAME_STYLE:
                return "properties_name";
            case PropertiesHighlighter.VALUE_STYLE:
                return "properties_value";
            case PropertiesHighlighter.COMMENT_STYLE:
                return "properties_comment";
        }
 
        return null;
    }
 
    protected ExplicitStateHighlighter getHighlighter() {
        return new PropertiesHighlighter();
    }
}
```




You can leave `DEFAULT_CSS` empty, but in this case the styles should always be present in jhighlight.properties file. But it is essential that the `PropertiesHighlighter` token constants are mapped to the CSS styles.



We also need to tell the factory class that a new renderer exists: for this `XhtmlRendererFactory` class should be updated. We don't provide the code here as it is very simple (in fact, two lines should be added).



### Step four: Running the JHighlight



JHighlight patch is ready, let's check it out in action. Put the properties file to the 'examples' directory and run the commands from JHighlight home directory:



```shell
ant
java -cp build/classes/ com.uwyn.jhighlight.JHighlight examples/
firefox examples/test.properties.html

```




Voilà! Our properties file is highlighted:



![Properties-screen-small.png](Properties-screen-small.png)



### Including JHighlight Changes into TeamCity Distribution



TeamCity uses only public JHighlight API, that's why if your patched JHighlight successfully generates the HTML, you have to do just few steps to integrate it to TeamCity:


	
* repack `jhighlight.jar` (call `ant jar`)
	
* replace `/WEB-INF/lib/jhighlight-njcms-patch.jar` with it
	
* restart TeamCity server




Good luck!
