---
title: "A brief look at the new Protobuf APIv2 for Golang"
date: 2020-04-15T20:51:29+02:00
draft: true
toc: false
featuredImg: ""
tags: 
  - protobuf
  - golang
---

There has been a [release](https://blog.golang.org/protobuf-apiv2) of a new protobuf API v2 for Golang which provides an entirely different API surface which in turn means that it is backward incompatible. But there are [good](https://docs.google.com/document/d/19kfhro7-CnBdFqFk7l4_HmwaH2JT_Rhw5-2FLWLEGGk/edit?usp=sharing) [reasons](https://docs.google.com/document/d/1epnKUCTeP_yQWmOJgOhELdmQTk8a3VPQGOV9d6RFbt0/edit?usp=sharing) for that.
Many want to access the generated proto `Message` programmatically and without knowing the concrete type at runtime. The release post describes one such use case, but there exists many in the wild.
For example, generating a table schema based on a provided proto `Message`, which coincidentally shares similarity with my use case.

So, I recently tried to see if we can migrate to use the new API. Since it is fairly new, there can be a bit of confusion around the whole go-protobuf ecosystem about the support of APIv2.
This post is a result of my efforts to investigate projects(that I regularly use) which need upgradation and a little fun experiment with `protoreflect` API for minor template-generation.

A very good overview of this new release is on the official [Protobuf Go reference FAQ page](https://developers.google.com/protocol-buffers/docs/reference/go/faq). I highly recommend going through them.

### Import Path
Most notably, the new import path for APIv2 is changed to `google.golang.org/protobuf` vs the old one `github.com/golang/protobuf/protoc-gen-go`.
As mentioned in the original release post, the old API v1 versions will start from the current [`v1.3.5`](https://github.com/golang/protobuf/releases/tag/v1.3.5) where as the new APIv2 is starting from [`v1.20.0`](https://github.com/protocolbuffers/protobuf-go/releases/tag/v1.20.0). 

Also `protoc-gen-go` [APIv1](github.com/golang/protobuf) is implemented in the terms of [APIv2](https://github.com/protocolbuffers/protobuf-go) at version [`1.4.0`](https://github.com/golang/protobuf/releases/tag/v1.4.0-rc.4) which means just upgrading to the new versions of old API will allow the programs to use the new API.

### Gotchas(that I _could_ found)
If your project uses a external protobuf configuration/management tool like [`prototool`](https://github.com/uber/prototool), it might take some time to get them to support the new APIv2. For example with prototool, the changed behaviour on new API w.r.t. [`go_package`](https://developers.google.com/protocol-buffers/docs/reference/go-generated#package) creates [problems](https://github.com/uber/prototool/issues/549).

If using gRPC with protobuf, only *old* protoc-gen plugin will support the generation of gRPC stubs for a while and [not the new one](https://github.com/protocolbuffers/protobuf-go/releases/tag/v1.20.0#v1.20-grpc-support).

Also, when using the new API, gRPC version must be upgraded to [1.27.0](https://github.com/grpc/grpc-go#compiling-error-undefined-grpcsupportpackageisversion)

If using extra whistles like `grpc-gateway`, those might be still on their way to [upgrade](https://github.com/grpc-ecosystem/grpc-gateway/pull/1165) and can prove to be non-trivial.

### Using `protoreflect`
Now, onto the fun part.
New APIv2 provides a revamped reflection API that allows access to the `Message` values as per the protocol buffer type system.
The release post does a good job on describing a use case where we need to iterate through all _populated_ fields of a proto message.

But, what if I wanted to iterate through all fields of a `Message` irrespective of whether they are populated or not and want to infer their proto type information.
For example, I want to map protobuf type system to another type system, of course for reasons ;)

Let's say I've a template that looks like this:
```
var caseTmpl = `

// declared field name as in proto
case %s:

//	(proto generated name)  =>  (a mapped type name)
	message.%s 				= anotherField.(%s)
`
```
Here `message` is a protobuf Message and `anotherField` is a field that should be assigned to the message but type signature of the field is different.

The first basic attempt:

```golang
func generate(message proto.Message) {

	// Returns all Fields/`FieldDescriptors` from the `MessageDescriptor`
	fds := message.ProtoReflect().Descriptor().Fields()
	for i := 0; i < fds.Len(); i++ {
		fd := fds.Get(i)
		//short name
		fieldName := string(fd.Name())
		// json name
		protoField := strings.Title(fd.JSONName())
		// the template is defined above and we pass the string arguments
		// fd.Kind() returns basic Kind/type for the field
		fmt.Printf(caseTmpl, fieldName, protoField, fd.Kind())
	}
}
```
We access the [`MessageDescriptor`](https://pkg.go.dev/google.golang.org/protobuf/reflect/protoreflect?tab=doc#MessageDescriptor) with the help of the new API and get the list of all fields, which has the type [`FieldDescriptors`](https://pkg.go.dev/google.golang.org/protobuf/reflect/protoreflect?tab=doc#FieldDescriptors).
`FieldDescriptors` is basically a container type for [`FieldDescriptor`](https://pkg.go.dev/google.golang.org/protobuf/reflect/protoreflect?tab=doc#FieldDescriptor).
`FieldDescriptor` has all the information regarding the name, type of the underlying field, which we can use to pass into the template.

I relied on the [`fd.Kind()`](https://pkg.go.dev/google.golang.org/protobuf/reflect/protoreflect?tab=doc#Kind)'s String [implmentation](https://github.com/protocolbuffers/protobuf-go/blob/v1.20.1/reflect/protoreflect/proto.go#L281) to get the type name. We can also use a map on the type name itsef if its signature in `anotherField` is called differently.

This works fine for a very simple proto definition(in terms of type richness).
```protobuf
message Customer {
  uint64 id = 1;
  string another_identifier = 2;
}
```
and we get output:
```golang
case id:
        message.Id = anotherField.(uint64)
case another_identifier:
        message.AnotherIdentifier = anotherField.(string)
```

However if we put a nested `Message` inside the proto definition, the `Kind` will print them as `anotherField.message` which is not what we want.
[Well known types](https://developers.google.com/protocol-buffers/docs/reference/google.protobuf) are also an embedded `Message` type.

Next version,
```golang
var MapWellKnownTypesToGoTypes = map[protoreflect.FullName]string{
	"google.protobuf.BoolValue":       "uint8",
	"google.protobuf.Timestamp":       "time.Time",
}

func generate(message proto.Message) {

	fds := message.ProtoReflect().Descriptor().Fields()
	for i := 0; i < fds.Len(); i++ {
		fd := fds.Get(i)
		fieldName := string(fd.Name())
		protoField := strings.Title(fd.JSONName())
		// this _can_ mean WKTs or nested messages
		if fd.Kind() == protoreflect.MessageKind {
			typeStr, wkt := MapWellKnownTypesToGoTypes[fd.Message().FullName()]
			if wkt {
				fmt.Printf(typeStr)
				// WKTs assignment requires a bit more sophistication but is possible. I'll leave that as an exercise.
			} else {
				m := message.ProtoReflect()
				// call it recursively on the nested Message
				// but we need to catch field marked as `repeated` 	
				if fd.Cardinality() != protoreflect.Repeated {
					generate(m.Get(fd).Message().Interface())
				} else {
					repeatedElem := m.Get(fd).List().NewElement()
					// this should be only be possible for nested message fields
					generate(repeatedElem.Message().Interface())
				}
			}
		} else if fd.Kind() == protoreflect.EnumKind {
			// enums can have type `int8`
			fmt.Printf(caseTmpl, fieldName, protoField, "int8")
		} else {
			fmt.Printf(caseTmpl, fieldName, protoField, fd.Kind())
		}
	}
}
```

This works for nested `Message` fields, but I also added some edge cases for `repeated` `Message` field and a separate case for `Enum`.

An example:
```protobuf
message Embed {
	string info = 1;
}

message Customer {
  uint64 id = 1;
  string another_identifier = 2;
  Embed embedded = 3;
}
```
and we get output :
```golang
case id:
        message.Id = anotherField.(uint64)
case another_identifier:
        message.AnotherIdentifier = anotherField.(string)
case embedded:
		message.Embedded = anotherField.(string)
```

However note that this doesn't pick the proper `fieldName` in the nested message case.
We can fix that by picking the `Parent` name from `FieldDescriptor`.
So the final version looks like below.

```golang
func generate(message proto.Message) {
	fds := message.ProtoReflect().Descriptor().Fields()
	for i := 0; i < fds.Len(); i++ {
		fd := fds.Get(i)
		fieldName := ""
		protoField := ""
		// in case of nested message
		parentName := string(fd.Parent().Name())
		if parentName != "" {
			fieldName = parentName + "_" + string(fd.Name())
			protoField = string(fd.Parent().Name()) + "." + strings.Title(fd.JSONName())
		} else {
			fieldName = string(fd.Name())
			protoField = strings.Title(fd.JSONName())
		}

		// this _can_ mean WKTs or nested messages
		if fd.Kind() == protoreflect.MessageKind {
			typeStr, wkt := MapWellKnownTypesToGoTypes[fd.Message().FullName()]
			if wkt {
				fmt.Printf(typeStr)
				// WKTs assignment requires a bit more sophistication but is possible. I'll leave that as an exercise.
			} else {
				m := message.ProtoReflect()
				// call it recursively on the nested field
				if fd.Cardinality() != protoreflect.Repeated {
					// ugh
					generate(m.Get(fd).Message().Interface())
				} else {
					repeatedElem := m.Get(fd).List().NewElement()
					// this should be only be possible for nested message fields
					generate(repeatedElem.Message().Interface())
				}
			}
		} else if fd.Kind() == protoreflect.EnumKind {
			// enums can have type `int8`
			fmt.Printf(caseTmpl, fieldName, protoField, "int8")
		} else {
			fmt.Printf(caseTmpl, fieldName, protoField, fd.Kind())
		}
	}
}
```

And that's it !

I hope this gives a bit more insight into the new `protoreflect` API. There is a lot of the API surface that I haven't touched but I'll leave that for another post.
