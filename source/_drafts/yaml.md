---
title: YAML
tags: [YAML]
category: [Note, Dev]
---
![YAML Ain't Markup Language](thumbs.png)    

## [YAML](http://yaml.org/)  
> YAML Ain't Markup Language  
YAML is a human friendly data serialization standard for all programming languages.

YAML은 마크업 언어가 아니고, `사람에게 친숙한 데이터 Serializaition 표준`이다.  
마크업 언어는 태그를 이용하여 문서나 데이터의 구조를 표현하는 언어이다. (HTML, XML)   
Serialization(직렬화)은 오브젝트를 프로그램 외부(파일로 쓰거나 네트워크로 전송하거나)에서 사용할 때 사용한다. (Byte Array, [JSON](https://www.json.org/), YAML)  

[Spring](https://spring.io/)에서는 YAML Parser인 [SnakeYAML](https://bitbucket.org/asomov/snakeyaml)이 내장돼있다.  
따라서 YAML 파일을 POJO로 매핑할 수도 있고, Configuration 파일에서도 사용할 수 있다.  
주의사항은 Spring에 내장된 SankeYAML은 YAML 1.1 스펙을 구현한 점이다.  

[Node.js](https://nodejs.org/) 기반의 블로그 프레임워크인 [Hexo](https://hexo.io/)에서도
[JS-YAML](https://github.com/nodeca/js-yaml)라는 YAML Parser를 이용해 Configuration을 설정하고 있다.  
JS-YAML은 Python의 YAML Parser인 [PyYAML](https://pyyaml.org/)을 포팅하면서 YAML 1.1 스펙을 지원했는데, 현재는 YAML 1.2 스펙까지 구현했다.  

### YAML vs. JSON
> JSON’s foremost design goal is simplicity and universality.
Thus, JSON is trivial to generate and parse, at the cost of reduced human readability.  
In contrast, YAML’s foremost design goals are human readability and support for serializing arbitrary native data structures.
Thus, YAML allows for extremely readable files, but is more complex to generate and parse.  

JSON의 최우선 설계 목표는 간편성과 보편성이다. 따라서 JSON은 가독성을 떨어트리는 대신에 생성 및 파싱이 용이하다.  
반면에 YAML의 최우선 설계 목표는 가독성과 데이터 구조 Serialization이다. 따라서 YAML은 사람이 읽기 쉬운 반면에 생성 및 파싱이 좀 더 복잡하다.

> YAML can therefore be viewed as a natural superset of JSON.  
This is also the case in practice; every JSON file is also a valid YAML file.  
This makes it easy to migrate from JSON to YAML if/when the additional features are required.  

YAML을 JSON의 Superset으로 볼 수도 있다.  
모든 JSON 파일은 유효한 YAML 파일이다.  
따라서 JSON에서 YAML로 마이그레이션 하기가 용이하다.  

#### 가독성
YAML 홈페이지 조차도 아래와 같이 유효한 YAML 문법으로 이루어져있다.  
```yaml
%YAML 1.2
---
Projects:
  C/C++ Libraries:
  - libyaml            # "C" Fast YAML 1.1
  - libcyaml           # YAML de/serialization of C data structures (using libyaml)
  - Syck               # (dated) "C" YAML 1.0
  - yaml-cpp           # C++ YAML 1.2 implementation
```

과연 JSON 형태로 YAML 홈페이지를 가독성 좋게 표현할 수 있었을까...?
```json
{
  "Projects": {
    "C/C++ Libraries": ["libyaml", "libcyaml", "Syck", "yaml-cpp"]
  }
}
```

#### 퍼포먼스
> JSON It also uses a lowest common denominator information model, ensuring any JSON data can be easily processed by every modern programming environment.
YAML ventures beyond the lowest common denominator data types, requiring more complex processing when crossing between different programming environments.  

JSON 보다 [YAML의 모델](http://yaml.org/spec/1.2/spec.html#id2763452)이 좀 더 복잡하기 때문에 파싱하고 생성하는데 YAML이 더 느리다.  
속도도 느린데 직접 파싱을 한다고 생각해도 JSON이 훨씬 간단하다.

#### 용도
* 대부분의 웹 기술([AJAX](https://developer.mozilla.org/en-US/docs/Web/Guide/AJAX) 등등)이 YAML을 지원하지 않기 때문에 대부분 JSON을 많이 사용하고,
reference가 있는 YAML은 복잡한 object 구조를 표현하는데 적합해서, 오프라인에서 data serialization을 위해 더 적합하다.     
* 파이썬 커뮤니티에서는 {} 보다는 파이썬 문법과 비슷한 indent로 indicate level을 구분하는 YAML을 더 선호하고,
자바스크립트 진영에서는 별도의 파서가 필요없고, Javascript Object와 구조가 유사한 JSON을 선호하는 편이다.  

#### 그 외.    
* JSON은 주석이 없다.  
* YAML은 한 파일에 여러 Object를 표현할 수 있다.
* > JSON's RFC4627 requires that mappings keys merely “SHOULD” be unique, while YAML insists they “MUST” be.
  Technically, YAML therefore complies with the JSON spec, choosing to treat duplicates as an error.
  In practice, since JSON is silent on the semantics of such duplicates, the only portable JSON files are those with unique keys, which are therefore valid YAML files.
    
  JSON은 Key의 중복을 Warning으로 표시하지만, YAML은 허용하지 않는다.  
  
### YAML vs. XML
> two languages may actually compete in several application domains, there is no direct correlation between them.
  YAML is primarily a data serialization language.
  XML was designed to be backwards compatible with the Standard Generalized Markup Language (SGML), which was designed to support structured documentation.
  
XML과 YAML은 전혀 연관이 없다.  
XML은 구조화된 문서를 위해 설계된 언어이고, YAML은 data serialization을 위해 설계된 언어이다.   

### [Version 1.2 Specification](http://yaml.org/spec/1.2/spec.html)
1.1에서 1.2로 바뀐 주요 사항은 JSON의 Superset처럼 동작하기 위해서 많은 사항들이 바뀌었다.  
> All other characters, including the form feed (#x0C), are considered to be non-break characters. 
  Note that these include the non-ASCII line breaks: next line (#x85), line separator (#x2028) and paragraph separator (#x2029).
  YAML version 1.1 did support the above non-ASCII line break characters; however, JSON does not.
  Hence, to ensure JSON compatibility, YAML treats them as non-break characters as of version 1.2. In theory this would cause incompatibility with version 1.1;
  in practice these characters were rarely (if ever) used.
  YAML 1.2 processors parsing a version 1.1 document should therefore treat these line breaks as non-break characters, with an appropriate warning.

따라서 JSON Syntax를 YAML 1.2에서는 거의 완벽하게 지원한다.  

#### 목적
> YAML is easily readable by humans.  

YAML은 사람이 읽기 쉽다.

> YAML data is portable between programming languages.  

YAML 데이터는 프로그래밍 언어 간에 이동이 가능하다.

> YAML matches the native data structures of agile languages.  

YAML은 agile 언어의 native data structure를 매치한다.  

> YAML’s core type system is based on the requirements of agile languages such as Perl, Python, and Ruby.
YAML directly supports both collections (mappings, sequences) and scalars.
Support for these common types enables programmers to use their language’s native data structures for YAML manipulation

Perl, Python, Ruby와 같은 agile language에 존재하는 scalar, collection 타입을 지원해서,  
프로그래머가 native data structure를 조작하기가 용이하다.   

> YAML has a consistent model to support generic tools.
  
YAML은 tool들을 지원하는 [일관적인 모델](http://yaml.org/spec/1.2/spec.html#id2763452)이 있다.  

> YAML supports one-pass processing.
  
YAML은 one-pass 처리를 지원한다.(??)  

> YAML is expressive and extensible.
  
YAML은 표현력과 확장성이 뛰어나다.  

> YAML is easy to implement and use.

YAML은 구현 및 사용이 쉽다.  

#### Prior Art(선행 기술??)
HTML의 EOL(End of Line)에 영감을 받아서 sinlge line break는 single space로 해석되고,
empty line은 line break character를 뜻한다.  

YAML 스펙 문서는 [BNF 표기법](/2018/08/15/bnf/)을 사용하고 있기 때문에 BNF 표기법에 대해 조금은 알고 있어야 읽기가 수월하다.

#### Node Kinds
* Scalar  
> The content of a scalar node is an opaque datum that can be presented as a series of zero or more Unicode characters.

스칼라 노드는 0개 이상의 유니코드 문자이다.  
일반적인 primitive type을 생각하면 편하다.  

```yaml
a: b
```
여기서 a와 b 모두 scalar 노드이다.

* Sequence
> The content of a sequence node is an ordered series of zero or more nodes.
  In particular, a sequence may contain the same node more than once.
  It could even contain itself (directly or indirectly).

시퀀스 노드는 0개 이상의 순서가 보장된 노드이다.  
또한 똑같은 노드를 여러 번 포함할 수 있다.  
심지어 자기 자신을 포함할 수도 있다.  
일반적인 list를 생각하면 편하다.  

```yaml
- a
- b
```
여기서 `-a -b`를 통틀어 시퀀스 노드라고 부른다.

* Mapping
> The content of a mapping node is an unordered set of key: value node pairs, with the restriction that each of the keys is unique.
  YAML places no further restrictions on the nodes. 
  In particular, keys may be arbitrary nodes, the same node may be used as the value of several key: value pairs, and a mapping could even contain itself as a key or a value (directly or indirectly).

매핑 노드는 순서가 보장되지 않은 key:value 쌍의 집합이다.  
key는 고유하다는 제한을 빼고는 다른 제한은 있지 않다.  
1. 키는 임의의 노드일 수도 있고 자기 자신을 포함할 수 있다.  
-> 키에 문자열을 대부분 넣는데 문자열은 scalar 노드이니까 임의의 노드일 수도 있다고 표현한 게 아닐까?
2. 동일한 노드를 여러 key:value 쌍의 값으로 사용할 수 있고,  
-> key는 고유한데, value는 고유하지 않다는 걸 표현한 게 아닐까?  
3. 자기 자신을 포함할 수 있다. 

```yaml
a: b
c:
  - e
  - f
```
`a: b`, `c: -e -f` 이 두 개 모두 매핑 노드이다.

> When appropriate, it is convenient to consider sequences and mappings together, as collections.

sequence와 mapping 두 개를 합쳐 collections로 퉁친다는 소리 같다.

#### Block Styles
> In YAML block styles, structure is determined by indentation.
  In general, indentation is defined as a zero or more space characters at the start of a line.
  To maintain portability, tab characters must not be used in indentation, since different systems treat tabs differently.

구조를 표현할 때 [indentation](#)를 사용한다.  
라인의 시작점에 존재하는 0개 이상의 [white space character](#)에 의해 정의된다.
하지만 이식성을 고려해서 Tab키는 사용하면 안 된다.  
시스템들 사이에서 Tab키를 취급하는 방법이 서로 다르기 때문이다.

#### Indicator
> In YAML block styles, structure is determined by indentation.
  In general, indentation is defined as a zero or more space characters at the start of a line.
  To maintain portability, tab characters must not be used in indentation, since different systems treat tabs differently.

Block Style의 구조는 들여쓰기에 의해 결정된다.  
라인의 시작점에 존재하는 0개 이상의 [white space character](#)에 의해 정의된다.
하지만 이식성을 고려해서 Tab키는 사용하면 안 된다.  
시스템들 사이에서 Tab키를 취급하는 방법이 서로 다르기 때문이다.

```
s-indent(n) ::= s-space × n
```

> A block style construct is terminated when encountering a line which is less indented than the construct.
  Each node must be indented further than its parent node.
  All sibling nodes must use the exact same indentation level. 
  However the content of each sibling node may be further indented independently.

Block Style의 구조는 이전 라인보다 더 적은 들여쓰기가 있는 라인을 만나면 끝난다.  
각각의 노드는 부모 노드보다 더 많은 들여쓰기를 써야만 한다.  
모든 형제 노드는 같은 들여쓰기 레벨을 써야만 한다.  
하지만 각 형제 노드의 내용들은 독립적이다.

##### White Space Characters
```
s-space ::= #x20 /* SP */
s-tab   ::= #x9  /* TAB */
s-white ::= s-space | s-tab
```
스페이스와 탭 문자만 white space character로 인식한다.






