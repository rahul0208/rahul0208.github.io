---
layout: post
title:  Runtime XML binding using EclipseLink
#subtitle: Each post also has a subtitle
#image: /img/hello_world.jpeg
tags: [dynamic jaxb, eclipselink,]
---
Recently I was doing a POC for a dynamic JXB user-case. So our customer was consuming a JAXB based schema from one of their upstreams. The schema published  information fields of supported products. Since the upstream was also developing and adding more projects so they did not freeze their schema. It was undergoing periodic changes. The upstream still supported all older schema while moving to the new schema. The service which we were building was consuming product events and the registering them using the schema. So the problem was to support the changing schema.  
> The product will identify the schema which they work with. Then we need to register the product based on the supported schema .

If you have worked with `JAXB` then covering `schema.xsd` to Java Classes is quite simple. We just need to invoke the `xjc` or `wsimport` command. The classes will be generated for you consumption. We will have new XSDs published by the upstream. The ask was to consume these XSDs without building a new release. If we created the new release we could have generated the new schema, along-with the older schemas. But the the issue was how do you support so many changing schemas. Thus instead we took a different approach of runtime generation of schema.

### Runtime Schema Generation

Initially we were struggling to find a solution for runtime generation of JAXB classes. But the looking into EclipseLink, we found the required API to do so.

The first step is to parse the XSD at runtime to build a `DynamicContext`. This can be accomplished as the following :

```
DynamicJAXBContextFactory.createContextFromXSD(new FileInputStream(xsdFile),
                (x,y) -> {
                    String fileId = "Location" + File.separator + x;
                    InputSource is = new InputSource(ClassLoader.getSystemResourceAsStream(fileId));
                    is.setSystemId(fileId);
                    return is;
        }, null, null);
```
The lambda function helps in resolving the correct location of included schema files. This is helpful incase all related schema files are located at a convention-based location.

The `DynamicContext` can be used to generate `DynamicEntity<CLassName>`. The entity is a close representation of JavaBean. All properties are referred using getters and setters.

```
DynamicEntity genEntity = jaxbContext.newDynamicEntity("SchemaClassName");
genEntity.set("propertyName", "propertyValue");
```
It is important to mention the naming convention. Class names start with UpperCase and property names start with lowercase. `DynamicEntity`  provides a comprehensive API to read back values or to traverse to the parent object.

After building the required objects we ca generate the required XML using the `Marshaller` API.

```
Marshaller marshaller;
marshaller = jaxbContext.createMarshaller();
marshaller.setProperty(Marshaller.JAXB_FORMATTED_OUTPUT, true);

DynamicEntity rootElement = buildRootEntity(); // create the top Node

SchemaFactory sf = SchemaFactory.newInstance(XMLConstants.W3C_XML_SCHEMA_NS_URI);
Schema schema = sf.newSchema(xsdFile);
marshaller.setSchema(schema);

marshaller.setEventHandler((x) -> true); // A Validation EventHandler can be used do custom handling

marshaller.marshal(rootElement, new File("GenerateProduct.xml"));
```

Lastly we arrived to a dynamic generation of Products using the upstream Schema.

API Reference : https://www.eclipse.org/eclipselink/documentation/2.4/solutions/jpatoxml006.htm
