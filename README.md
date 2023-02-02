# 'Natural Speech' Expression

The repository contains a TCL-procedure-based implementation of the 'Natural Speech' expression language and its interpreter, which I have recently developed. The original implementation was utilized as a user-defined search filter within a self-hosted administration page of an F5 LTM, but the included interpreter can be used to evaluate search expressions on any type of input data.

The major design goal of the 'Natural Speech' expression language is to enable humans to write complex search expressions without having to learn a complicated search language syntax. People may already struggle to articulate what they want to search for, and the addition of complex search expression languages like RegEx can increase this difficulty and require specific education and extensive practice.

The 'Natural Speech' expression language addresses this issue by adopting the grammar and syntax of human natural language.

***To be fair:** I have to admit that RegEx is a Swiss Army knife with virtually unlimited capabilities due to its generic syntax. This versatility can make it seem paradoxical that it can have so many features yet still be user-friendly. While RegEx is incredibly powerful, its complexity can make it overly challenging for relatively simple string comparison tasks.*

# Integration of the 'Natural Speech' Procedure

The TCL procedure provided accepts an input string and a 'Natural Speech' expression of your choice, and then returns the evaluation result to an [if] statement to initiate actions.

```
when HTTP_REQUEST {

	if { [call ::ask_natural_speech_if [HTTP::path] {

                                              equals /

                                      leading with /appfolder/ 
                             and contains /somefolder/ or /anotherfolder/ 
                                  and /deepfolder1/ or /deepfolder2/
                                     and trailing with / or .php 
                                            but exclude 
                                        /evilsomething.php 
                                 or contains /admin/ nor /upload/

                                 starts with /resourcefolder/ 
                               and ends with .jpg or .css or .js

	}] } then {		

		pool mypool

	} else {

		HTTP::respond 403 content "Access Denied" "Content-Type" "text/html"	

	}

}
```
***Note:** I've decided to parse an ordinary HTTP path in this sample integration code because HTTP is a well-known data structure. Don't consider this proof-of-concept as a recommendation to actually use it this way, but feel free to do so if you see an advantage for your deployment.*  

# Explanation of the 'Natural Speech' expression language

The 'Natural Speech' expression interpreter reads a list of words or escaped "s t r i n g" sequences in sequential order from start to end. It analyzes each item and check if an operational keyword is used to guide the logical expression. All other non-operational words are utilized as opaque string elements to define the conditions.

The interpreter reads and employs logical operators (e.g. AND) and positional operators (e.g. EQUALS) to create a precise search expression. A single 'Natural Speech' expression may include one or multiple individual rule chains, which will be combined into a final expression once all rule chains have been processed.

## About Rule Chains

A rule chain is a series of consecutive operators and string elements that follow the rules you already know from speaking your own natural language. The only difference is probably that the interpreter does not utilize punctuation between two or more sentences (aka. rule chains) and does not incorporate grammatical articles.

Instead of using punctuation, the interpreter splits the input expression into different rule chains using two simple conventions:

***Rule 1: A string element followed by another string element starts a new rule chain.***

For example:

```
'STRING-1 and STRING-2 or STRING-3 STRING-4'
    Chains => 'STRING-1 and STRING-2 or STRING-3' 'STRING-4' = 2 rule chains
```
***Rule 2: The pure NOT operator always starts a new negative chain.***

For example:

```
'STRING-1 and STRING-2 but not STRING-3 not STRING-5'
    Chains => 'STRING-1 and STRING-2 but not STRING-3' 'not STRING-4' = 2 rule chains
```
The interpreter understands either positive or negative rule chains (also known as white/black lists). A positive rule chain will return **TRUE** if the input matches certain conditions, and a negative rule chain will return **TRUE** if the input does not match certain conditions.

## Merge of Rule Chains

After parsing the 'Natural Speech' expression, the individual rule chains are combined into a complete set of rules by adding either AND or OR operators between them, based on the type of the rule chain.

The interpreter will always merge two or more positive rule chains with OR operators, so that if any of them match, the result will be set to **TRUE**. On the other hand, two or more negative rule chains will be merged with AND operators, so that none of them is allowed to match to set the result to **TRUE**.

If a 'Natural Speech' expression contains both positive and negative rule chains, the already combined positive and negative rule chains are getting merged with AND operators. This means that at least one positive rule chain must match and no negative rule chain is allowed to match, in order to set the result to **TRUE**.

```
'STRING-1 and STRING-2 STRING-3 not STRING-4 not STRING-5'
    Positive Chains => 'STRING-1 and STRING-2' 'STRING-3'
    Negative Chains => 'not STRING-4' 'not STRING-5'
    Combined Result => ((STRING-1 and STRING-2) or (STRING-3)) and ( not ( STRING-4 ) and not ( STRING-5 ))
```
## About Operator Aliases

The "Natural Speech" interpreter allows you to customize its search operator aliases. This customization enables you to specify search conditions using your own language and phrases, making the use of the interpreter more intuitive and easier.

Multiple aliases or phrases can be assigned to a single search operator. The default alias list provided with the iRule includes the following mappings:

```
{ not } { - }
{ and  not } { ! } {not  and } { ! } { but  not } { ! } { but  exempt } { ! } { also  exempt } { ! } { but  deny } { ! } { but  allow } { ! } { also  not } { ! } { also  deny } { ! } { also  allow }  { ! } { nand }  { ! }
{ nor } { | } { as  well  as } { | } { or } { | }
{ in  combination  with } { + } { also } { + } { and } { + } { & } { + }
{ starts_with } { < } { starts  with } { < } { starting  with } { < } { leading  with } { < }
{ ends_with } { > } { ends  with } { > } { ending  with } { > } { trailing  with } { > }
{ contains } { <> } { containing } { <> }
{ equals } { = } { eq } { = }
```

These aliases will convert your human-readable 'Natural Speech' expression into the search operators used by the 'Natural Speech' interpreter.

```
'leading with STRING-1 or STRING-2 in combination with STRING-3 but exclude contains STRING-4 as well as trailing with STRING-5'
    Alias Tagging => '[leading with] STRING-1 [or] STRING-2 [in combination with] STRING-3 [but exclude] [contains] STRING-4 [as well as] [trailing with] STRING-5'
    Alias Resolved => '< STRING-1 | STRING-2 + STRING-3 ! <> STRING-4 | > STRING-5'
```
```
'starts with STRING-1 or STRING-2 and STRING-3 and not contains STRING-4 or ends with STRING-5'
    Alias Tagging => '[starts with] STRING-1 [or] STRING-2 [and] STRING-3 [and not] [contains] STRING-4 [or] [ends with] STRING-5'
    Alias Resolved => '< STRING-1 | STRING-2 + STRING-3 ! <> STRING-4 | > STRING-5'
```

***Note:** The alias list provides a human-readable overlay. You can bypass the alias translation and specify your expression using native operators, but this may require some training to get used to it..*

The customization of aliases is a major strength of the 'Natural Speech' expression language, but it also introduces a potential issue. The more keywords you add, the more likely it is that a string element used to define a condition will be accidentally translated into an operator. The interpreter is robust and will not translate string elements that partially match an operator alias, but it will not understand your intended expression if you use a reserved alias to define a string element.

To explicitly indicate that a word should not be treated as an operator, you can enclose it in quotes.

For example:

```
'starts with and or not'
    Chains => '' because only operators are used
```
```
'starts with "and" or "not"'
    Chains => '< "and" | "not"'
```

However, enclosing words in quotes can detract from the natural language feel of the 'Natural Speech' expression language. To minimize the risk of accidental operator translation and maintain the natural language feel, it's best to keep the list of aliases concise, context-specific, and small.

## About Operators

Once the 'Natural Speech' aliases are resolved, the interpreter recognizes four logical and four positional operators.

Positional Operators:

- The positional operator < (aka STARTS_WITH) matches a condition at the start of an input string.
- The positional operator > (aka ENDS_WITH) matches a condition at the end of an input string.
- The positional operator <> (aka CONTAINS) matches a condition anywhere within an input string.
- The positional operator = (aka EQUALS) matches a condition to the entire input string.

Logical Operators:

- The logical operator - (aka. NOT) starts a new negativ rule chain.
- The logical operator + (aka. AND) requires two conditions to match at the same time.
- The logical operator | (aka. OR) allows to specifying two replacable conditions at the same time.
- The logical operator ! (aka. NAND) adds exceptions to the main conditions of a rule chain.

### Positional Operator usage

The positional operator functions like a register. When the interpreter initializes, it will default to CONTAINS. Each change to the register will remain in effect until the next change. You can chain as many positional operators in a row (only the last one will be used), and use them before or after other logical operators as desired. The tracking of the positional operator register occurs independently of the other operators and is evaluated when a string element is consumed by the interpreter.

### Logical NOT Operator usage

The logical NOT operator ends any ongoing rule chain (if already started) and begins a new negative rule chain. You can switch between positive and negative rule chains as often as you like within a single 'Natural Speech' expression, and include multiple negative rule chains if needed.

```
'STRING-1 not STRING-2 STRING-3 not STRING-4 and STRING-5'
    Positive Chains => 'STRING-1' 'STRING-3'
    Nagative Chains => 'not STRING-2' 'not STRING-4 and STRING-5'
    Combined Result => ((STRING-1) | (STRING-3)) + ! ( STRING-2 ) + ! ( STRING-4 and STRING-5)
```
### Logical AND / OR Operator usage

When dealing with logical AND and OR operators in a single rule chain, it's important to understand which operator has implicit precedence. The 'Natural Speech' expression language, like any spoken language, does not support any form of parentheses groupings (e.g. 'a or ( b and c )') to give higher precedence to certain parts of the search string.

During development, the different approaches were tested and it has turned out to give higher precedence to logical OR expressions over logical AND expressions.

This approach allows for expressions like the one shown below:

For example:

```
'string-1 or string-2 and string-3 or string-4'
    Implicit : ( string-1 or string-2 ) and ( string-3 or string-4 )
    Result : ( string-1 and string-3 ) | ( string-1 and string-4 ) | ( string-2 and string-3 ) | ( string-2 and string-4 )
```
The decision not to give the logical AND operator precedence was mainly because splitting a single rule chain into multiple rule chains already adds such functionality, so it would be redundant to assign precedence to the logical AND operator.

For example:

```
'string-1 and string-2 string-3 and string-4'
   Chains : 'string-1 and string-2' 'string-3 and string-4'
   Result : ( string-1 and string-2 ) | ( string-3 and string-4 )
```

The lack of parentheses groupings can be further addressed by using logical NAND operations.

### Logical NAND Operator usage

Regardless of the type of rule chain, you can express one or multiple logical NAND exemption in the currently processed rule chain to limit its scope.

```
'STRING-1 but not STRING-2 not STRING-3 but exempt STRING-4 and STRING-5 also exempt STRING-6 and STRING-7'
    Main Rule 1 => 'contains STRING-1'
    NAND Rule 1 => 'contains not STRING-2'
    Main Rule 2 => 'contains not STRING-3'
    NAND Rule 2 => 'contains STRING-4 and STRING-5' 'starts with STRING-6 and STRING-7'
    Combined Rule => '( STRING-1 + ! STRING-2 ) + ! ( STRING-3 + ! ( STRING-4 + STRING-5 ) + ! ( STRING-6 and STRING-7 ))
```

Imagine parents asking their children: "This huge chocolate bar OR this tiny candy AND that long-lasting chewing gum?"

Does it actually mean that the children can choose between the huge chocolate bar or the tiny candy and then take the long-lasting chewing gum in addition? This would probably be a jackpot for every child, but from the parents' perspective, the word "huge" was more or less a hidden warning to not be greedy. But such hidden warnings can be pretty hard to understand for children, leading to arguments and eventually tears.

NAND operations can help avoid misunderstandings without relying on parentheses groupings:

```
'Candy and "Chewing Gum" but not Chocolate' 'Chocolate but not Candy or "Chewing Gum"'
```

This NAND example uses two independent rule chains to give clear, selectable options. Each chain specifies a strict main condition and then exempts the invalid combinations.

***Note:** It's important to note that logical NAND operations should only exempt a subset of the conditions specified in the main part of the rule chain. If you allow the string 'Hello World' and then add the exemption for the letter 'W', it would render the rule chain useless. However, if you allow the letter 'W' but exempt the string 'Hello World', it would make sense in terms of NAND operations. The same strategy also applies when combining positive and negative rule chains: if a negative rule chain denies everything, a positive rule chain can't allow anything.*

# Explanation of the "Natural Speech" intepreter

Some interesting facts: It took me longer to develop a deep understanding of how humans define expressions and the basic rules of "Natural Speech" expressions than it did to write the iRule code. The explanation of the "Natural Speech" expression in this readme is also much longer than the code itself.

However, it still took me a few attempts to implement it correctly. My initial approaches were based on lots of handcrafted string comparisons to analyze the input data based on the provided 'Natural Speech' expression, but they resulted in poor performance. I finally ended up with an implementation where the 'Natural Speech' interpreter itself does not need to parse any input data. The 'Natural Speech' interpreter acts as an indermediate layer between the user defined 'Natural Speech' expression and a regular [if...then...else] syntax. The interpreter reads the provided 'Natural Speech' expressions and then translate the contained instructions to become a traditional TCL expression. Once the TCL expression is completely translated, the interpreter will pass the expression to the [if...then...else] syntax in an TCL variable and execute the evaluation.

The rather complex [HTTP::path] filtering expression I showed at the beginning of the readme:

```
                                              equals /

                                      leading with /appfolder/ 
                             and contains /somefolder/ or /anotherfolder/ 
                                  and /deepfolder1/ or /deepfolder2/
                                     and trailing with / or .php 
                                            but exclude 
                                        /evilsomething.php 
                                 or contains /admin/ nor /upload/

                                  starts with /resourcefolder/ 
                                and ends with .jpg or .css or .js
```
Outputs the TCL expression below.
```
(
	(
		(
			(
				(
					$input_string equals "/"
				)
			)
		)
	)
	||
	(
		(
			(
				(
					$input_string starts_with "/appfolder/"
				)
			)
			&&
			(
				(
					$input_string contains "/somefolder/"
				)
				||
				(
					$input_string contains "/anotherfolder/"
				)
			)
			&&
			(
				(
					$input_string contains "/deepfolder1/"
				)
				||
				(
					$input_string contains "/deepfolder2/"
				)
			)
			&&
			(
				(
					$input_string ends_with "/"
				)
				||
				(
					$input_string ends_with ".php"
				)
			)
		)
		&& !
		(
			(
				(
					$input_string ends_with "/evilsomething.php"
				)
				||
				(
					$input_string contains "/admin/"
				)
				||
				(
					$input_string contains "/upload/"
				)
			)
		)
	)
	||
	(
		(
			(
				(
					$input_string starts_with "/resourcefolder/"
				)
			)
			&&
			(
				(
					$input_string ends_with ".jpg"
				)
				||
				(
					$input_string ends_with ".css"
				)
				||
				(
					$input_string ends_with ".js"
				)
			)
		)
	)
)
```
## Translation of 'Natural Speech' to TCL expression

The resulting TCL expression is organized in different layers to support a fully streamed translation without knowing what the next action will be. The translation process requires only a look-behind operation that correlates the last seen logical operation for the next string element appended to a given rule chain. This look-behind operation is maintained via a variable that tracks the instruction set used for a single rule chain.

The translation process uses repeatable TCL expression building blocks, designed to speed up the TCL expression creation. You can take a look at the snippets below to get an idea of how countless positive and negative rules relying on arbitrary AND/OR/NAND operations can be fitted into a single TCL expression by chaining those building blocks in a simple foreach loop.

Positive Rule Chain Start:
```
	(
		(
			(
				(
					$input_string $position "$element"
				)
```				
Negative Rule Chain Start:
```
	! (
		(
			(
				(
					$input_string $position "$element"
				)
```
Logical OR Operator:
```
				||
				(
					$input_string $position "$element"
				)
```
Logical AND Operator:
```
			)
			&&
			(
				(
					$input_string $position "$element"
				)
```
Logical NAND Operator:
```
			)
		)
		&& !
		(
			(
				(
					$input_string $position "$element"
				)
```
Rule Chain Finish:
```
			)
		)
	)
```
Joining Positive Rules:
```
	||
```
Joining Negative Rules:
```
	&&
```
Combining Final TCL Expression:
```
(
  Combined Positive Rule Chains
)
&&
(
  Combined Negative Rule Chains
)
```
## Caching of 'Natural Speech' expressions

The fully translated TCL expression is stored in the static::* namespace within an array variable and then passed to the [if...then...else] syntax for evaluation.

The array variable acts as a TMM-specific memory cache, allowing subsequent executions of the same 'Natural Speech' expression to reuse the previously translated TCL expression and also its internal byte-code-optimized interpretation computed during the initial [if...then...else] execution.

Compared to traditional TCL expression evaluation, the overhead for subsequent executions of the same 'Natural Speech' expression is only a few CPU clicks (aka. microseconds) and is mainly caused by using the TCL procedure as a wrapper.

Compared to using RegEx, the initial 'Natural Speech' expression evaluation is not much slower and, depending on the complexity of the expression, may already be much faster after a few consecutive executions.

## Sanitization of the 'Natural Speech' expression

The sanitization of the 'Natural Speech' expression is crucial, as it may be defined by untrusted end-users, depending on the use case.

The interpreter relies on a controlled double-substitution of a just-in-time compiled TCL expression (sort of JIT). Without proper sanitization, the TCL interpreter may fail to process the expression, process rules incorrectly, or even allow untrusted end-users to execute additional TCL commands that could crash or reprogram TMM.

The 'Natural Speech' procedure includes a thorough sanitization process for the 'Natural Speech' expression, which aims to fix broken expression syntax for stable operations and \ escapes dangerous char sequences, so that user-provided content is interpreted as pure string values during the double-substitution process.

## Code documentation

The provided 'Natural Speech' procedure includes many line-by-line comments to explain the TCL syntax used, even for non-developers. If you still have questions, don't hesitate to contact me.

# Extensibility of the 'Natural Speech' expression language

The 'Natural Speech' expression language provides a solid framework for translating human natural language into existing expression languages. You may extend the language by adding new positional operators (e.g. support for glob matches as a positional operator) or by implementing additional registers to customize the translation process as desired (e.g. adding a new register to switch between different input strings or to translate the search string into two or more separate output expressions).

The possibilities to extend the 'Natural Speech' expression language and its interpreter are virtually limitless...

# Greetings

I'd like to invite peers to review the concepts of the 'Natural Speech' expression language and the accompanying procedure containing the 'Natural Speech' interpreter.

Please feel free to contact me (kw @ itacs.de) if you have any questions or if you need adjustments for your own implementation.

Cheers, Kai
