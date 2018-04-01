# Customizing Xtext Hyperlink helper

In Xtext DSL's, it is quite common to have cross-references (represented using referenceName=[reference-type]). You can have cross-references within a file or across files (a.k.a (Resource)[http://download.eclipse.org/modeling/emf/emf/javadoc/2.4.2/org/eclipse/emf/ecore/resource/Resource.html]). In the latter case, you'll require imports in order to reference the types. You could have implicit imports (((ImportedNamespaceAwareLocalScopeProvider)[http://download.eclipse.org/modeling/tmf/xtext/javadoc/2.9/org/eclipse/xtext/scoping/impl/ImportedNamespaceAwareLocalScopeProvider.html]), but that is out of the scope of the current discussion.

In the editor, shortcuts "CTRL + Left Mouse click or F3" can be used for navigating to the cross-referenced objects. It works out-of-the-box for corss-references, but fails for import statements, especially if you language is extending from the Terminals grammar. 
Hyperlink detection for import statements can be enabled by customizing the [HyperlinkHelper] (http://download.eclipse.org/modeling/tmf/xtext/javadoc/2.3/org/eclipse/xtext/ui/editor/hyperlinking/HyperlinkHelper.html). It is used for detecting hyperlinks at a given offset in a Xtext Resource. 

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
In the overridden method #createHyperlinksByOffset(), we check if the resolved element at a given offset is an instance of Import type and create a hyperlink object. The ImportWrapper provides utility API to work with the Import object. The method getImportedEntity() makes use of the [EntityDslIndex](link here) to get an EObjectDescription with a given qualified name and resolves it if proxy.