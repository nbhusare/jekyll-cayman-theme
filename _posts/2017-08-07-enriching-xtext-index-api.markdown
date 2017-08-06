---
layout: default
title:  "Enriching Xtext Index API"
date:   2017-08-07
categories: main
---

# Enriching Xtext Index API

In Eclipse, it is quiet common to use the [Open Type](https://help.eclipse.org/mars/index.jsp?topic=%2Forg.eclipse.jdt.doc.user%2Freference%2Fref-dialog-open-type.htm) (for Java types) and [Open Resource](https://help.eclipse.org/neon/index.jsp?topic=%2Forg.eclipse.platform.doc.user%2Freference%2Fref-dialog-open-resource.htm) (for files) selection dialogs. The types are indexed by the JDT indexing, and the files by the Eclipse platform indexing mechanism. [Xtext](https://eclipse.org/Xtext/) supports indexing of DSL elements and provides the Open Model Element (Ctrl + Shift + F3) dialog to quickly open the indexed elements. 

Xtext Index (implemented by [IResourceDescriptions](http://download.eclipse.org/modeling/tmf/xtext/javadoc/2.3/org/eclipse/xtext/resource/IResourceDescriptions.html)) is a data structure that holds information/metadata about all the [EMF Resources](http://download.eclipse.org/modeling/emf/emf/javadoc/2.9.0/org/eclipse/emf/ecore/resource/Resource.html) and the contained [EObjects](http://download.eclipse.org/modeling/emf/emf/javadoc/2.9.0/org/eclipse/emf/ecore/EObject.html). The metadata of a [Resource](http://download.eclipse.org/modeling/emf/emf/javadoc/2.9.0/org/eclipse/emf/ecore/resource/Resource.html) is stored using [IResourceDescription](http://download.eclipse.org/modeling/tmf/xtext/javadoc/2.3/org/eclipse/xtext/resource/IResourceDescription.html) and of [EObject](http://download.eclipse.org/modeling/emf/emf/javadoc/2.9.0/org/eclipse/emf/ecore/EObject.html) using [IEObjectDescription](http://download.eclipse.org/modeling/tmf/xtext/javadoc/2.3/org/eclipse/xtext/resource/IEObjectDescription.html). The later provides the following details - Simple name, Qualified name, EObject URI, EClass, additional user data.

The index is global, all the indexed objects are globally accessible/referenceable. Visibility is handled using [IContainer](http://download.eclipse.org/modeling/tmf/xtext/javadoc/2.3/org/eclipse/xtext/resource/IContainer.html). Objects having name (string attribute `name`) will land into the index. The name computation is delegated to the [IQualifiedNameProvider](http://download.eclipse.org/modeling/tmf/xtext/javadoc/2.3/org/eclipse/xtext/naming/IQualifiedNameProvider.html) and the decision to create an [IEObjectDescription](http://download.eclipse.org/modeling/tmf/xtext/javadoc/2.3/org/eclipse/xtext/resource/IEObjectDescription.html) instance is delegated to the [IDefaultResourceDescriptionStrategy](http://download.eclipse.org/modeling/tmf/xtext/javadoc/2.3/org/eclipse/xtext/resource/IDefaultResourceDescriptionStrategy.html). As a DSL author, you have the flexibility to decide what goes into the index by defining a custom ResourceDescriptionStrategy. 

The type [IResourceDescriptions](http://download.eclipse.org/modeling/tmf/xtext/javadoc/2.3/org/eclipse/xtext/resource/IResourceDescriptions.html) provides a very basic API to get hold of the exported objects and the Resource descriptions. As your DSL grows or the number of DSL's in your project increase, the API might look insufficient. In such situations, a "local index" can come handy. It basically wraps up the global index and provides a much richer API, abstracting the API consumer from the complexity of the actual index. You could have one local index per dsl.

```
import com.google.inject.Inject
import org.eclipse.emf.ecore.EClass
import org.eclipse.emf.ecore.EObject
import org.eclipse.xtext.resource.IContainer
import org.eclipse.xtext.resource.impl.ResourceDescriptionsProvider
import org.xtext.example.mydsl.myDsl.MyDslPackage
import org.eclipse.xtext.naming.QualifiedName

class EntityDslIndex {

	@Inject ResourceDescriptionsProvider indexProvider

	@Inject IContainer.Manager containerManager
	
	// Returns all EOD's with EClass matching Entity EClass
	def getAllEntities(EObject context) {
		val index = indexProvider.getResourceDescriptions(context.eResource)
		index.getExportedObjectsByType(MyDslPackage.Literals.ENTITY)
	}

	// Returns all EOD's from the current resource with EClass matching Entity EClass
	def getEntities(EObject context) {
		val resource = context.eResource
		val index = indexProvider.getResourceDescriptions(resource)
		val resourceDescription = index.getResourceDescription(resource.URI)
		resourceDescription.getExportedObjectsByType(MyDslPackage.Literals.ENTITY)
	}
	
	// Return EOD's with EClass matching Entity EClass and qualifiedName matching qName 
	def getEntity(EObject context, QualifiedName qName) {
		context.entities.filter[it.qualifiedName.equals(qName)]?.head
	}
	
	// Return EOD's from the current resource and all the visible resources
	def getAllVisibleEntities(EObject context) {
		context.getVisibleEObjectsByType(MyDslPackage.Literals.ENTITY)
	}

	private def getVisibleEObjectsByType(EObject context, EClass type) {
		context.visibleContainers.map[getExportedObjectsByType(type)].flatten
	}

	private def getVisibleContainers(EObject context) {
		val resource = context.eResource
		val index = indexProvider.getResourceDescriptions(resource)
		val resourceDescription = index.getResourceDescription(resource.URI)
		if (resourceDescription === null) {
			return emptyList
		}
		val visibleContainers = containerManager.getVisibleContainers(resourceDescription, index)
		visibleContainers
	}
}
```

The call to [indexProvider.getResourceDescriptions](http://download.eclipse.org/modeling/tmf/xtext/javadoc/2.3/org/eclipse/xtext/resource/impl/ResourceDescriptionsProvider.html#getResourceDescriptions(org.eclipse.emf.ecore.resource.Resource)) returns a index instance based on the context in which the resource set is used. The context is indicated by the load options ([resourceSet.getLoadOptions()](http://download.eclipse.org/modeling/emf/emf/javadoc/2.9.0/org/eclipse/emf/ecore/resource/ResourceSet.html#getLoadOptions())).

The call to [containerManager.getVisibleContainers](https://goo.gl/Pxasn2) on a [IContainer.Manager](http://download.eclipse.org/modeling/tmf/xtext/javadoc/2.3/org/eclipse/xtext/resource/IContainer.Manager.html) returns the [IContainer](http://download.eclipse.org/modeling/tmf/xtext/javadoc/2.3/org/eclipse/xtext/resource/IContainer.html) and all the visible containers. Following which, the call to [getExportedObjects](http://download.eclipse.org/modeling/tmf/xtext/javadoc/2.3/org/eclipse/xtext/resource/ISelectable.html#getExportedObjects()) on all the visible containers returns the [IEObjectDescription](http://download.eclipse.org/modeling/tmf/xtext/javadoc/2.3/org/eclipse/xtext/resource/IEObjectDescription.html) elements that are externally visible (globally exported) from a given resource.

---
PS - The code above is written using [Eclipse Xtend](https://www.eclipse.org/xtend/)                                                     
References - [Documentation](http://www.eclipse.org/Xtext/documentation/2.5.0/Xtext%20Documentation.pdf), [Implementing Domain-Specific Languages with Xtext and Xtend](https://goo.gl/J4ijTn)
