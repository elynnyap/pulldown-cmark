## How the algorithm works

We make two passes over the input, producing an intermediate AST in the first pass, and a finalized AST in the second pass.

In the first pass, we construct an AST that captures all the information about how to parse the document (such as the block structure of the document, container blocks, link reference definitions), except for inlines. 

In the second pass, we traverse the intermediate AST that was constructed from the first pass, producing Events from this representation. Our tree walk combines both pre- and post- order traversal: for any non-leaf node, we first evaluate it as a starting tag before traversing all its descendants, then backtrack to evaluate the node as an ending tag. 

The reason why a second pass is needed is because the intermediate AST will contain nodes that are of type ```ItemBody::Inline```, indicating that these components may potentially form valid inline elements. It is during the second pass that we determine if these nodes are indeed part of valid inline elements, while applying the precedence rules for inlines (e.g. parsing for links before parsing for emphasis/strong emphasis).

When a valid inline element is found, we perform "tree surgery" to modify the AST (more below). In order to facilitate these surgical operations, the Node struct has a value for a child pointer (which points to its first direct child), a next pointer (which points to its first sibling), and an item (which represents its contents) -- rather than a traditional Node struct which has a vector of all its children. 

For example, given the input ```[a*b](url*)Word**```, the intermediate AST produced from the first pass looks like this:

```
Inline '[' -> Text 'a' -> Inline '*' -> Text 'b' -> Inline ']' -> Text 'url' -> Inline '*' -> Text ')' -> Text 'Word**'
```

We then traverse the tree starting from the first node, ```Inline '['```. Scanning forward, we see that there is a valid inline link. We therefore surgerize the tree to produce a Link node and associate it with its children (downward-pointing arrows are child pointers, while rightward-pointing arrows are next pointers).

```
Link -> Text 'Word**'
 |
Text 'a*b' -> Destination 'url*'
```
