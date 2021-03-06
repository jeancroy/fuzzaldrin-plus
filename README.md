## What is fuzzaldrin-plus?

- A fuzzy search / highlight that specialize for programmer text editor. It tries to provide intuitive result by recognizing patterns that people use while searching.

- A rewrite of the fuzzaldrin library. API is backward compatible with some extra options. Tuning has been done from report usage of the Atom text editor.

- At this point in time, it may either be merged back into fuzzaldrin or lives as a forked library, we'll see.


## What Problem are we trying to solve?

### Score how matched characters relate to one another.

- One of the most often reported issues is not being able to find an exact match.
- A great source of questionable results come from scattered character, spread seemingly randomly in the string.

We plan to address those issues by scoring runs of consecutive characters. In that scheme, an exact match will be a special case where the run is 100% of the query length.

In original fuzzaldrin, candidate length was used as a proxy for match quality. This work reasonably well when subject is a single word, but break when subject contain multiple words, for example, see:

- **Core**
- **Co**nt**r**oll**e**r
- Extention**Core**

In `Core` vs. `Controller` size is a good indicator of quality, but not so much in `Controller` vs. `ExtentionCore`. This situation happens because match compactness matters more than haystack size. Match compactness is the principle behind the scoring of the *Selecta* project.


#### Run length / consecutive

So far the examples can be handled by an `indexOf` call. However, there are times where a single query can target multiple parts of a candidate.

For example when candidate contains multiple words
- `git push` vs. `Git Plus: Push`
- `email handler` vs. `email/handler.py`

Another example is to jump over common strings.
- `toLowerCase`
- `toLocaleString`
- `toLocalLowerCase`

We could use a query like `tololo`  to select the third option of these.


### Select character based on score.

The previous algorithm always selects the first available matching character (leftmost alignment). Only after selection, it will try to identify how to score that character. The problem then is that the most interesting instance of a character is not necessarily on the left.

For example on query `itc`, we should match
-  **I**mportance**T**able**C**trl.

Instead leftmost aligement miss the acronym pattern:
- **I**mpor**t**an**c**eTableCtrl.

For query `core` against `controller_core` leftmost alignment miss the consecutive run:
- **co**nt**r**oll**e**r_core

To handle this, we propose to embed the pattern detection (consecutive and more) inside an optimal alignment scheme. Imagine you have an algorithm that allows you to recognize objects in images; it would make little sense to run it exclusively on the top left corner.


### Prevent Accidental acronym.

Fuzzladrin handles acronym by giving a large bonus on character matches that start words.  Currently, a start-of-word bonus matches almost as much as three proper-case character.

For query `install` should result be in this order ?
- F**in**d & Replace **S**elec**t** **All**
- Application: **Install**

In that example, we have '**S**elect **A**ll' boost the score of the first candidate because we score two word-starts while we only score one for 'install'.

For query `git push`, should we order result in that order ?
- "**Git** **P**l**u**s: **S**tage **H**unk"
- "**Git** Plus: **Push**"

What about the fact we match three start-of-words in `Plus Stage Hunk`? PSH is very close to '**p**u**sh**' (And `Plus` contains `u`).

That kind of question arises even more often when we use optimal selection because the algorithm will lock on those extra acronym points.

What we propose in this project is that start-of-words characters should only have a moderate advantage by themselves. Instead, they form strong score by making an acronym pattern with other start-of-words characters.

For example with query `push`:
- against `Plus: Stage Hunk`:  we have `P + u + SH`  grouped as 1, 1, 2
- against `push`:  we have a  single group of 4.
- The substring wins for having the largest group

For example with query `psh`:
- against `Plus: Stage Hunk`: we have `PSH` so a single group of 3
- against `push`: we have `p + sh` so grouped as 1, 2
- The acronym wins for having the largest group.


This way we can score both substring and acronym match using the structure of the match. We'll refine the definition of consecutive acronym later.


### Score the context of the match.

Some people proposed to give a perfect score to exact case-sensitive matches. This proposition can be understood because exact matches and  case-sensitivity are two area where fuzzaldrin is not great.

However should 'sw**itc**h.css' be an exact match for `itc`?
Even when we have **I**mportance**T**able**C**trl available?

Few people will argue against `diag` preferring `diagnostic` to `Diagnostics`.

However should `install` prefer "Un**install**" over "**Install**" ? Or should it be the other way around?  In this case, we have to consider the relative priority of case-sensitivity and start-of-word.

Exact matches are used with enougth frequency that we should not only ensure they win against approximate matches but also ensure to rank quality properly amongst them.

### Manage the multiples balances of path scoring.

We want to prefer match toward the start of the string.
Except we also want to prefer match in the filename (which happens near the end of the string)

We want to prefer shorter and shallower path.
Except we also want to retrieve some deeper files when filename is clearly better.

We want to prefer matches in the filename
Except when the query describes a full path much better than an approximate file name. (Let's consider query `model user` vs `models/user.rb` or `moderator_column_users.rb`)


## Proposed Scoring Rules


### 1. Characters are chosen by their ability to form a pattern with others.

Patterns can be composed of
 - consecutive letters of the subject,
 - sequential letters in the Acronym of the subject.

Start-of-words (acronym) characters are special in that they can either forms pattern with the rest of the word or with other acronym characters.

- Pattern based scoring replaces a situation where acronyms characters have a large bonus by themselves. Now the bonus is still large but conditional to being part of some pattern.

- CamelCase and snake_case acronym are treated exactly the same. They will, however, get different score for matching uppercase/lowercase query


### 2. Primary score attribute is pattern length.

- A pattern that span 100% of the query length is called an exact match.
     - There is such a thing as an acronym exact match.

- Because every candidate is expected to match all of query larger group should win against multiple smaller one.
    - The rank of a candidate is the length of it's largest pattern.
    - When all patterns of a first candidate are larger than patterns in a second one, the candidate with the highest rank is said to be dominant.
        - 1 group of 6 >  2 group of 3 > 3 group of 2.

    - When some groups are larger, and some are smaller, the highest rank match is said to be semi-dominant.
         - Let's consider a first candidate grouped as 4+2 vs. a second candidate grouped as 3+3.
              - The first group of 4 wins against the first group of 3.
              - However, the group of 2 loose against the second group of 3.
             - In this case, we'll consider some extra information.

### 3. Secondary score attribute is the quality of matches.

- Match quality is made of proper casing and context score
    - The main role of match quality is to order candidate of the same rank.
    - When match is semi-dominant match quality can overcome a small rank difference.

- Context score considers where does the match occurs in the subject.
    - Full-word > Start-of-word > End-of-word > Middle-of-word
    - On that list, Acronym pattern score at Start-of-word level. (That is just bellow full-word)

- Score for a proper case query has both gradual and absolute components.
   - The less error, the better
   - 100% Case Error will be called wrong-case, for example matching a `CamelCase` acronym using lowercase query `cc`.  
   - Exactly 0 error is called CaseSentitive or ExactCase match.
     - CaseSentitive matches have a special bonus that is smaller than start-of-word bonus but greater than the end-of-word bonus.
     - This scoring scheme allows a start-of-word case-sensitive match to overcome a full-word wrong-case match.
     - It also allows to select between a lowercase consecutive and CamelCase acronym using case of query.
     - To answer the question asked in the introduction, "Installed" win over "Uninstall" because start-of-word > Exact Case.

- **Q:** Why can't you simply add extra length for some bonus. For example, score a start-of-word match as if it had an extra character.
   - **A:** We cannot do that on partial matches because then the optimal alignment algorithm will be happy to split word and collect start-of-word bonus like stamps. (See accidental acronym)

- **Q:** Why do you add extra length on exact matches?
   - **A:** First, once you have matched everything, there's no danger of splitting the query. Then,  that bonus exists to ensure exact matches will bubble up in the firsts results, despite longer/deeper path. If, after more test and tuning, we realize it's not needed, we'll be happy to remove it, the fewer corner cases, the merrier.

- **Q:** Why are you using lowercase to detect CamelCase?
  - **A** CamelCase are detected as a switch from lowercase to UPPERCASE. Defining UPPERCASE as not-lowercase, allow case-invariants characters to count as lowercase.   For example `Git Push` the `P` of push will be recognised as CamelCase because we consider `<space>` as lowercase. 

### 4. Tertiary score attributes are subject size, match position and directory depth

- Mostly there to help order match of the same rank and match quality, unless the difference in tertiary attributes is large.
     -(Proper definition of large is to be determined using real life example)

- In term of the relative importance of effects it should rank start-of-string > string size > directory depth.

## Score Range

- **Score is 0 if and only if there is no match.**

- Otherwise, the score is a strictly positive integer.

- The maximum range is `score(query,query)` whatever that number is. A longer query will have a greater maximal score.

- Score exist mainly to for relative order with other scores of the same query and to implements scoring rule described above. 

- Score have a high dynamic range and consider a lot of information. Equality is unlikely. For that reason, **multiplicative bonuses should be preferred over additive ones**.

-------------

### Acronym Prefix
More detail on acronym match


An acronym prefix is a group of characters that are consecutive in the query and sequential in the acronym of the subject. That group starts at the first character of the query and end at the first character of the query, not in the acronym. If there's no missed character, then we have an acronym exact match (100% of query is sequential in the acronym)


**For example if we match `ssrb` against `Set Syntax Ruby` we'll score it like so**

````
  012345678901234
 "Set Syntax Ruby"
 "000 SSR0b0 0000"
````

- Acronym scored as three consecutive character at start-of-word + an isolated letter.
- Here we have a wrong-case match. "SSRb" or "SSRB" would have case-sensitive points on the acronym pattern (case of isolated letter is not important)
- Position of the equivalent consecutive match is the average position of acronym characters.
- For scoring, we use the size of the original candidate.


**Another example is matching `gaa` against `Git Plus: Add All` we'll score it like so**

````
  01234567890123456
 "Git Plus: Add All"
 "000000 GAA00 0000"
````

- here we conveniently allow to skip the `P` of `Plus`.


**Then what about something like "git aa" ?**

This is a current limitation. We do not support acronym pattern outside of the prefix. Mostly for performance reason.
Acronym outside of the acronym prefix will have some bonus, scoring between isolated character and 2 consecutive.
There are multiple thing we can improve if one day we implement a proper multiple word query support, and this is one of them.

### Optional characters

Legacy fuzzaldrin had some support for optional characters (Mostly space, see `SpaceRegEx`). Because the scoring does not support errors, the optional character was simply removed from the query.

With this PR, optimal alignment algorithm supports an unlimited number of errors. The strict matching requirement is handled by a separate method `isMatch`. The optional character implementation is done by building a subset of the query containing only non-optional characters (`coreQuery`) and passing that to `isMatch`.

This new way of doing thing means that while some characters are optional, candidates that match those characters have a better score. What this allow is to add characters to the optional list without compromising ranking.

Optional character contains space, but also `-` and `_` because multiple specs require that we should treat them as space. Also `\` and `:` are also optional to support searching a file using the PHP or Ruby name-space. Finally `/` is optional to mirror `\` and support a better workflow in a multi-OS environment.

Finally option `allowErrors` would make any character optional. Expected effect of that options would be some forgiveness on the spelling at the price of a slower match.


### Path Scoring

- Score for a given path is computed from the score of the fullpath and score of the filename. For low directory depth, the influence of both is about equal. But, for deeper directory, there is less retrieval effect (importance of basename) 

- The full path is penalized twice for size. Once for its own size, then a second time for the size of the basename. Extra basename penalty is dampened a bit.

- The basename is scored as if `allowErrors` was set to true. (Full-path must still pass `isMatch` test).  This choice is made to support query such as `model user` against path `model/user`. Previously, the basename score would be 0 because it would not find `model` inside basename `user`. Variable `queryHasSlashes` partially addressed this issue, but was inconsistent with usage of `<space>` as folder separator

- When query has slashes (`path.sep`) the last or last few folder from the path are promoted to the basename. (as many folder from the path as folder in the query)

-------------

## Algorithm (Optimal alignment)

### LCS: Dynamic programming Table
Let's compare  A:`surgery` and B:`gsurvey`.
To do so we can try to match every letter of A against every letter of B.

This problem can be solved using a score matrix.
- The match starts at [0,0] trying to compare the first letter of each.
- The match end at [m,n] comparing the last letter of each.

At each position [i,j] the best move can be one of the 3 options.

- match `A[i]` with `B[j]` (move diagonal, add 1 to score)
- skip `A[i]` (move left, copy score)
- skip `B[j]` (move down, copy score)

We do not know which one of these 3 is the best move until we reach the end, so we record the score of the best move so far. The last cell contains the score of the best alignment. If we want to output that alignment we need to rebuild it backward from the last cell.

````
    s u r g e r y
 g [0,0,0,1,1,1,1]  : best move is to align `g` of *g*survey with `g` of  sur*g*ery, score 1
 s [1,1,1,1,1,1,1]  : we can align `s`, but doing so invalidate `g`. Both score 1, we cannot decide
 u [1,2,2,2,2,2,2]  : if we align s, we can also align u, we have a winner
 r [1,2,3,3,3,3,3]  : we can align `r`
 v [1,2,3,3,3,3,3]  : nothing we can do with that `v`, score stay the same
 e [1,2,3,3,4,4,4]  : we can align `e`  (we skipped `g` the same way we skipped `v`)
 y [1,2,3,3,4,4,5]  : align y (we skipped `r` )
````

**Best alignment is**

````
gsur-ve-y
-|||--|-|
-surg-ery
````

For those familiar with code diff, this is essentially the same problem. Except, in this case, we the do the alignment of characters in a word and a diff performs alignment of lines in a file. Characters present in the second word but not in the first counts as additions; characters present only in the first word are deletions and characters present in both are matches - like unchanged lines in a diff.

To get that alignment, we start from the last character and trace back the best option.  The pattern to looks for an **alignment** is the corner increase (diagonal+1 is greater than left or up.)

````
4,4   3,3   2,2    1,1    0,0
4,5   3,4   2,3    1,2    0,1
````

- (There are an implicit row and column of 0 before the matrix)

The pattern to look for to **move left** is:

````
3,3
4,4
````

The pattern to look for to **move up** is:

````
3,4
3,4
````

We try to resolve equality the following way:

````
3,3
3,3
````

1. Prefer moving UP: toward the start of the candidate. This strategy ensures we highlight toward the start of string instead of the end when all else is equal.
2. If not available, prefer moving LEFT (optional character)
3. Only accept alignment DIAG when it is the absolute best option.




### Algorithm Conclusion

The LCS algorithm allows to detect which character of the query are common to both words while being in proper order. (For example g is common to both word but discarded because out of order.)

LCS is not immediately useful for fuzzaldrin needs. Because fuzzaldrin require ALL characters of the query to be in subject to have a score greater than 0, LCS for all positive candidates would be the length of the query.

However, the dynamic programming table used to solve LCS is very useful to our need. The ability to select the best path and skip that `g` even if it is present in both query and candidate is the key to improves over left-most alignment. All we need for this to works is a bit more detail in score than 0 or 1. 

### Similarity score

Matching character does not have to be binary. Case sensitive match can still prefer proper case, same goes with accents. A diff tools can decide a line has been modified, instead of registering an addition and a deletion. A handwriting recognition tool can decide `a` and `o` are somewhat more similar to each other than they are to `w`, and so on.

We use character similarity as a way to build and score patterns. That is, we consider that character are similar from their own quality ( such as case) as well of being part of a similar neighborhood (consecutive letters or acronyms)

There are some rules that limit our scoring ability (for example we cannot go back in time and correct the score based on future choice) but overall that scheme is very flexible.

### Where is the matrix?

While the programming table describes computation, we do not need to store the whole matrix when we only output the score. Fundamentally when computing a score, we only need 3other previously computed cell: UP, LEFT and DIAG.

Suppose we process the cell [3,5]

20, 21, 22, 23, 24, 25, 26, *27, 28, 29*   
*30, 31, 32, 33, 34,* **35**, 36, 37, 38, 39  

To build that score we only need values 24(DIAG), 25(UP), 34(LEFT).
So instead of a whole matrix we can keep only the two current lines.

Furthermore, anything on the left of 24 on the first line is not needed anymore. Also, anything to the right of 35 on the second line has not yet been computed. So we can build a more compact structure using one composite row + one diagonal.

score_diag =  24    
score_row = 30, 31, 32, 33, 34, 25, 26, 27, 28, 29   

#### Preparing next value

Once we have computed the value of the cell [3,5], we can insert that value into the structure, taking care of saving next diagonal before overwriting it.

diag =  25    
row = 30, 31, 32, 33, 34, **35**, 26, 27, 28, 29   

To compute value of cell [3,6] we take 
- UP value (26) from the row.
- DIAG value, from the diag register. 
- LEFT value from the previously computed value: 35

### Initial values

Before entering the matching process, the row is initialized with 0. Before scoring each row, the LEFT and DIAG register are reset to 0.

That strategy has the effect of placing a virtual row and column of 0 before the matrix. Moreover, it allows to deal with boundary condition without any special case.

### Memory management

We set up the row vector with the size of the query. Using a full matrix, scoring a query of size 5 against a path of size 100, would require a 500 cells. Instead, we use a 5 item row + some registers. This should ease memory management pressure.

Each character of the query manages its best score. More precisely, each cell `row[j]` manage the best score so far of matching `query[0..j]` against candidate[0..i]. 

### Consecutive Score (Neighbourhood) Matrix.

We cache the consecutive score in a virtual matrix following the same composite row scheme that we do with score values.

In `fuzzaldrin.score` The candidate entirely determines the Neighbourhood quality. It is not affected by which character has been chosen. In highlight, (`fuzzaldrin.match`) we further refine the formula to make the consecutive bonus conditional to not breaking the consecutive chain:

For example query `abcdz` vs. subject `abcdzbcdz`. Between `abcd` and `bcdz`, `abcd` wins for being sooner in the string. Now between the two `z`, the first one is isolated and the second one is part of a rank 4 group. However given that `bcd` are matched sooner, the second `z` is an isolated match, so the first `z` wins.

-------------

## Performance

Let's consider the following autocomplete scenario.
- Symbol bank has 1000 items.
- The user receives about 5 suggestion for its query.
- Of those 5, 1 is a exact case-sensitive match.
- That particular user almost always wants that case sensitive match.

Should we optimize for case sensitive `indexOf` before trying other things? Our answer to that question is no. 

Case sensitive exact match are valuable because they are rare. Even if the user tries to get them, for each one of those we have to reject 995 entry and deal with 4 other kinds of matches.

This is our first principle for optimization: **Most of the haystack is not the needle**. Because rejection of candidate happens often, we should be very good at doing that. 

Failing a test for case-sensitive `indexOf` tell us exactly nothing for case-insensitive `indexOf`, or acronyms, or even scattered letters.
That test is too specific. To reject match efficiently, we should aim for the lowest common denominator: scattered case-insensitive match.

This is exactly the purpose of `isMatch`.

### Most of the haystack is not the needle

We just have shown how that sentence applies at the candidate level, but it is also at the character level. 

**Let's consider this line: `if (subject_lw[i] == query_lw[j])`**
This test is for match points (or hits). It refers to the `diag+1` in the algorithm description, with the `+1` being refined to handle the differents levels of character and neighborhood similarity.


**How often is that condition true ?**

Let's consider an alphabet that contain 26 lowercase letters, 10 numbers, a few symbols ` _!?=<>`. That is a 40+ symbol alphabet. Under a uniform usage model of those symbols, we have the hit condition occurs about 2.5% of the time (1/40). If we suppose only 10-20 of those characters are popular, the hit rate is about 5-10%.

This means we'll try to minimize the number of operation that happens outside of math points. In that context, increasing the cost of a hit, while decreasing the cost of non-hits looks like a possibly worthwhile proposition. 

A canonical example of this is that, instead of testing each character against the list of separators, setting a flag for next character being a start-of-word, we first confirm a match then look behind for separator. This characterization work is sometimes repeated more than once, but so far this scheme benchmarked better than alternatives we have tried to avoid doing extra work.

Having work concentrated at hit points is also a natural fit to our logic, the most expensive part being to determine how to score similarity between characters (including context similarity). However, it also means we'll want to have some control over the number of positive hits we'll compute - that is the purpose of missed hit optimisation.


### What about a stack of needles?

To the extent the user is searching for a specific resource, this should be uncommon.

It can still happen in some situation such as:
 - Search is carried as user type (the query is not intentional)
 - The intentional query is not fully typed, match-all is a temporary step.

One way to deal with that is not to use the full matching algorithm when we can deal with something simpler. This is what we have done while searching for `indexOf` instance. 

One special note: Acronym still have to be checked even if we have an exact match: for example query `su` against `StatusUrl`. As an exact match it is poor: 'Statu**sU**rl' is a middle of word match and have the wrong case. However as an acronym it is great: '**S**tatus**U**rl'. That motivated us to create the specialized `scoreAcronyms`.

What is nice is that while `scoreAcronyms` was created to speed up exact matches search, it also provided very valuable information for accuracy. It later became a corner stone in the processing of accidental acronym. 

The result is that for exact matches and exact acronym matches we bypass the optimal alignment algorithm, giving very fast results.
We still have to deal with fuzzier stacks of needles and the next two optimization address this.

### Hit Miss Optimization.

A hit occurs when character of query is also in the subject.
 - Every (i,j) such that subject[i] == query[j], in lowercase.

A missed hit occurs when a hit does not improve the score.

To guarantee optimal alignment, every hit has to be considered.
However when candidate are long (deep path) & query contains common use character, for example, vowels , we can spend a huge amount of time scoring accidental hits.

So we use the number of missed hit as a heuristic for current score that are unlikely to improve. Let's score `itc` vs `ImportanceTableControl`

- `I` of `Importance`: First occurrence, improve over none.
- `t` of `Importance`: First occurrence, improve over none.
- `c` of `Importance`: First occurrence, improve over none.
- `T` of `Table` : Acronym match, improve over an isolated middle of word.
- `C` of `Control` : Acronym match, improve over an isolated middle of word.
- `t` of `Control`: no improvement over acronym `T`: first hit miss.

- After a certain threshold of missed hit we can consider it is unlikely the score will improve by much.
- Despite above example hit miss optimization do not affect scoring of exact match (sub-string or acronym)
- There are some legitimate use for hit miss, for example while scoring query `Mississippi` each positive match for `s` or `i` may trigger up to 3 hit miss on the other occurrence of that letter in query.

- For that reason, we propose counting consecutive hit miss and having a maximum of one hit miss per character of the subject.

**Q:** Does this grantee improvement over leftmost alignment?
**A:** It'll often be the case but no guarantee on pathological matches.
 For example, in query `abcde` against candidate '**abc**abcabcabcabcabcabczde' we may trigger the miss count before matching `de`. It'll still be registered as a match and probably a good one with `abc` at the start, `de` will be scored as optional characters not present.

Candidate  'abcabcabcabcabcabc**abcde**' will not have any problem because it does not affect exact match.

A real world example is searching `index` in the benchmark. Where `i`, `n`, `d`, `e` exist scattered in folder name, but x exist in the extension `.txt`. However, the whole point of this project is to prefer structured match to scattered one so this might not be a problem.

### High Positive count mitigation
**[option `maxInners`, disabled by default]**

A lot of the speed of this PR come from the idea that rejection happens often, and we need to be very efficient on them to offset slower higher quality match. Unfortunately, some query will match against almost everything.

- Fast short-circuit path for exact substring acronym help a lot.
- Missed hit heuristic also help a lot for general purpose match.

However, we may still be too slow for interactive time query on large data set. This is why `maxInners` option is provided.

This is the maximum number of positive candidate we collect before sorting and returning the list.

The realization is that a query that match everything on a 50K item data set is unlikely to show anything useful to the user above the fold (say in the first 15 results).

So then the priority is to detect such case of low quality (low discrimination power) query and report fast to the user so user can refine its query.

A `maxInners` size of about 20% of the list works well. It is not needed on a smaller list.

### Active Region Optimization

Before the first occurrence of the first char of query in the subject, or after the last occurrence of the last char of query in the subject it is impossible to make a match. So we'll trim the subject to that active region. The search for those boundaries is linear while the optimal alignment algorithm is quadratic, so it is an improvement, however, little or large we move.

### Benchmark
- All test compare this PR to previous version (legacy)

- The first test `index` is a typical use case, 10% positive, 1/3 of positive are exact matches.
  - We are about 2x faster

- Second test `indx` remove exact matches. Just under 2x faster

- Third test `walkdr`, 1% positive, mostly testing `isMatch()`, above 2x faster.

- Fourth test `node`, exact match, 98% positive, bit under 2x faster.

- Test 5 `nm`, exact acronym match, 98% positive, about 10% slower.

- Test 6 `nodemodules` is special in that it use a string that score on almost every candidate, often multiple time per candidate and individuals characters are popular. It also avoid exact match speed-up. About 2x slower, but unlikely to happens in real life. `maxInners` mitigation cover that case.


````
Filtering 66672 entries for 'index' took 62ms for 6168 results (~10% of results are positive, mix exact & fuzzy)
Filtering 66672 entries for 'index' took 120ms for 6168 results (~10% of results are positive, Legacy method)
======
Filtering 66672 entries for 'indx' took 69ms for 6192 results (~10% of results are positive, Fuzzy match)
Filtering 66672 entries for 'indx' took 126ms for 6192 results (~10% of results are positive, Fuzzy match, Legacy)
======
Filtering 66672 entries for 'walkdr' took 30ms for 504 results (~1% of results are positive, fuzzy)
Filtering 66672 entries for 'walkdr' took 70ms for 504 results (~1% of results are positive, Legacy method)
======
Filtering 66672 entries for 'node' took 112ms for 65136 results (~98% of results are positive, mostly Exact match)
Filtering 66672 entries for 'node' took 213ms for 65136 results (~98% of results are positive, mostly Exact match, Legacy method)
======
Filtering 66672 entries for 'nm' took 60ms for 65208 results (~98% of results are positive, Acronym match)
Filtering 66672 entries for 'nm' took 56ms for 65208 results (~98% of results are positive, Acronym match, Legacy method)
======
Filtering 66672 entries for 'nodemodules' took 602ms for 65124 results (~98% positive + Fuzzy match, [Worst case scenario])
Filtering 66672 entries for 'nodemodules' took 123ms for 13334 results (~98% positive + Fuzzy match, [Mitigation])
Filtering 66672 entries for 'nodemodules' took 295ms for 65124 results (Legacy)
````

**Q:** My results are not as good.
**A:** Run the benchmark a few time, it looks like some optimization kick in later. (Or CPU on energy efficient device might need to warm up before some optimization are activated)



## Prior Art

[Chrome FilePathScore](https://chromium.googlesource.com/chromium/blink/+/master/Source/devtools/front_end/sources/FilePathScoreFunction.js#70)

[Textmate ranker](https://github.com/textmate/textmate/blob/master/Frameworks/text/src/ranker.cc#L46)

[VIM Command-T ](https://github.com/wincent/command-t/blob/master/ruby/command-t/match.c#L22)

[Selecta](https://github.com/garybernhardt/selecta/blob/master/selecta#L415)

[PeepOpen](https://github.com/topfunky/PeepOpen/blob/master/Classes/Models/FuzzyRecord.rb)

[flx](https://github.com/lewang/flx)


## List of addressed issues

### Exact match vs directory depth.

https://github.com/atom/fuzzaldrin/issues/18
-> actionsServiceSpec

https://github.com/atom/atom/issues/7783
-> usa_spec

### Start of string VS directory depth

https://github.com/atom/fuzzy-finder/issues/57#issuecomment-133531653
-> notification

https://github.com/atom/fuzzy-finder/issues/21#issuecomment-48795958
-> video backbone

### Folder / file query

https://github.com/atom/fuzzy-finder/issues/21#issue-29106280
-> src/app vs destroy_discard

https://github.com/atom/fuzzy-finder/issues/21#issuecomment-46920333
-> email handler

https://github.com/substantial/atomfiles/issues/43
-> model user

### Spread/group vs directory depth

https://github.com/atom/fuzzy-finder/issues/21#issuecomment-138664303
-> controller core

### Initialism

https://github.com/atom/fuzzy-finder/issues/57#issue-42120886
-> itc switch / ImportanceTableCtrl

https://github.com/atom/fuzzy-finder/issues/57#issuecomment-95623924
-> application controller

https://github.com/atom/fuzzaldrin/issues/21
-> fft vs FilterFactorTests

### Accidental Acronym

https://github.com/atom/command-palette/issues/28
-> Install / Find Select All

https://github.com/atom/fuzzaldrin/issues/20#issue-93279352
-> Git Plus Stage Hunk / Git Plus Push


### Case sensitivity

https://github.com/atom/autocomplete-plus/issues/42
-> downloadThread / DownloadTask

https://github.com/atom/fuzzaldrin/issues/17
-> diagnostics / Diagnostic


### Optional Characters

https://github.com/atom/fuzzy-finder/issues/91
https://github.com/atom/fuzzaldrin/issues/24

-> PHP Namespaces, let "\" match "/"
-> (would be nice for config file in mixed OS environment too)

https://github.com/atom/fuzzy-finder/pull/51

-> Ruby Namespaces, let "::" match "/"

https://github.com/atom/fuzzy-finder/issues/10
-> SpaceRegex, let " " match "/"
-> was already implemented, posted here to show parallel.


### Suggestions

https://github.com/atom/fuzzy-finder/issues/21#issue-29106280
-> we implement suggestion of score based on run length
-> todo allows fuzzaldrin to support external knowledge.
