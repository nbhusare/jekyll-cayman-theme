---
layout: default
title: "EMF EPackage and Resource Factory Registry"
date: 2020-02-04
categories: main
---

# EMF EPackage and Resource Factory Registry

An EPackage in [EMF](https://www.eclipse.org/modeling/emf/) is the source of the model meta-data (eClass, eAttribute…etc). The generated EPackage class provides API for accessing the meta-data, it programmatically builds the ECore model (typically created using the ECore editor), and registers the created EPackage in the Global EPackage registry.

The registries in EMF, EPackage registry and the Resource Factory registry, are used for mapping the namespace URI (NsUri) of an EPackage with the EPackage instance and registering the factory used for creating an EMF Resource, respectively.

## EPackage registry

An EPackage registry is used for registering EMF [EPackages](https://download.eclipse.org/modeling/emf/emf/javadoc/2.5.0/org/eclipse/emf/ecore/EPackage.html) instances. Once registered, it is used for creating the instances of the serialized objects (using EPackage#getEFactoryInstance()) when the resource is parsed and loaded in the memory.

The registration of an EPackage can be done in either Local (`ResourceSet#getPackageRegistry()`) or the Global (`EPackage.REGISTRY.INSTANCE`) package registry. Typically the requested EPackage is first checked in the local registry, and then the global registry. A `PackageNotFound` exception is thrown if the package cannot be found in either of these registries. The following code demonstrates the registration process.

### In Global Registry

```
final CreditcardPackage einstance = CreditcardPackage.eINSTANCE;
```

### In Local Registry

```
final ResourceSet resourceSet = new ResourceSetImpl();
resourceSet.getPackageRegistry().put(CreditcardPackage.eNS_URI, CreditcardPackage.eINSTANCE);
resourceSet.getResourceFactoryRegistry().getExtensionToFactoryMap().put("xml", new XMIResourceFactoryImpl());

final Resource resource = resourceSet.getResource(URI.createFileURI(new File("cbp.xml").getAbsolutePath()), true);
final EObject eObject = resource.getContents().get(0);
```

## EMF Resource Factory and Resource Factory Registry

When you call `ResourceSet#createResource(URI)`, the Resource Factory is consulted to create a new Resource instance. The scheme, protocol, or the file extension of the passed URI is used while selecting a partiular resource factory. It also decides the format in which the objects are to be persisted. For example, the default EMF Resource Factory `XMIResourceFactoryImpl` is used for creating `XMIResource`, and persist the object in the [XMI](https://www.omg.org/spec/XMI/About-XMI/) format.

A Resource factory can be registered either in Local (`ResourceSet#getResourceFactoryRegistry()`) or the Global (`Resource.Factory.Registry.INSTANCE`) registry. Similar to the EPackage registry, the requested factory is first checked in the local registry, and then the global registry. Calling `ResourceSet#createResource(URI)` returns a `null` value if the factory cannot be found in either of the above registries. The following code demonstrates the registration process

### In Global Registry

```
Resource.Factory.Registry.INSTANCE.getExtensionToFactoryMap().put("xml", new XMIResourceFactoryImpl());
```

### In Local Registry

```
final ResourceSet resourceSet = new ResourceSetImpl();
resourceSet.getResourceFactoryRegistry().getExtensionToFactoryMap().put("xml", new XMIResourceFactoryImpl());
final Resource resource = resourceSet.createResource(URI.createURI("test.xml"));
```

## Dos and Don'ts

- Explicit EPackage registration is required only if you are running the application in a standalone mode. Under Eclipse, the same happens automatically via the extension in the manifest file of your model project

  ```
  <extension point="org.eclipse.emf.ecore.generated_package">
        <package uri="http://www.yourcompany.com/emf/model"
              class="com.mycompany.training.emf.model.creditcard.CreditcardPackage"
              genModel="model/creditcard.genmodel"/>
  </extension>
  ```

- Similar to the above, explicit Resource Factory registration is required in standalone mode. UnderEclipse, the same happens automatically via the extension in the manifest file of the **org.eclipse.emf.ecore.xmi** plugin

  ```
  <extension point="org.eclipse.emf.ecore.extension_parser">
      <parser type="*" class="org.eclipse.emf.ecore.xmi.impl.XMIResourceFactoryImpl"/>
  </extension>
  ```

- At times, an EPackageDescriptor is registered in place of the actual EPackage. When a request for an EPackage
  is made, the descriptor is replaced lazily with the actual EPackage instance. This happens only if the request is made using method `Registry#getEPackage(NsUri)`

## References

[EMF: Eclipse Modeling Framework Book](https://www.amazon.com/EMF-Eclipse-Modeling-Framework-2nd/dp/0321331885),
[EMF discussion forum](https://www.eclipse.org/forums/index.php?t=thread&frm_id=108)
