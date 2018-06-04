# A grammar of tables

*The Grammar of Graphics* introduced data analysts to an abstract and generalized "grammar" to translate raw data into data visualizations. **ggplot2** implemented those ideas in R. **dplyr** provides a similar grammar for data transformations. These packages have showcased the value of an abstract grammar as an approach to translating a data object into some kind of output and, in particular, have highlighted the value of working with "tidy" data frames as a fundamental unit of data analysis. But lacking from the tidy verse is a grammar of *tables*, the common, rectangular data structure that summarizes a dataset into cells representing subsets of the data, arranged as rows, columns, and facets. This document outlines a grammar of tables and provides a provisional R implementation.

## Why a grammar of tables?

**ggplot2** is successful in part because it separates the content of visualization being created ("geoms" and "aesthetics") and the styling of that visualization ("themes"). A grammar of tables would do the same for tabular presentations of data and it is something that has already been discussed in [various](http://stats.stackexchange.com/questions/3542/what-is-a-good-resource-on-table-design) [places](https://github.com/yihui/knitr/issues/53). This would represent a significant advance over existing approaches to tabular construction which closely couple content and styling (e.g., `xtable()`) and styling/rendering tools (e.g., `stargazer()`) which must be customized to work with every possible input format. R's tabular functions are also needlessly complex, such as the common expression `prop.table(table(x))` which summarizes a *table* rather than conveniently summarizing the original data.

Other statistical software also merges content and styling inflexibly, such as the `tab var1, summarize(var2)` and `tabstat var2, s(min max mean median) by(var1)` approaches in Stata or the tabular summarization tools in SPSS that provide limited access to the abstract table content independent of the output tables.

Creating a single, abstract table representation makes it possible to render *any* table without regard for the form of the original data structure from which the table's content is generated. Further separating themes and format-specific rendering should simplify cross-format consistency of representation.

A grammar of tables further allows a computational approach to generating any arbitrary table by specifying how cells should be arranged according to what subsets of the original data they represent, and how those subsets should be set as rows, columns, and/or facets alongside table metadata. This ability to permute table arrangements is one of the reasons that "pivot tables" are seen as useful and that interactive approach to table design might be used to create a table arrangement that can later be used computationally.

## What is a table?

A **table** is an *arrangement* of *summaries* of a dataset into rows, columns, and facets (or subtables). A table contains the following components:

 1. A *cell* or set of *cells*, each of which represents a (sub)set of the data specified by one or more grouping factor(s) analyzed by the summarizing function. A summarizing function (or *summarizer*) is any function that evaluates to a single output (e.g., a numerical value or character string) for any vector of input values. The simplest summarizer is the identity function; more common examples would be the mean or variance function, or a function that returns a character string combining the two (e.g., `mean (variance)`). Contingency tables typically utilize only one summarizer (the count or length function), but the general class of tables include tables with multiple summarizers (e.g., contingency tables sometimes contain "marginals"; a regression output table that reports coefficient estimates plus summary statistics for the model).
 
 2. An *arrangement*, which describes the positions of cells within the table into rows, columns, and facets (or subtables).
 
 3. A set of *metadata* that are not cells or necessarily summaries of data, such as a title, subtitle, notes, annotations, hidden identifiers (e.g., HTML or LaTeX IDs), etc.
 
 4. A *theme*, which describes the overall visual aesthetics of the table.
 
 5. A *rendering*, which translates an abstract table representation into a particular output file format, such as markdown, LaTeX, HTML, docx, rtf, Excel, etc.
 
Each of the components 2-5 (arrangement, metadata, theme, and rendering) can be combined in multiple ways to express the informational content of the table (i.e., its cells) for some particular output. This means that a *table* takes two forms: (1) its abstract form containing only the table *content* and (2) its finalized form containing the content expressed through an arrangement, theme, and rendering (that may display metadata).

### Summarizers

The contents of a cell are determined by a *summarizer* (a function, often a summary statistic) applied to each subset of the data defined by the grouping factors that compose the rows, columns, and facets. An implementation of the grammar of tables does not necessarily need to implement summarizers because the necessarily aggregations can be performed using existing functionality (e.g., `aggregate()`, **dplyr** syntax, etc.).

That said, implementing some summarizers within a grammar of tables implementation may be useful, as it enables the combination of cells that reflect different summarizers within a single table. For example, a contingency table may contain subgroup counts as well as "row totals" and/or "column totals" that summarize the data at a higher level of aggregation. Combining these cell types requires knowing how the various cells summarize the original data in order to be able to arrange them systematically.

The separation of summarization from arrangement can be highlighted in a simple example. This tabular representation of the `Titanic` survivor data:

```R
ftable(Titanic, row.vars = 1:3)
##                    Survived  No Yes
## Class Sex    Age                   
## 1st   Male   Child            0   5
##              Adult          118  57
##       Female Child            0   1
##              Adult            4 140
## 2nd   Male   Child            0  11
##              Adult          154  14
##       Female Child            0  13
##              Adult           13  80
## 3rd   Male   Child           35  13
##              Adult          387  75
##       Female Child           17  14
##              Adult           89  76
## Crew  Male   Child            0   0
##              Adult          670 192
##       Female Child            0   0
##              Adult            3  20
```

is simply a particular representation of a `sum()` aggregation of the data by `Class`, `Sex`, `Age`, and `Survived`:

```R
aggregate(Freq ~ ., data = as.data.frame(Titanic), FUN = sum)
##    Class    Sex   Age Survived Freq
## 1    1st   Male Child       No    0
## 2    2nd   Male Child       No    0
## 3    3rd   Male Child       No   35
## 4   Crew   Male Child       No    0
## 5    1st Female Child       No    0
## 6    2nd Female Child       No    0
## 7    3rd Female Child       No   17
## 8   Crew Female Child       No    0
## 9    1st   Male Adult       No  118
## 10   2nd   Male Adult       No  154
## 11   3rd   Male Adult       No  387
## 12  Crew   Male Adult       No  670
## 13   1st Female Adult       No    4
## 14   2nd Female Adult       No   13
## 15   3rd Female Adult       No   89
## 16  Crew Female Adult       No    3
## 17   1st   Male Child      Yes    5
## 18   2nd   Male Child      Yes   11
## 19   3rd   Male Child      Yes   13
## 20  Crew   Male Child      Yes    0
## 21   1st Female Child      Yes    1
## 22   2nd Female Child      Yes   13
## 23   3rd Female Child      Yes   14
## 24  Crew Female Child      Yes    0
## 25   1st   Male Adult      Yes   57
## 26   2nd   Male Adult      Yes   14
## 27   3rd   Male Adult      Yes   75
## 28  Crew   Male Adult      Yes  192
## 29   1st Female Adult      Yes  140
## 30   2nd Female Adult      Yes   80
## 31   3rd Female Adult      Yes   76
## 32  Crew Female Adult      Yes   20
```

Other tables with the same arrangement as the first might be created from an alternative summarizer, such as taking the mean age of subset, proportion of women, etc.


### Arrangements

Consider the following abstract table that contains four cells, which summarize a dataset by two grouping factors (`V1` and `V2`):

| V1  | V2  | Summarizer | Value |
| --- | --- | ---------- | ----- |
|  A  |  C  | Count      |   1   |
|  A  |  D  | Count      |   2   |
|  B  |  C  | Count      |   3   |
|  B  |  D  | Count      |   4   |

This example highlights what distinguishes a table from a general dataset: it contains only one "value" column. This column contains the cells of the table and other columns contain the group factors or other information (e.g., what summarizing function generated that value).

The table's *content* can be transformed in any of the following arrangements (plus numerous others) which hold the content (and theme and rendering) constant:

```
| B | C | 3 |
| B | D | 4 |
| A | C | 1 |
| A | D | 2 |

| A | D | 2 |
| A | C | 1 |
| B | D | 4 |
| B | C | 3 |

| B | D | 4 |
| B | C | 3 |
| A | D | 2 |
| A | C | 1 |

| A | C | 1 |
| B | C | 3 |
| A | D | 2 |
| B | D | 4 |

| A | D | 2 |
| B | D | 4 |
| A | C | 1 |
| B | C | 3 |

| A | A | B | B |
| C | D | C | D |
| 1 | 2 | 3 | 4 |

| B | B | A | A |
| C | D | C | D |
| 3 | 4 | 1 | 2 |

| A | B | A | B |
| C | C | D | D |
| 1 | 3 | 2 | 4 |

| A | B | A | B |
| D | D | C | C |
| 2 | 4 | 1 | 3 |

|   | C | D |
| A | 1 | 2 | 
| B | 3 | 4 |

|   | C | D |
| B | 3 | 4 | 
| A | 1 | 2 |

|   | D | C |
| A | 2 | 1 | 
| B | 4 | 3 |

|   | D | C |
| B | 4 | 3 | 
| A | 2 | 1 |
```

### Themes

The theme contains the general aesthetics of the final output table. Ideally, themes are independent of the renderer, such that any theme can be rendered similarly in any markup language. The simplest theme is the empty theme, which contains no aesthetic information, allowing users to express a theme in the output format (e.g., CSS, etc.).

Table themes are complex and something to be thought about later on. A useful reference may be: https://medium.com/mission-log/design-better-data-tables-430a30a00d8c#.cprfjagxl

### Rendering

The *renderer* of the table convey the cell content, arrangement, metadata, and theme in a markup language. Commonly needed renderers for R would be:

 - markdown
 - latex
 - html
 - OpenDocument
 - docx
 - rtf
 - Excel
 - others?

Implementing a renderer to work with this grammatical approach to tables will require a set of renderers that respect an general grammar of arrangement and the format-independent expression of a theme. Existing rendering functions - xtable, kable, rtf, stargazer, etc. etc. - are probably not going to be adequate.

# Relevant existing packages and functions

```
# - renderers
#   - xtable
#   - rtf
#   - knitr::kable()
#   - formatttable
#   - knitLatex
#   - htmlTable
#   - psytabs
#   - SortableHTMLTables
#   - tablaxlsx
#   - table1xls
#   - tableHTML
#   - TableMonster
#   - texreg
#   - ztable
#   - apaStyle
#   - apaTables
#   - apsrtable
# - higher-level functionality
#   - huxtable
#   - tables
#   - stargazer
#   - pixiedust
#   - reporttools
#   - rtable
#   - summarytools
#   - tab
#   - tableone
#   - carpenter
#   - dtables
#   - etable
# - pivots
#   - rpivotttable
# - other
#   - gtable
```

The [huxtable website](https://hughjonesd.github.io/huxtable/design-principles.html) provides a useful breakdown of many of these existing packages.
