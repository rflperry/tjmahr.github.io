---
title: "Fixing APA citations from Pandoc with stringr"
excerpt: That pesky ampersand.
tags:
  - stringr
  - pandoc
  - r
---





[Pandoc][pandoc-home] is awesome. It's the universal translator for plain-text
documents. I especially like that it can do inline citations. I write
`@Jones2005 proved aliens exist` and pandoc produces "Jones (2005) proved
aliens exist".

But it doesn't quite do [APA style][apa-owl] citations correctly. A citation
like `@SimpsonFlanders2006 found...` renders as "Simpson & Flanders (2006)
found...". Inline citations are not supposed to have an ampersand. It should be
"Simpson and Flanders (2006) found...".

In the grand scheme of writing and revising, these errors are tedious low-level 
stuff. But I have colleagues who will read a draft of a manuscript and
write unnecessary comments about how to cite stuff in APA. And the problem is
just subtle and pervasive enough that it doesn't make sense to manually fix
the citations each time I generate my manuscript. My current project has 15 of
these ill-formatted citations. That number is just big enough to make manual
corrections an error-prone process---easy to miss 1 in 15.

## Find and replace

I wrote a quick R function that replaces all those inline ampersands with
*and*'s. 


```r
library("stringr")

fix_inline_citations <- function(text) {
```

Let's assume that an inline citation ends with an author's last name followed
by a parenthesized year: `SomeKindOfName (2001)`. We encode these assumptions
into [regular expression patterns][r-regex], prefixed with `re_`.

The year is pretty easy. If it looks weird, it's because I prefer to escape
special punctuation like `(` using brackets like `[(]`. Otherwise, a year is
just four digits: `\\d{4}`.


```r
  re_inline_year <- "[(]\\d{4}[)]"
```

**What's in a name?** Here we have to stick our necks out a little bit more
about our assumptions. I'm going to assume a last name is any combination of
letters, hyphens and spaces (spaces needed for `von Name`).


```r
  re_author <- "[[:alpha:]- ]+"
  re_author_year <- paste(re_author, re_inline_year)
```

We define the ampersand.


```r
  re_ampersand <- " & "
```

**Lookaround, lookaround**. Our last regular expression trick is [positive
lookahead][regex-lookaround]. Suppose we want just the word "hot" from the
larger word "hotdog". Using just `hot` would match too many things, like the
"hot" in "hoth". Using `hotdog` would match the whole word "hotdog", which is
more than we asked for. Lookaround patterns allow us to impose more constraints
on a pattern. In the "hotdog"" example, positive lookahead `hot(?=dog)` says
find "hot" if it precedes "dog".

We use positive lookahead to find only the ampersands followed by an author name
and a year. We replace the strings that match this pattern with *and*'s.




```r
  re_ampersand_author_year <- sprintf("%s(?=%s)", re_ampersand, re_author_year)  
  str_replace_all(text, re_ampersand_author_year, " and ")
}
```

We can now test our function on a variety of names that it should _and should
not_ fix.


```r
do_fix <- c(
  "Jones & Name (2005) found...",
  "Jones & Hyphen-Name (2005) found...",
  "Jones & Space Name (2005) found...",
  "Marge, Maggie, & Lisa (2005) found..."
)

fix_inline_citations(do_fix)
#> [1] "Jones and Name (2005) found..."         
#> [2] "Jones and Hyphen-Name (2005) found..."  
#> [3] "Jones and Space Name (2005) found..."   
#> [4] "Marge, Maggie, and Lisa (2005) found..."

do_not_fix <- c(
  "...have been found (Jones & Name, 2005)",
  "...have been found (Jones & Hyphen-Name, 2005)",
  "...have been found (Jones & Space Name, 2005)",
  "...have been found (Marge, Maggie, & Lisa, 2005)"
)  

fix_inline_citations(do_not_fix)
#> [1] "...have been found (Jones & Name, 2005)"         
#> [2] "...have been found (Jones & Hyphen-Name, 2005)"  
#> [3] "...have been found (Jones & Space Name, 2005)"   
#> [4] "...have been found (Marge, Maggie, & Lisa, 2005)"
```

By the way, our final regular expression `re_ampersand_author_year` is 
``  & (?=[[:alpha:]- ]+ [(]\d{4}[)]) ``. It's not very readable or comprehensible in
that form, so that's why we built it up step by step from easier sub-patterns
like `re_author` and `re_inline_year`. (Which is a micro-example of the strategy
of managing complexity by combining/composing simpler primitives.)

## Steps towards production

These are complications that arose as I tried to use the function on my actual
manuscript:

**Placing it in a build pipeline**. My text starts with an RMarkdown file that
is knitted into a markdown file and rendered into other formats by pandoc.
Because this function post-processes output from pandoc, I can't just hit the
"Knit" button in RStudio. I had to make a separate script to do
`rmarkdown::render()` to convert my .Rmd file into a .md file which can then be
processed by this function.

**Don't fix too much**. When pandoc does your references for you, it also does
a bibliography section. But it would be wrong to fix the ampersands there. So
I have to do a bit of fussing around by finding the line `"## References"` and
processing just the text up until that line.

**Accounting for encoding**. I use `readr::read_lines()` and
`stringi::stri_write_lines()` to read and write the text file to preserve the
encoding of characters. (readr just released its own `write_lines()` today
actually, so I can't vouch for it yet.)

**False matches are still possible**. Suppose I'm citing a publication by an
organization, like Johnson & Johnson, where that ampersand is part of the name.
That citation would [wrongly be corrected][pandoc-issue]. I have yet to face
that issue in practice though.



***

*Last knitted on 2021-02-15. [Source code on
GitHub](https://github.com/tjmahr/tjmahr.github.io/blob/master/_R/2016-08-04-fixing-apa-citations-from-pandoc.Rmd).*[^si] 

[^si]: 
    
    ```r
    sessioninfo::session_info()
    #> - Session info ---------------------------------------------------------------
    #>  setting  value                       
    #>  version  R version 4.0.3 (2020-10-10)
    #>  os       Windows 10 x64              
    #>  system   x86_64, mingw32             
    #>  ui       RTerm                       
    #>  language (EN)                        
    #>  collate  English_United States.1252  
    #>  ctype    English_United States.1252  
    #>  tz       America/Chicago             
    #>  date     2021-02-15                  
    #> 
    #> - Packages -------------------------------------------------------------------
    #>  package     * version date       lib source        
    #>  assertthat    0.2.1   2019-03-21 [1] CRAN (R 4.0.2)
    #>  cli           2.3.0   2021-01-31 [1] CRAN (R 4.0.3)
    #>  evaluate      0.14    2019-05-28 [1] CRAN (R 4.0.2)
    #>  git2r         0.28.0  2021-01-10 [1] CRAN (R 4.0.3)
    #>  glue          1.4.2   2020-08-27 [1] CRAN (R 4.0.2)
    #>  here          1.0.1   2020-12-13 [1] CRAN (R 4.0.3)
    #>  knitr       * 1.31    2021-01-27 [1] CRAN (R 4.0.3)
    #>  magrittr      2.0.1   2020-11-17 [1] CRAN (R 4.0.3)
    #>  ragg          0.4.1   2021-01-11 [1] CRAN (R 4.0.3)
    #>  rprojroot     2.0.2   2020-11-15 [1] CRAN (R 4.0.3)
    #>  sessioninfo   1.1.1   2018-11-05 [1] CRAN (R 4.0.2)
    #>  stringi       1.5.3   2020-09-09 [1] CRAN (R 4.0.2)
    #>  stringr     * 1.4.0   2019-02-10 [1] CRAN (R 4.0.2)
    #>  systemfonts   1.0.0   2021-02-01 [1] CRAN (R 4.0.3)
    #>  textshaping   0.2.1   2020-11-13 [1] CRAN (R 4.0.3)
    #>  withr         2.4.1   2021-01-26 [1] CRAN (R 4.0.3)
    #>  xfun          0.20    2021-01-06 [1] CRAN (R 4.0.3)
    #> 
    #> [1] C:/Users/Tristan/Documents/R/win-library/4.0
    #> [2] C:/Program Files/R/R-4.0.3/library
    ```

[pandoc-home]: http://pandoc.org/ "pandoc: a universal document converter"
[apa-owl]: https://owl.english.purdue.edu/owl/section/2/10/ "Purdue Online Writing Lab: APA Style" 
[r-regex]: https://stat.ethz.ch/R-manual/R-devel/library/base/html/regex.html "R manual page on regular expressions"
[regex-lookaround]: http://www.regular-expressions.info/lookaround.html "Lookaround patterns in regular expressions"
[pandoc-issue]: https://github.com/jgm/pandoc-citeproc/issues/177#issuecomment-144743188 "Github issue: Joining author names in text and in parentheses"
