---
layout: default
title: "EMF EPackage Registry, Resource Factory, and Resource Factory Registry"
date: 2020-02-04
categories: main
---

# Background

An EPackage in EMF is the source of the model meta-data (eClass, eAttribute…etc). The generated EPackage provides the API for accessing the meta-data, programmatically builds the ECore model, which we typically create using the ECore editor, and registres the created EPackage in the Global EPackage registry.

EMF has two registries - EPackage registry, and a Resource Factory registry. The former is used for mapping the namespace URI (NsUri) of an EPackage with the EPackage instance. The later is used for registering the factory used for creating an EMF Resource.

# EPackage registry

EPackage registry is used for registering EMF EPackages instances. Once registered, it is used to recreate the instances of the serialized models (using EPackage#getEFactoryInstance()) when the resource is parsed and loaded in the memory.

The registration of an EPackage can be done in either the Local (ResourceSet#getPackageRegistry()) or the Global (EPackage.REGISTRY.INSTANCE) package registry. Typically the local registry is first checked for the package, and then the global registry. The PackageNotFound exception is thrown if the package is not found in either of these registries. The following code demonstrates the package registration process

## In Global Registry

```
final CreditcardPackage einstance = CreditcardPackage.eINSTANCE;
```

## In Local Registry

```
final ResourceSet resourceSet = new ResourceSetImpl();

resourceSet.getPackageRegistry().put(CreditcardPackage.eNS_URI, CreditcardPackage.eINSTANCE);
resourceSet.getResourceFactoryRegistry().getExtensionToFactoryMap().put("xml", new XMIResourceFactoryImpl());

final Resource resource = resourceSet.getResource(URI.createFileURI(new File("cbp.xml").getAbsolutePath()), true);
final EObject eObject = resource.getContents().get(0);
```

# EMF Resource Factory and Resource Factory Registry

A Resource Facory is used for creating the EMF Resource instances. It is used when you call the method `ResourceSet#createResource(URI)`. The selection of a ResourceFactory is based on the scheme, the protocol, or the file extension of the passed URI. The factory also decides the format in which the objects are to be persisted. For example, the default EMF Resource Factory `XMLResourceFactoryImpl` is used for creating XML Resources, and for persisting the object in the XML format.

A Resource factory can be registered either in the Local (`ResourceSet#getResourceFactoryRegistry()`) or a Global (`Resource.Factory.Registry.INSTANCE`) Resource Factory registry. Similar to the EPackage registry, a request for the factory is first made in the local registry, and then the global registry. A null value for`ResourceSet#createResource(URI)` is returned if the factory cannot be found in either of these registries. The following code demonstrates the registration process

## In Global Registry

```
Resource.Factory.Registry.INSTANCE.getExtensionToFactoryMap().put("xml", new XMIResourceFactoryImpl());
```

## In Local Registry

```
final ResourceSet resourceSet = new ResourceSetImpl();
resourceSet.getResourceFactoryRegistry().getExtensionToFactoryMap().put("xml", new XMIResourceFactoryImpl());
final Resource resource = resourceSet.createResource(URI.createURI("test.xml"));
```

# Dos and Don'ts

- Explicit registration of an EPackage is required only if you are running the application in a standalone mode. When running under Eclipse, the registration happens automatically via the extension in the manifest file of your EMF model project

```
<extension point="org.eclipse.emf.ecore.generated_package">
      <package uri="http://www.gyaltso.com/training/emf/model"
            class="com.mycompany.training.emf.model.creditcard.CreditcardPackage"
            genModel="model/creditcard.genmodel"/>
</extension>
```

- Similarly, explicit registration of a Resource Factory is required in standalone mode. When running under Eclipse, the registration happens automatically via the extension in the manifest file of the **org.eclipse.emf.ecore.xmi** plug- in

```
<extension point="org.eclipse.emf.ecore.extension_parser">
    <parser type="*" class="org.eclipse.emf.ecore.xmi.impl.XMIResourceFactoryImpl"/>
</extension>
```

- In order to get the registred EPackage instance from the registry, the method `Registry#getEPackage(NsUri)` should be used. At times, an `EPackageDescriptor` is registered in place of the actual EPackage instance in the registry. The actual EPackage is resolved and replaced in the registry when the request for the same is made. The process of resolution is altogether omitted if you do not use the `getEPackage(NsUri)` method to get the EPackage.

# References

[EMF: Eclipse Modeling Framework Book](https://www.amazon.com/EMF-Eclipse-Modeling-Framework-2nd/dp/0321331885),
[EMF discussion forum](https://www.eclipse.org/forums/index.php?t=thread&frm_id=108)
