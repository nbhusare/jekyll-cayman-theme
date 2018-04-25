---
layout: default
title:  "Invoking cross DSL Validations"
date:   2018-04-21
categories: main
---

# Invoking Cross DSL Validations

In DSL's, it is quite common to have Custom validations. With Xtext, such validations are defined in the generated Validator class by annotating the methods with [@Check](http://download.eclipse.org/modeling/tmf/xtext/javadoc/2.9/org/eclipse/xtext/validation/Check.html) annotation. Behind the scenes, these methods are invoked reflectively by the framework while the user of the DSL is typing in the editor so that an immediate feedback is provided. In addition, you could use a [CheckType](http://download.eclipse.org/modeling/tmf/xtext/javadoc/2.9/org/eclipse/xtext/validation/CheckType.html) value to control when a given validation method should be called.

Typically, in projects, you'll have more than one DSL's, each having it own custom validator. There are situations where you want a certain validation to be invoked that is "not" associated with the DSL being modified. For example - In the below DSL's, if I define a class as "deprecated", all its occurrences should be highlighted with an error. 

**Class DSL**
```
namespace org.neclipse.example
<deprecated> clazz Customer {  
    String name 
    String age }  
```

**Function DSL**
```
namespace org.neclipse.example
import org.neclipse.example.Customer
func TestFunction (Customer customer, String param1, Customer param2...) { } 
```

In Xtext, an [IEObjectDescription](http://download.eclipse.org/modeling/tmf/xtext/javadoc/2.9/org/eclipse/xtext/resource/IEObjectDescription.html) is used for representing a model object (EObject). It has a name, URI and may have some additional data. In the example above, making the Customer class as deprecated won't alter the state of the associated EOD. The Xtext builder will invoke the validations for the Class DSL but fail in finding all the related resources that are affected (i.e. TestFunction in our case), which is why the validator for the Function DSL won't be called (PS - [ClusteringBuilderState#queueAffectedResources()](https://github.com/eclipse/xtext-eclipse/blob/master/org.eclipse.xtext.builder/src/org/eclipse/xtext/builder/clustering/ClusteringBuilderState.java)). 

There are two ways to solve the problem

1. Customizing the DefaultResourceDescriptionManager
2. Creating IEObjectDescription with additional Data

### 1. Customizing the DefaultResourceDescriptionManager

```
@Singleton
public class FuncDslResourceDescriptionManager extends DefaultResourceDescriptionManager
        implements IResourceDescription.Manager.AllChangeAware {

    @Override
    public boolean isAffectedByAny(Collection<Delta> deltas, IResourceDescription candidate,
            IResourceDescriptions context) throws IllegalArgumentException {
        return isAffected(deltas, candidate, context);
    }

    @Override
    protected boolean hasChanges(Delta delta, IResourceDescription candidate) {
        return true;
    }
}
```
Above, the FuncDslResourceDescriptionManager implements the [IResourceDescription.Manager.AllChangeAware](http://download.eclipse.org/modeling/tmf/xtext/javadoc/2.9/org/eclipse/xtext/resource/IResourceDescription.Manager.AllChangeAware.html) interface so that it is notified for all changes. In the overridden method  [isAffectedByAny()](http://download.eclipse.org/modeling/tmf/xtext/javadoc/2.9/org/eclipse/xtext/resource/IResourceDescription.Manager.AllChangeAware.html#isAffectedByAny(java.util.Collection,%20org.eclipse.xtext.resource.IResourceDescription,%20org.eclipse.xtext.resource.IResourceDescriptions)), we invoke the super [isAffected()](http://download.eclipse.org/modeling/tmf/xtext/javadoc/2.9/org/eclipse/xtext/resource/impl/DefaultResourceDescriptionManager.html#isAffected(java.util.Collection,%20org.eclipse.xtext.resource.IResourceDescription,%20org.eclipse.xtext.resource.IResourceDescriptions)) which checks if the passed IResourceDescription (candidate here) is affected by the given deltas. In the overridden method **hasChanges()**, we always return true to indicates that all the deltas resulting from the build have changes, even if there is no change in the IEObjectDescription. 

### 2. Creating IEObjectDescription with additional Data

```
class ClazzDslResourceDescriptionStrategy extends DefaultResourceDescriptionStrategy {

    private final static Logger LOG = Logger.getLogger(ClazzDslResourceDescriptionStrategy)

    private static val DEPRECATED = "deprecated"

    override createEObjectDescriptions(EObject eObject, IAcceptor<IEObjectDescription> acceptor) {
        if (qualifiedNameProvider !== null) {
            try {
                val qualifiedName = qualifiedNameProvider.getFullyQualifiedName(eObject)
                if (qualifiedName !== null) {
                    acceptor.accept(EObjectDescription.create(qualifiedName, eObject, createUserData(eObject)));
                    return true
                }
            } catch (Exception exc) {
                LOG.error(exc.getMessage(), exc);
            }
        }
        return false
    }

    def createUserData(EObject eObject) {
        val Builder<String, String> userData = ImmutableMap.builder()
        if (eObject instanceof Clazz) {
            userData.put(DEPRECATED, Boolean.toString(eObject.isDeprecated))
        }
        return userData.build
    }
}
```

Above, the method #createUserData() is called during the creation of the EObjectDescription. It creates a map and adds an entry key is the "deprecated" string constant and the value is the boolean value of the structural feature "deprecated". The entry is updated every time you make a change to the deprecated feature.


---
