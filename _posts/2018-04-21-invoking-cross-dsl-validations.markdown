---
layout: default
title:  "Invoking cross DSL Validations"
date:   2018-04-21
categories: main
---

# Invoking Cross DSL Validations

In DSL's, it is quite common to have Custom validations. With Xtext, such validations are defined in the generated Validator class by annotating the methods with [@Check](http://download.eclipse.org/modeling/tmf/xtext/javadoc/2.9/org/eclipse/xtext/validation/Check.html) annotation. Behind the scenes, these methods are invoked reflectively by the framework while the user of the DSL is typing in the editor. In addition, you could use [CheckType](http://download.eclipse.org/modeling/tmf/xtext/javadoc/2.9/org/eclipse/xtext/validation/CheckType.html) to control when a given validation method should be called.

Typically, in projects, you'll have more than one DSL's, each having it own custom validator. There are situations where you want a certain validation to be invoked which is "NOT" associated with the DSL being modified. For example - In the below DSL's, if I define a class as "deprecated", all its occurrences should be highlighted with an error (i.e. the parameters in the TestFunction should be highlighted with an error).

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

In Xtext, an [IEObjectDescription](http://download.eclipse.org/modeling/tmf/xtext/javadoc/2.9/org/eclipse/xtext/resource/IEObjectDescription.html) is used for representing a model object (EObject). It has a name, URI and may have some additional data. In the example above, making the Customer class as deprecated won't alter the "state" of the associated EOD, by virue of which the the Xtext builder will fail in invoking the validator for the Function DSL (PS - [ClusteringBuilderState#queueAffectedResources()](https://github.com/eclipse/xtext-eclipse/blob/master/org.eclipse.xtext.builder/src/org/eclipse/xtext/builder/clustering/ClusteringBuilderState.java)). The validator for the Clazz DSL will only be called.

There are two ways to solve the problem

1. Customizing the DefaultResourceDescriptionManager
2. Adding data while creating IEObjectDescription 

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
The above changes ensure that the passed IResourceDescription is added to the queue of the affected resources to be processed.

The method [DefaultResourceDescriptionManager#hasChanges()](http://download.eclipse.org/modeling/tmf/xtext/javadoc/2.9/org/eclipse/xtext/resource/impl/DefaultResourceDescriptionManager.html#hasChanges(org.eclipse.xtext.resource.IResourceDescription.Delta,%20org.eclipse.xtext.resource.IResourceDescription)) checks if the [passed delta has changes](http://download.eclipse.org/modeling/tmf/xtext/javadoc/2.9/org/eclipse/xtext/resource/IResourceDescription.Delta.html#haveEObjectDescriptionsChanged()). We override the method and return true to indicate that the passed delta has changes even if there is no actual change.  

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

Above, we add User data while creating an EObjectDescription for the Clazz EObjects. Now, in the example above, if you change the Customer Clazz from deprecated to non-deprecated or vice versa, the user-data changes. This changes the state of the EObjectDescription representing the Customer EObject changes.
This being said, the call to [DefaultResourceDescriptionManager#hasChanges()](http://download.eclipse.org/modeling/tmf/xtext/javadoc/2.9/org/eclipse/xtext/resource/impl/DefaultResourceDescriptionManager.html#hasChanges(org.eclipse.xtext.resource.IResourceDescription.Delta,%20org.eclipse.xtext.resource.IResourceDescription)) returns true for IResourceDescription representing the TestFunction. The same is added to the queue of resource affected by the build and the validator for the Function DSL is called along with the Class DSL.

---
Source code - [ClazzDslResourceDescriptionStrategy](https://github.com/nbhusare/Xtext-sandbox/blob/master/org.neclipse.xtext.validator.example.clazzdsl/src/org/neclipse/xtext/validator/example/clazzdsl/descriptions/ClazzDslResourceDescriptionStrategy.xtend), [FuncDslResourceDescriptionManager](https://github.com/nbhusare/Xtext-sandbox/blob/master/org.neclipse.xtext.validator.example.funcdsl/src/org/neclipse/xtext/validator/example/funcdsl/descriptions/FuncDslResourceDescriptionManager.java)

References - [https://www.eclipse.org/forums/index.php/m/1724947/](https://www.eclipse.org/forums/index.php/m/1724947/)