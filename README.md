# Tag Handlers

> **Prerequisites**   
> This page is aimed at helping you write your own TagHandlers and does not covers the breadths of Design Importer. To gain a comprehensive insight into the Design Importer functionality, it's essential to go through the documentation links below:     
>    
> http://dev.day.com/docs/en/cq/current/wcm/campaigns/landingpages.html   
> http://dev.day.com/docs/en/cq/current/wcm/campaigns/landingpages/extending-and-configuring-the-design-importer.html

[Introduction](#introduction)    
[Lifecycle](#lifecycle)    
&nbsp;&nbsp;&nbsp;&nbsp;[Resolution](#resolution)    
&nbsp;&nbsp;&nbsp;&nbsp;[Content Aggregation](#content-aggregation)    
[Writing your own TagHandler](#writing-your-own-taghandler)    
[Inside the SDK](#inside-the-sdk)
[Useful Links](#useful-links)

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

**Question**: *How does the Design Importer framework decide which TagHandler should go about handling which html tag?*   
**Answer:** Each TagHandler declares the kind of tag it can handle via the tag.pattern OSGi property. This OSGi property stores the regular expression that needs be to matched against an HTML tag to determine if the TagHandler can handle the tag. Having the pattern stored as a configurable OSGi property has the clear advantage of ease with which the parsing logic could be configured.

Remember that the TagHandlers are plain java objects instantiated by their corresponding TagHandlerFactories. These TagHandlerFactories in turn are the OSGi services which need to define the configuration. Listed below are few examples of the tag.pattern property in out-of-the-box TagHandlerFactories:

**CanvasComponentTagHandlerFactory**

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

	static public final String TAG_PATTERN = "<div .*(?=id=\"(?i)cqcanvas\").*>";

}
```

**TextComponentTagHandler**    

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

	static public final String TAG_PATTERN = "<(p|span|div)\\s+.*data-cq-component=\"(?i)text\".*?>";

}
```    

**ImgTagHandler**    

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

	static public final String TAG_PATTERN = "<img(?!.* data-cq-component=\"(?i)image\").*>";

}
```

> **Note:** Since regular expressions could be overlapping, it's possible that multiple TagHandler qualify for a particular tag. In case of such conflicts, the TagHandler with the highest ranking value, as denoted by the OSGi property *SERVICE_RANKING*, is the one picked.

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

**Note:** Your TagHandler must implement the HTMLContentProvider interface if it emits any html content. Typically, most TagHandlers emit some content.

ComponentTagHandlers, in addition to the html content, also emit in-core cq components that are later persisted to the jcr repository. The TagHandlers that implement the PageComponentProvider are automatically called back for the components they've generated at the end of their handling.

## Writing your own TagHandler

Writing TagHandler should be fairly easy once you understand the architecture described above. Enlisted below are the cookbook steps you'll need to follow, in order to write your own TagHandlers:

- Define your TagHandler implementation class.
	- The TagHandler has to be designed keeping in mind the HTML fragment it shall handle.
	- Since the HTML fragment you're handling could contain nested tags, you'll need to handle them as well. The easiest way, is to simply extend from the [AbstractTagHandler](https://dev.day.com/docs/en/cq/current/javadoc/com/day/cq/wcm/designimporter/parser/taghandlers/AbstractTagHandler.html) class. There would be very few cases you wouldn't want to extend from AbstractTagHandler.
	- Implement/Override the beginHandling() method for any intialization activities
	- Implement/Override the endHandling() method for finalizing. This is where you should typically update the AbstractTagHandler buffers viz. *metadata*, *htmlBuffer*, *scriptBuffer*, *referencedScripts*, *pageComponents*
- Define your TagHandler factory
	- Make sure your TagHandlerFactory is a valid OSGi service that implements the design importer TagHandlerFactory service interface
	- Define the property OSGi TagHandlerFactory.PN_TAGPATTERN to be the regular expression that matches the HTML tag you intend to handle
	- Implement the create() method to instantiate and return your tag handler.
	- Use the @Reference annotation to have existing OSGi services injected. You may pass these service instances to your taghandlers when you instantiate them within the create() method.
	- You could also expose more OSGi configuration properties via the @Property or @Properties annotations and use them to configure the behaviour of your TagHandlers.
- Create and deploy bundle

Caveats:

- In case of conflicts, servce.ranking comes into picture
- Ensure that the tag pattern regex is as intended

## Inside the SDK

The SDK contains a starter maven project built by following steps mentioned at http://dev.day.com/docs/en/cq/aem-how-tos/development/how-to-build-aem-projects-using-apache-maven.html

> The maven project created by following the steps as described in the above link also contains a folder named 'content'. But since the output of this sdk is just a bundle that contains the TagHandler service implementation, the content folder has been excluded from this sdk.

## Useful Links

- http://dev.day.com/docs/en/cq/current/wcm/campaigns/landingpages.html   
- http://dev.day.com/docs/en/cq/current/wcm/campaigns/landingpages/extending-and-configuring-the-design-importer.html
- https://dev.day.com/docs/en/cq/current/javadoc/com/day/cq/wcm/designimporter/parser/taghandlers
- http://felix.apache.org/documentation/subprojects/apache-felix-maven-scr-plugin/scr-annotations.html