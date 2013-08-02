# Tag Handlers

## Introduction

A primary solution provided by the Design Importer is that of transforming the input HTML into a top-level generated canvas component and a set of CQ components contained therein. 

The TagHandler is responsible for handling an HTML element, and all the HTML elements nested therein. The TagHandler receives SAX events corresponding to the HTML tags as and when they are encountered while parsing the HTML document. The output of the TagHandler could be markup, meta, script, includes, cq components or a mix of any of those. The below diagram illustrates how HTML snippets are tranformed TagHandlers into desired output content.

![image](/apathela/designimporter-sdk/raw/diagrams/wiki/images/taghandlerintro.png)

TagHandlers are POJOs instantiated everytime a tag needs to be handled. Each TagHandler has an associated TagHandlerFactory which is responsible for rolling out TagHandler instances.

TagHandlerFactories are implemented as OSGi services responsible for:

- Configuration of TagHandler
- Rolling out the corresponding TagHandler instance
- Injecting other OSGi service references as required by the TagHandler

## Lifecycle

The Design Importer framework controls when and how the callback methods of individual TagHandlers are invoked. The below steps descibe how Design Importer framework invokes various TagHandlers:

1. Design Importer receives SAX events for individual tags as the HTML document is getting parsed.
2. If there is no TagHandler active, the Design Importer resolves an appropriate TagHandler. As a part of initialization, the callback method TagHandler#beginHandling() is invoked by the Design Importer.
3. If a TagHandler is already active, the Design Importer invokes either of the startElement(), endElement() or characters() callback method
4. It's the responsibility of the active TagHandler to instantiate and initialize child TagHandlers, if required. For example, the ParsysComponentTagHandler on encountering nested component div tags instantiates and delegates to appropriate child TagHandlers.
5. The delegation continues recursively till a "leaf" TagHandler is reached. Thus, the handling happens in form of a delegation chain.
6. On receiving endElement(), the parent TagHandler is responsible for finishing up the child TagHandler, popping the child TagHandler out of the stack, and aggregating the content generated by by child TagHandler into itself.

The below diagram illustrates the delegation chain of the existing TagHandlers.

![image](/apathela/designimporter-sdk/raw/diagrams/wiki/images/taghandlerdelegation.png)

### Resolution

*How does the Design Importer framework decide which TagHandler should go about handling which html tag?*   
Each TagHandler declares the kind of tag it can handle via the tag.pattern OSGi property. This OSGi property stores the regular expression that needs be to matched against an HTML tag to determine if the TagHandler can handle the tag. Having the pattern stored as a configurable OSGi property has the clear advantage of ease with which the parsing logic could be configured.

Remember that the TagHandlers are plain java objects instantiated by their corresponding TagHandlerFactories. These TagHandlerFactories in turn are the OSGi services which need to define the configuration. Listed below are few examples of the tag.pattern property in out-of-the-box TagHandlerFactories:

1. CanvasComponentTagHandlerFactory

```java
/**
 * The TagHandlerFactory that rolls out {@link CanvasComponentTagHandler} instances
 */
@Service
@Component(metatype = true)
@Properties({
        @Property(name = Constants.SERVICE_RANKING, intValue = 5000, propertyPrivate = false),
        @Property(name = TagHandlerFactory.PN_TAGPATTERN, value = CanvasComponentTagHandlerFactory.TAG_PATTERN)
})
public class CanvasComponentTagHandlerFactory implements TagHandlerFactory {
}
```

2. TextComponentTagHandler

```java
/**
 * The TagHandlerFactory that rolls out {@link TextComponentTagHandler} instances
 */
@Service
@Component(metatype = true)
@Properties({
        @Property(name = Constants.SERVICE_RANKING, intValue = 5000, propertyPrivate = false),
        @Property(name = TagHandlerFactory.PN_TAGPATTERN, value = TextComponentTagHandlerFactory.TAG_PATTERN)
})
public class TextComponentTagHandlerFactory implements TagHandlerFactory {
}
```

3. ImgTagHandler    

```java
/**
 * The TagHandlerFactory that rolls out {@link ImgTagHandler} instances
 */
@Service
@Component(metatype = true)
@Properties({
        @Property(name = Constants.SERVICE_RANKING, intValue = 5000, propertyPrivate = false),
        @Property(name = TagHandlerFactory.PN_TAGPATTERN, value = ImgTagHandlerFactory.TAG_PATTERN),
        @Property(name = "service.factoryPid", value = "com.day.cq.wcm.designimporter.api.TagHandler")
})
public class ImgTagHandlerFactory implements TagHandlerFactory {
}
```

### Content Aggregation

Each TagHandler is responsible for controlling the lifecycle of its nested TagHandlers. Once a TagHandler starts handling an html element, it must also handle all the nested html elements. The nested elements could well map to other TagHandlers. It's the responsibility of the TagHandler to instantiate, destroy and control the nested TagHandlers. The Design Importer framework doesn't interfere here.

This may sound intimidating but this recurring logic is encapsulated by the AbstractTagHandler. The easiest way to reuse this functionality is thus, by extending from the AbstractTagHandler.

It's important to understand the type of content the TagHandlers emit. The below table descibes various content types.
<table>

<tr>
<th>Content Type</th>
<th>Description</th>
</tr>

<tr>
<th>HtmlContentType.META</th>
<th>Meta content typically defined within the HTML meta tags</th>
</tr>

<tr>
<th>HtmlContentType.MARKUP</th>
<th>The HTML markup. This is what majority of the TagHandlers emit</th>
</tr>

<tr>
<th>HtmlContentType.SCRIPT_INCLUDE</th>
<th>External javascript included via the HTML script tag</th>
</tr>

<tr>
<th>HtmlContentType.SCRIPT_INLINE</th>
<th>The javascript defined inline, within the HTML script tag</th>
</tr>

<tr>
<th>HtmlContentType.STYLESHEET_INCLUDE</th>
<th>External css included via the HTML link tag</th>
</tr>

<tr>
<th>HtmlContentType.STYLES_INLINE</th>
<th>inline styles within the HTML style tag</th>
</tr>

</table>

** Note: ** Your TagHandler must implement the HTMLContentProvider interface if it emits any html content. Typically, most TagHandlers emit some content.

ComponentTagHandlers, in addition to the html content, also emit in-core cq components that are later persisted to the jcr repository. The TagHandlers that implement the PageComponentProvider are automatically called back for the components they've generated at the end of their handling.

## Writing your own TagHandler

Writing TagHandler should be fairly easy once you understand the architecture as described above. Enlisted below are the cookbook steps you'll need to follow, in order to write your own TagHandlers:

- Define your TagHandler implementation class.
- Define your TagHandler factory
- Create and deploy bundle

Caveats:
- In case of conflicts, servce.ranking comes into picture
- Ensure that the regex is as intended
