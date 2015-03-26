# Query Suggestion

## Functionality

This module powers the 'Suggest Query' feature on okcorpus. It takes as input a query and a table, as output it 
produces number of new queries to suggest to the user. The suggestions are made by finding variations of the input 
query that would perform well on the pre-existing entries in the given table.

### Suggestion Types
Two variations of this operation are supported. 'Narrowing' makes the given query more specific (so the suggested 
queries will match less sentences than original query), and 'Broadening' makes the given query broader (so the 
suggested queries will match more sentences than the original query).

### Limitations
Currently only fixed length queries are supported, which means queries that contain control
 characters like '*' or '+' are not supported.
Only one column tables are supported.
Currently only the following changes can be made to a user's query
1. Adding a prefix to the original query (ex. "fat cat" => "the fat cat")
2. Adding a suffix to the original query (ex. "fat cat" => "fat cat ran")
3. Replace a token in the original query (ex. "fat cat" => "fat NN")
4. Replacing a token with a disjunction (ex. "fat cat" => "{fat, lazy} cat")

prefixes, suffixes, or new tokens can be words, clusters, or part of speech tags.

## Implementation
We view the task of suggesting queries as the task of finding 'Query Operators', or functions that takes as 
input a query and outputs a new query, that produces a good query for when applied to the original query. Here
'good' is measured heuristically by evaluating the queries relative to the existing entries in the table.

Finding these operators requires three steps:

1. Subsampling. In this step we try to find a sample of sentences that are:
    1. 'Close' to the original query, in other words sentences that might match a query we might consider suggesting to the user
    2. Has a good mix of positive, negative, and unlabelled examples
2. Generating 'primitive query operations'. This phase generates a collection of query operations that make small, 
incremental changes to the original query (for example replacing a single token in the original query).
3. Search for the best best way to combine the primitive operations into a 'compound operation'. This is done by incrementally building up
larger and larger combinations of primitive operations using a beam search. This requires two subcomponents:
    1. An evaluation function, that produces a score for a given query indicating how good that query is
    2. A 'combiner' that knows how to combined the primitive operations, and which primitive operations are
       compatible and so can be applied to the same query.

When evaluating queries we avoid running them on the index, this would take too much time since 
we need to evaluate thousands of queries.
Instead we keep track, for each primitive query operation, which sentences from our subsample 
would match our query if the primitive operation is applied to it. So, for example, if we are using 
a primitive operation that adds 'the' as a prefix to the original query, and our original query
was 'cat', we record which sentences from our subsample includes the full phrase 'the cat'. For operations
that combine several primitive operations we can then take the intersection of the sentences each
 primitive operation matches to find out what sentences the combined operation would match.
 
An additional concept we need when dealing with 'broadening' queries is that of the edit distance
 between a query and a sentence. The edit distance is defined to be the number of operators
 required to make a particular query match a particular sentence. For example, if our query is 
 "a fast horse", to make the query match the phrase "a slow horse" we will need to edit at least
  one of the symbols in that query, (for example, replace the word "slow" in the query with the 
  word "fast"). The distance from the same query to the sentence "a slow cow" would be 2. During 
  subsampling we record the edit distance from our query of each sentence in our subsample.
  
In addition, we keep track of which operators are capable of reducing the edit 
distance from our query to each sentence. We say an operator that reduces the edit
 distance of our staring query to a given sentence is 'required' for that sentence, otherwise it 
 is not required. To give a more detailed example consider the following scenario:

Our starting query is "a cat"

Our query operation replaces "a" with "DT" (so our new query would "DT cat") 

We have three sentences:
1. "a cat"
2. "the cat"
3. "the dog"
4. "dog dog"

Now the distance from our query to each sentence is:
* 0 for sentence 1
* 1 for sentence 2
* 2 for sentence 3
* 2 for sentence 4
   
And our operator
* matches but is not required for sentence 1
* matches and fills a requirement for sentence 2
* matches and fills a requirement for sentence 3
* does not match sentence 4

It is left to the compound operators to calculate the number of required operators for each 
sentences will be made by applying all their primitive operators, and in particular to avoid 
"double counting" requirements if multiple operators would, for example, edit the same word 
within a query.
 
Implementation-wise we record this information using an map of integers to integers, where the keys 
are integers corresponding to particular sentences and the values are 0, if the given operation 
does not fulfill a requirement for a particular sentence, 1 if it fulfills one requirement, 2 if
 it fulfills 2 and so on. By comparing the number of requirement a operator has fulfilled for a
particular sentence to the edit distance from the starting query to that sentence we can quickly 
deduce whether the starting query would match then sentence if that set of operators were applied
. In the above example this means the operator would be associated with:

IntMap(1 -> 0, 2 -> 1, 3 -> 1)

We use IntMap rather then Seq\[(Int, Int)\] or Map\[Int, Int\] because:
1. It will have a sparse representation, and we expect many of the rules we examine to be sparse
2. It is optimized for merges (unions and intersections), operation we do frequently.

## Code
1. subsample package handles the subsampling phase.
2. primitiveop package defines what a primitive query operation is and how to generate them from a subsample.
3. compoundop package defines how primitive query operations can be combined into compound operations.
4. QueryEvaluator.scala defines the evaluation functions to use.
5. TokenizedQuery.scala breaks queries into sequences of smaller queries as a preprocessing step.
6. QuerySuggester.scala 'glues' these pieces together and implements the actual beam search.