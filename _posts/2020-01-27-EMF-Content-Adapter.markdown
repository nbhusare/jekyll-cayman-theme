---
layout: default
title: "EMF Content Adapter"
date: 2020-01-27
categories: main
---

# EMF Notification

In [EMF](https://www.eclipse.org/modeling/emf/), the generated model interface code extends from the EObject interface. This makes it possible to obtain the modeled object’s [Metadata](<https://download.eclipse.org/modeling/emf/emf/javadoc/2.5.0/org/eclipse/emf/ecore/EObject.html#eClass()>), the [Container](<https://download.eclipse.org/modeling/emf/emf/javadoc/2.5.0/org/eclipse/emf/ecore/EObject.html#eContainer()>), and the [Resource](<https://download.eclipse.org/modeling/emf/emf/javadoc/2.5.0/org/eclipse/emf/ecore/EObject.html#eResource()>) in which it is persisted. In addition, you are provided with an API used for accessing the object reflectively.

The interface [EObject](https://download.eclipse.org/modeling/emf/emf/javadoc/2.5.0/org/eclipse/emf/ecore/EObject.html) extends from the [Notifier](https://download.eclipse.org/modeling/emf/emf/javadoc/2.5.0/org/eclipse/emf/common/notify/Notifier.html) interface. An EObject instance sends a notification whenever the value of one of its structural feature changes. In the below code, a notification is sent whenever the value of the attribute `code` is changed. The call to `eNotificationRequired()` ensures that there are model change listeners before calling the `eNotify()` method.

```
public void setCode(String newCode) {
	String oldCode = code;
	code = newCode;
	if (eNotificationRequired())
		eNotify(new ENotificationImpl(this, Notification.SET,
		CreditcardPackage.REGION__CODE, oldCode, code));
}
```

# EMF Adapters

The Adapters in EMF are the model change listeners (A.K.A Observers). In addition, they can be used for extending the behavior of an EOject/Resource/ResourceSet without subclassing. The below code demonstrates its use

## Listening to model changes

```
public class RegionChangeListener extends AdapterImpl {
	@Override
	public void notifyChanged(Notification notification) {
		super.notifyChanged(notification);

		final String eClassName = ((Region) notification.getNotifier()).eClass().getName();
		final String featureName = ((EAttribute)notification.getFeature()).getName();
		System.out.println(String.format("Feature `%1$s` of EClass `%2$s` Changed", featureName, eClassName));
	}
}
```

The `RegionChangeListener` listens to the changes made to the `Region` model instance. The overridden method `notifyChanged()` is invoked whenever the state of the Region changes, for example, the value of the attribute `code` is updated. The parameter [Notification](https://download.eclipse.org/modeling/emf/emf/javadoc/2.5.0/org/eclipse/emf/common/notify/Notification.html) can be used for obtaining the information of the source EObject (A.K.A Notifier), the affected structural feature, the type of the change, and the new and the old values. The adapter can be hooked onto the Region instance using `region.eAdapters().add(new RegionChangeObserver()`, and removed using `region.eAdapters().remove(regionChangeAdaterInstance)`. Removing the adapter from an EObject will **remove it from all its contents**.

## Extending the EObject/Resource/ResourceSet behavior

```
public class ProjectAdapter extends AdapterImpl {

	Private final IProject project;

	public ProjectAdapter(IProject project) {
		This.project = project
	}
	public IProject getProject() {
		return project;
	}
	public boolean isAdapterOfType(Object object) {
		return object == IProject.class;
	}
}
```

```
resourceSet.eAdapters().add(new ProjectAdapter())
```

In the above example, we extend the behavior of the EMF [ResourceSet](https://download.eclipse.org/modeling/emf/emf/javadoc/2.5.0/org/eclipse/emf/ecore/resource/ResourceSet.html) by associating a `ProjectAdapter`. The extended ResourceSet can now give the information about the Project that was used for its creation. The associated ProjectAdapter and hench the Project can be obtained using `EcoreUtil.getAdapter(resourceSet.eAdapters(), IProject.class)`, and `projectAdapter.getProject()`.

# EMF Content Adapter

The above-explained approach of explicitly attaching an adapter is convenient while working with a handful of EObject’s. It easily becomes impractical when you have to deal with a deep object hierarchy. Iterating through the containment hierarchy and explicitly attaching the adapter makes sense only if you intend to do it for selected objects. You also have to manage the Adapter’s lifecycle - attaching and disposing of the adapters as the object hierarchy changes.

EMF provides [EContentAdapter](https://download.eclipse.org/modeling/emf/emf/javadoc/2.8.0/org/eclipse/emf/ecore/util/EContentAdapter.html) for situations where you want to be notified for changes to any object in the hierarchy. All you have to do is to attach the adapter to the **root-EObject/Resource/ ResourceSet**, and the adapter automatically attaches itself to all its contents. Once attached, it will receive the notification of the state changes to any of the objects in the hierarchy, and it will respond to the content change notifications itself, by attaching and detaching itself as appropriate. All this makes the EContentAdapter quite convenient.

The following code shows how an `EContentAdapter` can be defined and attached to an EObject

```
public class ProductStateChangeAdapter extends EContentAdapter {
	@Override
	public void notifyChanged(Notification notification) {
		// Your code comes here…
	}
}
```

```
product.eAdapters().add(new ProductStateChangeAdapter());
```

# Dos and Don'ts

- As the object hierarchy changes, the adapters should be attached and disposed of correctly. Not disposing of the adapters leads to memory leaks. They will be notified of every state change of the EObject and all its contents, which is not desirable. In some cases, exceptions are thrown
- EContentAdapters are convenient, however, attaching them to an EObject having a huge containment hierarchy can be quite time-consuming. In general, your first choice should be to attach adapters to the EObjects that are of interest for specific update operations
- Special care should be taken while implementing the overridden `notifyChanged()` method. It should not be a heavyweight operation or impact the user experience in any way
- `eAdapters()` returns a simple list. It is important that you check if the given adapter instance exists before adding to the list.

# References

[EMF: Eclipse Modeling Framework Book](https://www.amazon.com/EMF-Eclipse-Modeling-Framework-2nd/dp/0321331885),
[EMF discussion forum](https://www.eclipse.org/forums/index.php?t=thread&frm_id=108),
https://eclipsesource.com/blogs/2013/04/23/emf-dos-and-donts-7/, https://eclipsesource.com/blogs/2013/05/06/emf-dos-and-donts-8/
