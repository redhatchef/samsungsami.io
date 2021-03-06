---
title: "The Manifest"
---

# The Manifest

SAMI is designed to communicate with any device regardless of how data is structured. Devices can upload data to SAMI in any format that works for them. Data is contained in a [message.](/sami/sami-documentation/sami-basics.html#messages) SAMI uses what we call the Manifest to interpret the content and store it properly. Once you define the Manifest for a given device type, SAMI can make its data available to other services and devices.

Manifests are written in [Groovy](http://groovy.codehaus.org/) to process messages as they come in one-by-one. Messages are structured to contain a single package of data (no batch uploads) that goes through the Manifest. This keeps data organization and processing straightforward.

When [**defining or updating a device type**](/sami/sami-documentation/developer-user-portals.html#creating-a-device-type), you must provide a Manifest that describes the data that will be stored in SAMI.
{:.info}

## Peek into the basics

Manifests are created using the [Manifest SDK.](#about-the-manifest-sdk) Your Manifest is a derived class of the `Manifest` base class. In that class, you must define a list of fields that composes your data and complete two methods: `normalize`{:.param} and `getFieldDescriptors`{:.param}.

- `normalize`{:.param} requires most of your attention. You must take the data as it is generated by a device or service, and split it into values for the fields you defined.

- `getFieldDescriptors`{:.param} only requires that you return a list of the fields in your data. This will be useful when a client wants to inspect the device and understand what data is available.

**Manifest template**

~~~java
import com.samsung.sami.manifest.Manifest
import com.samsung.sami.manifest.fields.*

public class TestGroovyManifest1 implements Manifest {

  // Custom FieldDesc
  // Create your field descriptors here

  // Implement the normalize method, parse your data as it comes in and spit out a list of fields with normalized values
  @Override
  List<Field> normalize(String input) {
  
    // return a list of Fields here    
    return []
  }

  // implement the getFieldDescriptors method, return the list of fields that your data will contain so that others know
  List<FieldDescriptor> getFieldDescriptors() {
    // return a list of FieldDescriptors here
    return []
  }
}
~~~

Below is an example of the simple Manifest with the basic information filled in.

~~~java
import groovy.json.JsonSlurper

import com.samsung.sami.manifest.Manifest
import com.samsung.sami.manifest.fields.*

public class TestGroovyManifest2 implements Manifest {

  // Custom FieldDesc
  // The device sends a single data point and it's a String that describes its status
  static final STATUS = new FieldDescriptor("status", String.class)

  @Override
  List<Field> normalize(String input) {
    // data is sent by the remote device as a JSON, so SAMI will need to use a library to understand it
    def slurper = new JsonSlurper()
    // If the device sent the data in a more complex format here developer can implement their String operations, loops, etc. In our case it's a single field, very easy.
    def json = slurper.parseText(input)

    // now we return the Field to the system
    return [
      new Field(STATUS, (String)json.status)
    ]
  }

  @Override
  List<FieldDescriptor> getFieldDescriptors() {
    // This is a single field, if it was more than one it would be an Array
    return [STATUS]
  }
}
~~~

The data that can be correctly processed by the above Manifest looks like:

~~~json
{
  "status": "on"
}
~~~

## Manifest certification

The process of certifying Manifests is in active development. Any changes to the process will be reflected here. 
{:.info}

Manifests are submitted when [creating new device types in the Developer Portal.](/sami/sami-documentation/developer-user-portals.html#creating-a-device-type) The SAMI team reviews your submitted Manifests. You should receive a response within 1 business day of submitting your Manifest. If the response is taking longer than expected, please use the [Feedback](#feedback) link to send us a message.

On approval or rejection, you will receive an email with any necessary feedback on resubmitting your Manifest. Manifests are tested not only for consistency, but also for compliance to SAMI standards of data categorization and performance. 

In general, your Manifest code should be clean and efficient. Because you may eventually want to [publish your device type](/sami/sami-documentation/developer-user-portals.html#creating-a-device-type) for other developers to use, please write your Manifest bearing in mind that it may become a tool for the public.

Some typical reasons a Manifest can be rejected:

- Not processing or returning data, due to being incomplete.
- Having a name that is too generic to be published, or uses the names "Samsung" or "SAMI."
- Including `printIn` or log.
- Attempting anything malicious.
- Attempting to use local resources (files) or remote resources (network calls).
- Consuming too much memory or generating memory leaks (unreleased references).

We also strongly encourage you to [test the Manifest before submitting it.](/sami/demos-tools/manifest-sdk.html) You may use the Manifest SDK as a command-line tool to validate your Manifest or use its testing APIs to write Manifest tests.

### What to do

The following is a list of best practices to keep in mind when writing your Manifest. Please check your Manifest against this list before you submit. This will eliminate time needed to make basic corrections and focus the SAMI team on helping you resolve more complex issues (if any) with your Manifest. 

#### printout

Avoid including standard output such as `printIn` or logs in your Manifest.

#### Field descriptors

`FieldDescriptor` names should follow the convention of [Java variable names.](https://docs.oracle.com/javase/tutorial/java/nutsandbolts/variables.html) Avoid using the `$` character in the name. For example, use `new FieldDescriptor("activeSteps")` rather than `new FieldDescriptor("active_steps")`.

Standard fields, or aliases of standard fields, should be reused whenever possible. For example, use `StandardFields.TEMPERATURE`, or define the following to use a type different from the standard double type:

~~~
public final static FieldDescriptor TEMP_INTEGER = TEMPERATURE.alias(Integer.class)
~~~

The `FieldDescriptor` must also be created so that it may be returned by the `getFieldDescriptors()` method rather than the `normalize()` method. This is because not all the data returned by `normalize()` can be described.

For example, you would implement the following:

~~~
public final static FieldDescriptor TEMP_INTEGER = TEMPERATURE.alias(Integer.class)
normalize(){
return [ new Field(TEMP_INTEGER, value)]
}
getFieldDescritors(){
return [TEMP_INTEGER]
}
~~~

Instead of:

~~~
normalize(){
return [ new Field(new FieldDescriptor("temp1", ...), value)]
}
getFieldDescritors(){
return [???]
}
~~~

#### Units

Units should be specified wherever appropriate. For example, specify units with distance or weight count, but not with step count, which is a quantity. 

Standard units, defined in the `JScience` package or in the `StandardUnits` class, should be reused whenever possible. 

#### Conversions 

The Manifest library can perform unit conversions for you. Rather than write a conversion from input unit to expected output unit, when creating an instance of a `Field` you can usually pass the unit that the value is expressed into. For example:

~~~
new Field(MIN_TEMPERATURE,NonSI.FAHRENHEIT, Double.parseDouble((String)json.minTemp)))
~~~

#### Utilities

To make your life easier, we recommend that you use the JSON utility classes, such as the [Groovy utilities](#groovy-utilities), when parsing JSON or other string-based input.

#### normalize() method

`nomalize()` must capture some data or fail, rather than send an empty result.

## About the Manifest SDK

Writing a Manifest for SAMI and testing it should be a straightforward process. You use the Manifest SDK to achieve that. The Manifest SDK offers two major functionalities: 

- A set of APIs, described in the [Manifest SDK API specification](/sami/demos-tools/manifest-sdk-javadoc/) and [Groovy utilities](#groovy-utilities), that you can use in your Manifest and Manifest tests.
- A [command-line tool](/sami/demos-tools/manifest-sdk.html#test-with-the-command-line-tool) to manually validate a Manifest against a sample data file. 

To learn how to write and perform Manifest tests, see [Validate your Manifest.](/sami/demos-tools/manifest-sdk.html)

## Groovy utilities

The Groovy utilities packaged with the Manifest SDK contain a few APIs meant to be used in a Manifest. [Advanced Manifest examples](/sami/demos-tools/manifest-advanced-example.html) illustrates that these APIs make writing your Manifest very easy.

Groovy utilities are not documented in the [**Manifest SDK API specification**.](/sami/demos-tools/manifest-sdk-javadoc/)
{:.info}

#### JsonUtil

This is a small set of utility methods helpful to manipulate JSON in a Manifest. JsonUtil currently exposes the function `addToList`, which is overloaded with different parameter combinations.

`addToList` extracts a field with the same name as the `FieldDescriptor` name from the JSON object and adds it to a list of fields. The call is safe if the property does not exist in the JSON object.

 |Parameter |Description
 |--------------- |------------
 |`fields`{:.param} | a groovy list of fields
 |`json`{:.param} | a JSON object returned by a JsonSlurper parse method
 |`fd`{:.param} | a SAMI field descriptor

`addToList` extracts a field with the same name as the `FieldDescriptor` name from the JSON object and adds it to a list of fields using a unit other than the one specified in the field descriptor to interpret the input value. The call is safe if the property does not exist in the JSON object.

 |Parameter |Description
 |--------------- |------------
 |`fields`{:.param} | a groovy list of fields
 |`json`{:.param} | a JSON object returned by a JsonSlurper parse method
 |`unitForInputValue`{:.param} | the unit to use to interpret for the input value
 |`fd`{:.param} | a SAMI field descriptor

`addToList` extracts a field with the name `propertyInJson` from the JSON object and adds it to a list of fields with the `FieldDescriptor` name. The call is safe if the property does not exist in the JSON object.

 |Parameter |Description
 |--------------- |------------
 |`fields`{:.param} | a groovy list of fields
 |`json`{:.param} | a JSON object returned by a JsonSlurper parse method
 |`propertyInJson`{:.param} | the name of the property in the JSON object
 |`fd`{:.param} | a SAMI field descriptor
 
`addToList` extracts a field with the name `propertyInJson` from the JSON object and adds it to a list of fields using a unit other than the one specified in `FieldDescriptor` to interpret the input value. The call is safe if the property does not exist in the JSON object.

 |Parameter |Description
 |--------------- |------------
 |`fields`{:.param} | a groovy list of fields
 |`json`{:.param} | a JSON object returned by a JsonSlurper parse method
 |`propertyInJson`{:.param} | the name of the property in the JSON object
 |`unitForInputValue`{:.param} | the unit to use to interpret for the input value
 |`fd`{:.param} | a SAMI field descriptor
 
#### StringFieldUtil

These utility methods are useful for writing Manifests when reading field values as strings or text (XML, CSV, etc).

These only work for the basic supported types (Boolean, Double, Float, Integer, Long, String) and **not** for collections (e.g., "string1, string2, string3" will not be interpreted as a collection but as a string).

`addToList` adds a field with the given `FieldDescriptor` and the text value using the value class defined in the descriptor and adds it to the specified list of fields. The call does not add the value if the value is null or empty.

  |Parameter |Description |
  |--------------- |------------ |
  |`fields`{:.param} | a groovy list of fields |
  |`value`{:.param} | a text value for the field to add |
  |`fd`{:.param} | a SAMI field descriptor |


`addToList` adds a field with the given `FieldDescriptor` and the text value using the value class defined in the descriptor and adds it to the specified list of fields using a unit other than the one specified in the field descriptor to interpret the input value. The call does not add the value if the value is null or empty.

  |Parameter |Description |
  |--------------- |------------ |
  |`fields`{:.param} | a groovy list of fields |
  |`value`{:.param} | a text value for the field to add |
  |`unitForInputValue`{:.param} | the unit to use to interpret for the input value |
  |`fd`{:.param} | a SAMI field descriptor |
