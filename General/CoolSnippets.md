# Cool Snippets

This file contains snippets of code that can be useful sometimes.

- [Download a file with a progress bar](#download-a-file-with-a-progress-bar)
- [Bench and profile a project from the tests](#bench-and-profile-a-project-from-the-tests)
- [Automatic transformation of Pharo's methods source code](#automatic-transformation-of-pharo-s-methods-source-code)
- [Browse all available icons](#browse-all-available-icons)
- [Rename programatically methods](#rename-programatically-methods)

## Download a file with a progress bar

```Smalltalk
| outputFileName url |
outputFileName := './pharoLogo.png'.
outputFileName asFileReference ensureDelete.
url := 'http://files.pharo.org/media/logo/logo-flat.png' asZnUrl.
[ :bar | 
	bar title: 'Download: ' , url asString , ' to ' , outputFileName.
	[ ZnClient new
		url: url;
		signalProgress: true;
		downloadTo: outputFileName ]
		on: HTTPProgress
		do: [ :progress | 
			progress isEmpty ifFalse: [ bar current: progress percentage ].
			progress resume ] ] asJob run
```

## Bench and profile a project from the tests

```Smalltalk
packageSelectionBlock := [ :e | e name beginsWith: 'Spec' ].
testSuite := TestSuite new.
	
((RPackageOrganizer default packages select: packageSelectionBlock) flatCollect: #classes) select: [ :e | e inheritsFrom: TestCase ] thenDo: [ :e | e addToSuiteFromSelectors: testSuite ].

"Bench the test suite"	
[ testSuite run ] bench.

"Profile the test suite"
TimeProfiler spyOn: [ testSuite run ]

```

## Automatic transformation of Pharo's methods source code
During software maintenance, being able to transform automatically source code can be handy. It can be done easily in Pharo using the built-in code rewriting engine.

For example, let's say that we want to replace the occurences of `#symbolToReplace` symbol with occurences of `#replacedSymbol` in the following method:

```Smalltalk
ExampleCodeTransformation>>#methodToTransform
	1 + 1 = 2
		ifTrue: [ ^ #symbolToReplace ].

	^ #symbolToReplace , #symbolNotToReplace
```

It can be done easily using the following code snippet:

```Smalltalk
compiledMethod := ExampleCodeTransformation >> #methodToTransform.

ast := compiledMethod parseTree.

"Here we select only nodes holding the target symbol and we replace it with a node representing `#replacedSymbol`."
ast allChildren
	select: [ :n | n class = RBLiteralValueNode and: [ n value = #symbolToReplace ] ]
	thenDo: [ :node |
		rewriter := RBParseTreeRewriter new
			replaceTree: node
			withTree: (RBLiteralValueNode value: #replacedSymbol);
			yourself.
		(rewriter executeTree: ast)
			ifTrue: [ node replaceWith: rewriter tree ] ].
		
"Serialize the new AST as Pharo source code."
rewrittenSourceCode := (BIConfigurableFormatter format: ast).

"Recompile the method with transformed source code."
ExampleCodeTransformation compile: rewrittenSourceCode
```

## Browse all available icons
The following code snippet opens an inspector in which the 'Icons' tab allows to browse all icons available in the image.

```
Smalltalk ui icons inspect
```

To get a specific icon, use `#iconNamed:` method as follow:

```
Smalltalk ui icons iconNamed: #arrowUp
```

## Rename programatically methods

In this section we'll present a snippet of code to rename all the methods of a class containing a substring to replace it by another substring. 

```Smalltalk
"Class in which we want to rename the methods"
class := EpApplyPreviewerTest.
"Substring to replace"
from := 'With'.
"Substring to use"
to := 'Without'.

class methods
	select: [ :method | method selector includesSubstring: from ]
	thenDo: [ :method | 
		| permutationMap |
		"We want to keep the arguments in the same order"
		permutationMap := method numArgs = 0 ifTrue: [ #() ] ifFalse: [ (1 to: method numArgs) asArray ].
		
		(RBRenameMethodRefactoring
			renameMethod: method selector
			in: class
			to: (method selector copyReplaceAll: from with: to)
			permutation: permutationMap) execute ]
```

> Be careful, this will also rename the senders of those methods and if you have two methods of the same name in the image, it might badly rename some. Use this only on methods with unique names.
