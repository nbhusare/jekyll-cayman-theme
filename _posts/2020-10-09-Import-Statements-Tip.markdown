---
layout: default
title: "Import Statement tip"
date: 2020-10-09
categories: main
---

# **Import Statement tip**

```
EntityFile:
	'package' name=QName
	imports+=Import*
	entities+=Entity*;
Import:
	'import' importedNamespace=QNameWithWildcard;
QName:
	ID ('.' ID)*;
QNameWithWildcard:
	QName ('.*')?;
Entity:
	'entity' name=ID ('parent' parent=[Entity])?;
```

Using the above grammar definition, you can define entities and set a parent of a given entity. The parent can be defined in the same or a separate file. In the latter case, you need to define an import statement (import package_name.\* or import package_name.Entity_name) for the referenced entity to be resolved. The problem is that you have to define imports even if the two (entity and its parent) are defined in the same package (PS below)

```
package accounts
entity Account
```

```
package accounts
import accounts.Account
entity SpecialAccount parent Account
```

The scope provider provides the objects (of a certain type) that are reachable at a given point in the DSL. The local scope provider provides the ones defined in the same resource, and the global scope provider provides that defined in a separate resource. The ImportedNamespaceAwareLocalScopeProvider comes into the picture if you have imports in your grammar definition.

The above problem can be resolved by overriding the method ImportedNamespaceAwareLocalScopeProvider#internalGetImportedNamespaceResolvers() as shown below. We explicitly add an import statement that has a qualified name that matches the current package name. This avoids the need to define explicit imports if the two entities are defined in the same package.

```
public class EntityDslImportedNamespaceAwareLocalScopeProvider extends ImportedNamespaceAwareLocalScopeProvider {

	@Override
	protected List<ImportNormalizer> internalGetImportedNamespaceResolvers(EObject context, boolean ignoreCase) {
		List<ImportNormalizer> importNormalizers = super.internalGetImportedNamespaceResolvers(context, ignoreCase);
		if (context instanceof EntityFile) {
			String packageName = ((EntityFile) context).getName();
			importNormalizers.add(createImportedNamespaceResolver(packageName + ".*", ignoreCase));
		}
		return importNormalizers;
	}}
```

Reference: [Deep dive into Xtext scoping - local and global scopes](https://www.youtube.com/watch?v=8WDyST9EIZc) explained, by H. Schill & S. Zarnekow

Happy Xtexting!
