---
layout: default
title:  "Xtext - Parsing and Validating input string"
date:   2017-07-19
categories: main
---

In your Xtext project, it is common to come across situations where you have an input string and you need to - Parse, Validate and load it into a Resource set. If you have used EMF before, you can create a Resource instance using the ResourceSet and load the input string into the resource. It will also give you the errors that are typically produced as the resource is loaded. 

As the requirement is quit common, you migth end up duplicating the code in multiple locations in your codebase. Obviously this is bad and adds to the maintanence headache. A utility can come handy in such situations.

```
class MyDslParser {

	@Inject IResourceValidator validator

	def Model parse(String uri, String model, XtextResourceSet resourceSet, CancelIndicator cancelIndicator) {
		val resource = resourceSet.createResource(URI.createURI(uri))
		resource.load(new StringInputStream(model), null)
		
		// Check Syntactical errors. 
		// The errors list is typically produced as the resource is loaded
		if (!resource.errors.empty)
			throw new ParseException('Syntax error:\n' + resource.errors.map[message].join('\n'))
		
		// Invoke your DSL validator for user defined validations
		val issues = validator.validate(resource, CheckMode.ALL, cancelIndicator ?: CancelIndicator.NullImpl)
		if (issues.exists[severity == ERROR])
			throw new ParseException('Validation error:\n' + issues.filter[severity == ERROR].map[message].join('\n'))
		resource.contents.head() as Model
	}
}
```

If you are in a non-dsl project and you want the injection to work, you can get hold of the "IResourceServiceProvider" instance using the following technicques

```
val resourceServiceProvider = IResourceServiceProvider.Registry.INSTANCE.getResourceServiceProvider(URI.createURI("dummy.mydsl"))
resourceServiceProvider.get(Injector).injectMembers(this)
```

or you can call method "getResourceServiceProvider()" on an "XtextResource" instance as shown below.

```
val validator = resource.resourceServiceProvider.resourceValidator
val issues = validator.validate(resource, CheckMode.ALL, cancelIndicator ?: CancelIndicator.NullImpl)
```

PS - The code above is written using [Eclipse Xtend](https://www.eclipse.org/xtend/)
