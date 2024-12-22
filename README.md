
呃....4年前开了一个坑，准备写一套完整介绍TS 原理的文章。坑很大，要慢慢填，今天就来填一个把。


本节主要介绍语法增量解析。


## 什么是增量解析


增量解析的意思是，如果我们直接从源码解析成语法树，叫做全量解析。


语法树是由很多个节点对象组成的，比较吃内存。


当用户修改源码后（无论修改哪里，包括插入一个空格），我们都需要重新解析文件，生成新的语法树。


如果每次都全量解析，那意味着释放之前的所有节点，然后重新创建所有节点。


但实际上，用户每次只会修改一部分内容，而整个语法树的大部分节点都不会发生变化。


如果解析时，发现节点没有变化，就可以复用之前的节点对象，只重新创建变化的节点，这个过程叫增量解析。


增量解析是一种性能优化，它不是必须的。


## 实现增量解析的基本原理


假如我们修改了函数中某行代码的内容，从原则上说，这个函数之前的节点都是不变的。


函数之后的节点大概率不变，但也有小概率会变，比如我们插入了“}”，导致函数的范围发生变化，或者插入“\`”，导致后面的内容都变成字符串的一部分了。


看上去好像很复杂，但 TS 采用了一个折中的做法，大幅降低了复杂度。


TS 是以语句为单位进行复用的，即每条语句或者完全复用，或者完全不复用，即使单条语句里面存在可复用的部分子节点。（这种说法其实并不准确，但为了方便理解，可以先这么认为）


核心逻辑为：


1\. 当在 pos 位置解析一条语句前，TS 先检测该位置是否存在可复用的旧节点，如果不存在当然就无法增量解析，就转成常规解析。


2\. 如果存在旧节点，则检查该旧节点所在区域是否发生变化，如果发生变化，则放弃复用，转为常规解析。


3\. 如果没有发生变化，那这条语句就直接解析完毕，然后从这行语句的 end 位置继续解析下一条语句，重复前面的步骤。


 


## SyntaxCursor


代码位于 parser.ts




```
export interface SyntaxCursor {
     currentNode(position: number): Node;
}
```



```
SyntaxCursor 用于从原始的语法树中查找指定位置对应的可复用的旧节点。
```



```
    export function createSyntaxCursor(sourceFile: SourceFile): SyntaxCursor {
        let currentArray: NodeArray = sourceFile.statements;
        let currentArrayIndex = 0;

        Debug.assert(currentArrayIndex < currentArray.length);
        let current = currentArray[currentArrayIndex];
        let lastQueriedPosition = InvalidPosition.Value;

        return {
            currentNode(position: number) {
                // 函数内基于一个事实做了一个性能优化　　　　　　　　　 // 就是解析时，position 会逐步变大，因此查找的时候，不需要每次都重头查找，而是记住上一次查找的位置　　　　　　　　　 // 下次查找就从上次的位置继续查找，这样找起来更快
                if (position !== lastQueriedPosition) {
                    if (current && current.end === position && currentArrayIndex < (currentArray.length - 1)) {
                        currentArrayIndex++;
                        current = currentArray[currentArrayIndex];
                    }

                    // 如果上次的位置和要查找的位置不匹配，就重头查找。
                    if (!current || current.pos !== position) {
                        findHighestListElementThatStartsAtPosition(position);
                    }
                }

                // 记住本次查找的位置，加速下次查找
                lastQueriedPosition = position;

                Debug.assert(!current || current.pos === position);
                return current;
            },
        };

        // 标准的深度优先搜索算法，找到就近的节点
        function findHighestListElementThatStartsAtPosition(position: number) {
            currentArray = undefined!;
            currentArrayIndex = InvalidPosition.Value;
            current = undefined!;

            forEachChild(sourceFile, visitNode, visitArray);
            return;

            function visitNode(node: Node) {
                if (position >= node.pos && position < node.end) {
                    forEachChild(node, visitNode, visitArray);

                    return true;
                }

                // position wasn't in this node, have to keep searching.
                return false;
            }

            function visitArray(array: NodeArray) {
                if (position >= array.pos && position < array.end) {
                    for (let i = 0; i < array.length; i++) {
                        const child = array[i];
                        if (child) {
                            if (child.pos === position) {
                                currentArray = array;
                                currentArrayIndex = i;
                                current = child;
                                return true;
                            }
                            else {
                                if (child.pos < position && position < child.end) {
                                    forEachChild(child, visitNode, visitArray);
                                    return true;
                                }
                            }
                        }
                    }
                }

                return false;
            }
        }
    }
```


## 解析列表


每个列表（包括语句块的语句列表）都是使用 parseList 解析的。每个元素都是通过 parseListElement 解析。




```
function parseList(kind: ParsingContext, parseElement: () => T): NodeArray {
        const saveParsingContext = parsingContext;
        parsingContext |= 1 << kind;
        const list = [];
        const listPos = getNodePos();

        while (!isListTerminator(kind)) {
            if (isListElement(kind, /*inErrorRecovery*/ false)) {
                list.push(parseListElement(kind, parseElement));

                continue;
            }

            if (abortParsingListOrMoveToNextToken(kind)) {
                break;
            }
        }

        parsingContext = saveParsingContext;
        return createNodeArray(list, listPos);
    }
```


parseListElement 中会先检测可复用的节点，如果存在，就复用并解析下一个元素，否则正常解析当前元素。




```
    function parseListElement(parsingContext: ParsingContext, parseElement: () => T): T {
        const node = currentNode(parsingContext);
        if (node) {
            return consumeNode(node) as T;
        }

        return parseElement();
    }
```


currentNode 负责返回可复用的节点，除了基于 syntaxCursor 查找，还加了一些额外的限制，防止某些特殊情况会复用。


```
function currentNode(parsingContext: ParsingContext, pos?: number): Node | undefined {
       
        if (!syntaxCursor || !isReusableParsingContext(parsingContext) || parseErrorBeforeNextFinishedNode) {
            return undefined;
        }

        const node = syntaxCursor.currentNode(pos ?? scanner.getTokenFullStart());

        // 存在语法错误的节点不能复用，因为我们需要重新解析，重新报错。
        if (nodeIsMissing(node) || intersectsIncrementalChange(node) || containsParseError(node)) {
            return undefined;
        }

        const nodeContextFlags = node.flags & NodeFlags.ContextFlags;
        if (nodeContextFlags !== contextFlags) {
            return undefined;
        }

        // 有些节点不能复用，因为存在一定场景导致复用出错
        if (!canReuseNode(node, parsingContext)) {
            return undefined;
        }

        if (canHaveJSDoc(node) && node.jsDoc?.jsDocCache) {
         
            node.jsDoc.jsDocCache = undefined;
        }

        return node;
    }
```


当节点被复用后，使用 consumeNode 设置下次扫描的位置。




```
function consumeNode(node: Node) {
        scanner.resetTokenState(node.end);
        nextToken();
        return node;
    }
```


## 不能复用的场景


有些场景复用是有问题的，（很多场景都是社区通过 Issue 给 TS 报的 BUG，然后修复的）。


比如泛型：




```
var a = b < c, d, e
```


从复用角度，这是一个列表，列表项分别为:


* ```
a = b < c
```
* d
* e


理论在 e 后面插入任何字符，都不影响前面的节点，但存在一个特例，就是"\>"




```
var a = b
```


当 \<\> 成对，它变成了泛型。这会导致需要重新解析整个语句。


TS 的做法并不是检测是否插入了“\>”，而是因为存在整个特例，就完全不复用变量列表的任何节点，即使多数情况复用的安全的。


毕竟增量解析只是一种性能优化，没有也不是不能用。


完整的检测特殊情况的逻辑在 canReuseNode，因为比较琐碎，且逻辑都比较简单，这里就不贴了。


## 结论


经过增量解析后，部分节点会被重新使用。


从算法中可以得出，如果子节点被修改了，那父节点一定也会被修改。而源文件本身在每次增量解析时，都会被重新创建。



 


 本博客参考[飞数机场](https://ze16.com)。转载请注明出处！
