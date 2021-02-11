# HAL Schema Form Profile

* Version: 0.0.1
* Status: Draft

## Description
This specification defines an extension to the [HAL json format](https://tools.ietf.org/html/draft-kelly-json-hal-08) to improve the evolvability of HAL based APIs.

## Compliance

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC2119](https://tools.ietf.org/html/rfc2119).

JSON format is described using [MSON](https://github.com/apiaryio/mson).

## Table of Contents

<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-refresh-toc -->
**Table of Contents**

- [Dwolla HAL Form Profile](#dwolla-hal-form-profile)
    - [Description](#description)
    - [Compliance](#compliance)
    - [Table of Contents](#table-of-contents)
    - [Introduction](#introduction)
    - [Media Type](#media-type)
    - [Document structure](#document-structure)
        - [HAL document form extension (object)](#hal-document-form-extension-object)
            - [Properties](#properties)
        - [Form (object)](#form-object)
            - [Properties](#properties-1)
        - [Field (object)](#field-object)
            - [Properties](#properties-2)
        - [Validation (object)](#validation-object)
            - [Properties](#properties-3)
        - [Value group (object)](#value-group-object)
            - [Properties](#properties-4)
        - [Value (object)](#value-object)
            - [Properties](#properties-5)
    - [Processing](#processing)
        - [Target URL resolution](#target-url-resolution)
        - [Body construction](#body-construction)
            - [Form transcoding](#form-transcoding)
                - [Value transcoding](#value-transcoding)
            - [JSON transcoding](#json-transcoding)
                - [Value transcoding](#value-transcoding-1)

<!-- markdown-toc end -->

## Introduction

Evolvability is important for any API with more than one client. The HAL specification requires API consumers have deep knowledge of the domain and implementation details of the API provider in situations that require user input or resulting unsafe requests. This deep knowledge inherently couples API consumers to the provider(s) reducing the evolvability of the overall system.

Forms are well tested way to reduce the coupling between API consumers and providers. This form system takes a highly pragmatic approach similar to the approach of HAL to APIs.

## Media Type

This extension is to be used as a profile link ([RFC6906](https://tools.ietf.org/html/rfc6906)) in conjunction with a HAL style media type.

Examples

* `application/hal+json; profile="https://github.com/jbadeau/hal-schema-forms"`

## Document structure

### HAL document form extension (object)

This spec extends the HAL format by adding the reserved property `_forms`. This property MAY appear on any HAL document (including embedded HAL documents).

Example HAL document with a form
```json
{
  "_links":{
    "self":{
      "href":"http://api.example.com/customers"
    }
  },
  "_forms":{
    "default":{
      "_links":{
        "target":{
          "href":"http://api.example.com/customers"
        }
      },
      "method":"POST",
      "contentType":"application/json",
      "schema":{
        "title":"A registration form",
        "description":"A simple form example.",
        "type":"object",
        "required":[
          "name",
          "email",
          "password"
        ],
        "properties":{
          "username":{
            "type":"string",
            "title":"Username",
          },
          "email":{
            "type":"string",
            "title":"Email"
          },
          "password":{
            "type":"string",
            "title":"Password",
            "minLength":10
          }
        }
      }
    }
  }
}
```


#### Properties

 - `_forms` (object, optional)

   - `default` ([form](#form), optional)

     The default form for the context. API providers SHOULD use this to indicate the best form for consumers with limited domain knowledge.

   - *form id(custom string)* ([form](#form-object), optional)

     A form related to the context. A verb first clauses expressing a command (such as, `create-customer`, `search-customers`) is RECOMMENDED for *form id*.

### Form (object)

A form object is a recipe for making complex API requests.

#### Properties

 - `_links`
   - `target` ([link object][], required)

     The resource which accepts submission of this form.  The target of a form MAY be a [templated link](https://tools.ietf.org/html/draft-kelly-json-hal-08#section-5.2

 - `method` (enum, required)

   The HTTP method to use when submitting this form. Consumers MUST ignore the case of values of this property.

   Consumers SHOULD support the following values.
   - `GET`
   - `DELETE`
   - `PATCH`
   - `POST`
   - `PUT`

    Producers MAY use values outside this set. Consumer SHOULD ignore forms whose `method` they don't understand. Forms with a `method` outside the list above will likely be ignored by most consumers.


 - `contentType` (string, optional)

   The media type of submissions required by the target resource. This should be the value of the `Content-Type` header. The `contentType` property is REQUIRED when `method` is  `PATCH`, `POST` or `PUT`.

   Consumers SHOULD accept `application/x-www-form-urlencoded`, `multipart/form-data`, `application/json` and any media type ending in `+json`. Producers MAY use media types outside this set. Consumers SHOULD ignore forms that use media types they don't understand. Forms with a content type outside the set above will likely be ignored by most consumers

   Clients MUST follow JSON encoding rules for media types ending in `+json`.

 - `schema` (object[[field](#field-object)], required)

A schema must be a valid [JSON Schema](https://json-schema.org/draft/2019-09/json-schema-core.html)  object.


## Processing

To submit a form API consumers will
 1. collect information for each field from the user
 1. [resolve the target URL](#target-url-resolution)
 1. [construct submission body](#body-construction)
 1. make submission request

The submission requests MUST
 - be to the resolved target URL
 - use the `method` specified by the form
 - have a body if the method allows it
 - have a `Content-Type` header field whose value is the `contentType` of the form if there is a body

### Target URL resolution

The target of a form MAY be a [templated link](https://tools.ietf.org/html/draft-kelly-json-hal-08#section-5.2). If the target link is not templated it's `href` MUST be used verbatim.

When target link is a template it MUST be expanded using the form fields in order to resolve it into a actual URL. Form fields must be converted in to a set of variable definitions whose values are the field values encoded using the [form value transcoding rules](#value-transcoding). The templated is then expanded using that set of variable definitions using the process define by [RFC 6570](https://tools.ietf.org/html/rfc6570).

For example, this form
```json
{
  "_links":{
    "target":{
      "href":"http://example.com/customers{?cust_id,name}",
      "templated":true
    }
  },
  "contentType":"application/json",
  "method":"GET",
  "schema":{
    "properties":{
      "cust_id":{
        "type":"string",
        "title":"Customer ID"
      },
      "name":{
        "type":"string",
        "title":"Name"
      }
    }
  }
}
```
might resolve to any of the following depending on the user input
 - `http://example.com/customers?cust_id=42`
 - `http://example.com/customers?name=frolic`
 - `http://example.com/customers?cust_id=42&name=frolic`

A field MAY be used in both URL expansion and body construction.

API producers MUST NOT include fields on forms whose method is GET or DELETE and whose target link is non-templated. Client SHOULD ignore fields on forms whose method is GET or DELETE and whose target link is non-templated.

## Error Handling

Error responses must be valid [# vnd.error](https://github.com/blongden/vnd.error) documents

```
{
  "message":"Registration failed",
  "logref":42,
  "traceref": "2q4736b867b8331be5de7463179c3de1",
  "_links":{
    "describes":{
      "href":"http://path.to/describes"
    },
    "help":{
      "href":"http://path.to/help"
    },
    "about":{
      "href":"http://path.to/registration"
    }
  },
  "_embedded":{
    "errors":[
      {
        "message":"Password must contain at least three characters",
        "path":"/password",
        "_links":{
          "about":{
            "href":"http://path.to/registration"
          }
        }
      }
    ]
  }
}
```


[link object]: https://tools.ietf.org/html/draft-kelly-json-hal-08#section-5
[link object]: https://json-schema.org/draft/2019-09/json-schema-core.html
[RFC 3966 telephone URIs]: https://tools.ietf.org/html/rfc3966
[RFC 6068 mailto URIs]: https://tools.ietf.org/html/rfc6068
[ISO 8601 encoded date time]: https://en.wikipedia.org/wiki/ISO_8601#Combined_date_and_time_representations
[ISO 8601 encoded time]: https://en.wikipedia.org/wiki/ISO_8601#Times
[ISO 8601 encoded calendar date]: https://en.wikipedia.org/wiki/ISO_8601#Calendar_dates
