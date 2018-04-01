---
layout: default
title:  "Hyperlink support for Import statements"
date:   2018-03-31
categories: main
---

# Hyperlink support for Import statements

In Xtext DSL's, it is quite common to have cross-references (represented using **referenceName = [reference-type]**). You can have cross-references within or across files (a.k.a [Resource](http://download.eclipse.org/modeling/emf/emf/javadoc/2.4.2/org/eclipse/emf/ecore/resource/Resource.html])). In the latter case, you'll require imports in order to reference the types. You could choose to have [implicit-imports](http://download.eclipse.org/modeling/tmf/xtext/javadoc/2.9/org/eclipse/xtext/scoping/impl/ImportedNamespaceAwareLocalScopeProvider.html), but that is out of the scope of the current discussion.

In the editor, shortcuts "CTRL + Left Mouse click or F3" can be used for navigation. It works out-of-the-box for cross-references, but fails for import statements, especially for languages that extend from the Terminals grammar. 
Hyperlink detection for import statements can be enabled by customizing the [HyperlinkHelper](http://download.eclipse.org/modeling/tmf/xtext/javadoc/2.3/org/eclipse/xtext/ui/editor/hyperlinking/HyperlinkHelper.html).

```
public class EntityDslHyperlinkHelper extends TypeAwareHyperlinkHelper {

    @Inject
    private IQualifiedNameConverter.DefaultImpl qNameConvertor;

    @Inject
    private EntityDslIndex entityDslIndex;

    @Override
    public void createHyperlinksByOffset(XtextResource resource, int offset, IHyperlinkAcceptor acceptor) {
        final EObject eObject = getEObjectAtOffsetHelper().resolveElementAt(resource, offset);
        if (eObject instanceof Import) {
            final ImportWrapper iImport = new ImportWrapper((Import) eObject);
            if (!iImport.isWildcard()) {
                final Optional<ITextRegion> textRegion = getTextRegion(iImport.getImport(), offset);
                if (textRegion.isPresent()) {
                    final Region region = new Region(textRegion.get().getOffset(), textRegion.get().getLength());
                    final Optional<EObject> importedEntity = iImport.getImportedEntity();
                    if (importedEntity.isPresent())
                        this.createHyperlinksTo(resource, region, importedEntity.get(), acceptor);
                    return;
                }
            }
        }
        super.createHyperlinksByOffset(resource, offset, acceptor);
    }

    protected Optional<ITextRegion> getTextRegion(Import iImport, final int offset) {
        final List<INode> nodes = NodeModelUtils.findNodesForFeature(iImport,
                EntityDslPackage.Literals.IMPORT__IMPORTED_NAMESPACE);
        return nodes.stream().map(INode::getTextRegion).filter(textRegion -> textRegion.contains(offset)).findFirst();
    }

    private class ImportWrapper {

        private final Import iImport;

        public ImportWrapper(final Import iImport) {
            this.iImport = iImport;
        }

        public Import getImport() {
            return iImport;
        }

        protected boolean isWildcard() {
            final QualifiedName qualifiedImport = qNameConvertor.toQualifiedName(iImport.getImportedNamespace());
            return qualifiedImport.getLastSegment().equals("*");
        }

        protected Optional<EObject> getImportedEntity() {
            final QualifiedName entityQName = qNameConvertor.toQualifiedName(iImport.getImportedNamespace());
            final IEObjectDescription importedEod = entityDslIndex.getEntity(entityQName, iImport);
            if (importedEod == null)
                return Optional.empty();

            final EObject eObject = importedEod.getEObjectOrProxy();
            final EObject resolved = eObject.eIsProxy()
                    ? EcoreUtil.resolve(eObject, iImport.eResource().getResourceSet())
                    : eObject;
            return Optional.of(resolved);
        }

    }
}
```
In the overridden method #createHyperlinksByOffset(), we do the following - 1) Check if the object at a given offset is an instance of [Import](https://github.com/nbhusare/Xtext-sandbox/blob/master/org.nb.xtext.example.hyperlink.entitydsl/src-gen/org/nb/xtext/example/hyperlink/entitydsl/entityDsl/Import.java) and wrap it in the ImportWrapper, 2) Resolve the imported Entity Object using ImportWrapper#getImportedEntity(), 3) Call method #createHyperlinksTo() that creates an [XtextHyperlink](http://download.eclipse.org/modeling/tmf/xtext/javadoc/2.9/org/eclipse/xtext/ui/editor/hyperlinking/XtextHyperlink.html) instance.

The ImportWrapper provides a utility API for working with Import objects. The class [EntityDslIndex](https://github.com/nbhusare/Xtext-sandbox/blob/master/org.nb.xtext.example.hyperlink.entitydsl/src/org/nb/xtext/example/hyperlink/entitydsl/index/EntityDslIndex.xtend) provides utility API for working with the Xtext index (PS - [Enriching Xtext Index API](https://nbhusare.github.io/main/2017/08/07/enriching-xtext-index-api.html))

---
Source - [org.nb.xtext.example.hyperlink.entitydsl](https://github.com/nbhusare/Xtext-sandbox/tree/master/org.nb.xtext.example.hyperlink.entitydsl), [org.nb.xtext.example.hyperlink.entitydsl](https://github.com/nbhusare/Xtext-sandbox/tree/master/org.nb.xtext.example.hyperlink.entitydsl.ui)
References - [XbaseHyperLinkHelper](http://download.eclipse.org/modeling/tmf/xtext/javadoc/2.9/org/eclipse/xtext/xbase/ui/navigation/XbaseHyperLinkHelper.html), [https://www.eclipse.org/forums/index.php/t/521907/](https://www.eclipse.org/forums/index.php/t/521907/)
Discussion - [https://www.eclipse.org/forums/index.php/t/1092463/](https://www.eclipse.org/forums/index.php/t/1092463/)