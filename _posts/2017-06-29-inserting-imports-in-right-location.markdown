---
layout: default
title:  "XBase - Inserting Imports in the right location"
date:   2017-06-29
categories: main
---

Imagine you have a language like the one below - 1) Extends the Xbase language, 2) Has support for imports

```
grammar o.ss.xtext.entitydsl.EntityDsl with org.eclipse.xtext.xbase.Xbase
generate entityDsl "http://www.ss.o/xtext/entitydsl/EntityDsl"
Model:
	imports=XImportSection?
	'model' name=ID 'extends' parent=JvmTypeReference;	
```

When you set the parent, an import is automaticaly added to your file (BUG - [377860](https://bugs.eclipse.org/bugs/show_bug.cgi?id=377860)). You could also use Organize imports to do so. One of the problems you might face is that the import gets added in the wrong location (PS example below).

```
model foo extends 
import java.lang.reflect.AnnotatedArrayType

AnnotatedArrayType
```

The solution to the problem is to define a class that extends "org.eclipse.xtext.xbase.imports.DefaultImportsConfiguration" and override the method "getImportSectionOffset". You also need to bind the implementation class in your DSL Runtime Module.

```
class EntityDslImportConfiguration extends DefaultImportsConfiguration {
	
	override getImportSectionOffset(XtextResource resource) {
		val head = resource.contents.head
		val node = head?.findActualNodeFor
		if (node != null)
			node.offset
		else
			0
	}
	
}
```

This makes sure the imports get added in the right location.

References:                                                                                                                                                
[The XImportSection in Xbase 2.4](http://www.lorenzobettini.it/2013/01/the_ximportsection_in_xbase_2_4/),[BUG 377860](https://bugs.eclipse.org/bugs/show_bug.cgi?id=377860)                                                                                         
https://www.eclipse.org/forums/index.php/t/486071/
https://github.com/JanKoehnlein/XRobot/blob/master/org.xtext.xrobot.dsl/src/org/xtext/xrobot/dsl/imports/XRobotImportsConfiguration.xtend