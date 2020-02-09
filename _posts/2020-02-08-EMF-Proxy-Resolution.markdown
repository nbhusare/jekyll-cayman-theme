---
layout: default
title: "EMF Proxy resolution"
date: 2020-02-08
categories: main
---

# **EMF Proxy resolution**

In [EMF](https://www.eclipse.org/modeling/emf/), an `EReference` represents a relation between two objects (Source and the Target). By setting value of the **Containment** property, it can be modeled as **containment** or a **non-containment** reference. In the earlier case, the target object is contained inside the source (its container). It persists in the same resource as its container and can give the information of its container (using `eObject.eContainer()`). In the latter case, the two objects can be persisted in the same or different resources. Typically, Proxy resolution is related but-not-limited to the latter case.

When you talk of proxy resolution in EMF, two properties play an important role - **Resolve Proxies** and **Containment Proxies**. The earlier can be modified from the .ecore file and the later from the .genmodel file. The value of these two fields decide whether a reference is **proxy resolving** or **non-proxy resolving**, and the code is generated accordingly. By default, non-containment references are proxy resolving, whereas, containment references are non-proxy resolving. Below we explain in a bit more detail using the generated code as a reference

## EReference (non-containment)

The below code is generated for the reference `region`. It is a **single-valued, proxy resolving** reference. The value of the field "Resolve Proxies" is set to true by default. The generated code checks if the reference is a proxy and tries to resolve it. After resolution, the proxy object is replaced with the resolved object instance. It is retained if the resolution fails

```
public Region getRegion() {
  if (region != null && region.eIsProxy()) {
  InternalEObject oldRegion = (InternalEObject)region;
  region = (Region)eResolveProxy(oldRegion);
  ...
  }
  return region;
}
```

## EReference (containment)

The below code is generated for the reference `features`. It is a **multi-valued, non-proxy resolving, containment** reference. An interesting thing here is that the generated code does not contain any proxy resolution logic, even though the value of the "Resolve Proxies" filed is true. This is because the value of "Containment Proxies" value is set to **false** (default value).

In the case of containment references, the combination of the above two fields decide whether the generated code contains proxy resolution or non-proxy resolution logic

```
public EList<Feature> getFeatures() {
	if (features == null) {
		features = new EObjectContainmentEList<Feature>(Feature.class, this,
		CreditcardPackage.PRODUCT__FEATURES);
	}
	return features;
}
```

In the above code, a simple `EObjectContainmentEList` is generated.

## EReference (containment)

The below code is generated for the reference features. Like above, it is a **multi-valued, non-proxy resolving, containment** reference, with one difference, the value of "Containment Proxies" field is set to **true**. By setting this value, we make it possible for the source and the contained object to persist in different resources. This is quite powerful as it now allows the contained objects to be dealt with separately as top-level objects

```
 public EList<Feature> getFeatures() {
	if (features == null) {
		features = new EObjectContainmentEList.Resolving<Feature>(Feature.class, this,
		CreditcardPackage.PRODUCT__FEATURES);
	}
	return features;
}
```

In the above code, the generated EList implementation `EObjectContainmentEList.Resolving` does proxy resolution.

## Dos and Don'ts

- For a non-containment reference, it is beneficial to set the value of "Resolve Proxies" to false if the source and the target objects are supposed to be persisted in the same resource. This will simplify the generated code, as it will no more contain the proxy resolution logic

- Setting the value of the "Containment Proxies" property to true will make all the contained references in the model as proxy resolving. The value of the "Resolve Proxies" property should be explicitly set to **false** for those containment references for which proxy resolution should be disabled.

## References

[EMF: Eclipse Modeling Framework Book](https://www.amazon.com/EMF-Eclipse-Modeling-Framework-2nd/dp/0321331885),
[EMF discussion forum](https://www.eclipse.org/forums/index.php?t=thread&frm_id=108)
