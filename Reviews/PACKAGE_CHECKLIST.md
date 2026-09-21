# moveEZ: what to check in the package before the case study

Tested with moveEZ 1.3.1, biplotEZ 3.0, GPAbin 1.1.1 on 2026-09-21.
"Confirmed" means I reproduced it. "To check" means I saw it in the source but did not test it.
Details and numbers for most items are in `NOTES.md`.

Sources: my tests, both reviews in full (`1-review-4.pdf`, `1-review-5.txt`), and the "Package Changes" sheet of the tracker (23 rows).
IDs such as R5-C02 are the tracker IDs. Section G maps each of the 23 tracker rows to an item here.
Items marked **NEW** are not in any review. I found them in the tests.

## A. Problems that change results in the paper (fix first)

### A1. `evaluation()` calculates the bias measures on the unaligned biplots. Confirmed. (R4-04, R5-C02, R4-07)
- AMB, MB and RMSB use `target - testee`, where `testee` is `bp$coord_set[[i]]` (before GPA). A PCA sign flip then looks like a large bias.
- Effect in the current paper: AMB for 1950 is 1.27 and 0.39 to 0.53 for the other years. On the aligned biplots 1950 is 0.23, in line with the rest, and 2000 is the largest (which agrees with PS and CC). The text on Figure "bias-line-plot" ("the initial bias is high, but decreases and stabilizes from 1960") describes the sign flip.
- The function calculates `X.new` (the fitted configuration) and never uses it. PS leaves out `b.fact`.
- MB is always 0. Reviewer 4 says an `abs()` is missing. With `abs()` MB is the same as AMB. The real cause is the centring: the mean difference of two centred matrices is 0. Decide if MB stays, and what it must measure.
- R4-07: `Tot.SS`, `Fit.SS`, `n.X`, `p.X` and `p.Y` are also calculated and not used. Remove them or use them.
- **NEW:** the effect on the paper (next point). The reviewers saw the code problem but not that it changes the 1950 result.
- Note: the reviewer says CC is also on the wrong basis. CC uses distances between points, so rotation and reflection do not change it. CC and PS are correct.
- Fix: calculate the bias measures on `bp$GPA_list[[i]]` or on `X.new`. Then make `fitplot.png` and `biasplot.png` again and rewrite the two paragraphs.

### A2. `GPAbin::GPA()` standardises each coordinate column. Confirmed. (R5-07, R5-C01. The column standardisation and the constant `sk.F` are **NEW**)
- First step is `sapply(Xk, scale)`, which divides PC1 and PC2 each by its own sd. This is not an isotropic scaling. It changes the angles between the variable axes by up to 29 degrees (Month data). In December a correlation of +0.49 is drawn with an angle of 100 degrees.
- The paper says the GPA transformations "preserve the distances between the coordinates". That is not true for this step.
- The returned `sk.F` is always 1 (it is set once and never updated). So `s.list` in `moveplot3()` tells the user nothing.
- Decide: (a) change GPAbin to `scale(X, scale = FALSE)` and return the real scale factors, or (b) do the centring and the GPA call inside moveEZ with control of the scaling, or (c) keep it and document it. Option (a) or (b) is the clean answer to the reviewer.
- Then: expose a `scaling = TRUE/FALSE` argument in `moveplot3()` and return the scale factors.

### A3. `moveplot3()` stacks samples and axes in one matrix for GPA. Confirmed. (**NEW**. Related to R4-03 and R5-06)
- GPA gets 80 (or 120) sample rows and 6 axis rows, so the samples control the rotation. The axes get no weight of their own.
- The matrix is centred on the mean of samples plus axis end points. This mean is not zero, but the arrows are drawn from (0, 0). The error is small (1 to 2 degrees).
- Decide: is "align on the samples, then apply the same transformation to the axes" the intended method? If yes, fit GPA on the sample rows only and apply `Q` and `s` to the axis rows. That also removes the origin shift. State the choice in the paper.

### A4. `moveplot3()` ignores the level order of `time.var`. Confirmed for facets. To check for the animation. (**NEW**)
- The axis tables store the time variable as character (`time.var = iter_levels[i]`). The sample table keeps the factor. Facets come in alphabetical order (April, August, December, ...). Panel content is correct.
- `transition_states()` gets the same data, so check the order of the animation with `move = TRUE` and Month.
- Not visible with Year, because "1950" to "2020" sort the same way.
- Fix: `factor(iter_levels[i], levels = iter_levels)` for both axis tables. `moveplot()` and `moveplot2()` are correct.

### A5. `moveplot3()` depends on the row order and does not check it. Confirmed. (R5-C04)
- GPA matches row i of one slice with row i of the next. The same data in a shuffled row order gives a different alignment (axis coordinates differ by up to 0.07 in the test).
- A target with the wrong number of rows gives "non-conformable arguments".
- Fix: check `nrow(target)` against the slice size with a clear message. Document that rows must correspond. Better: add an `id.var` argument (or sort by `group.var` plus an id) so that the match is explicit.

## B. Problems that break use outside the Year example

### B1. `evaluation()` assumes numeric time levels and the name "Year". Confirmed. (R5-C03)
- `as.numeric(gsub("Target vs. ", "", ...))` gives NA for months. `fit.plot` and `bias.plot` are empty, with 12 warnings. The column and the axis label are always `Year`.
- Fix: keep the levels as an ordered factor from `bp$iter_levels`, and use the name of `time.var`. `moveplot3()` must then store `time.var` in the object.

### B2. The object from `moveplot2()` fails in `print()`. Confirmed. (**NEW**. Related to "inconsistent output structures" in R5-C00)
- `bp$quality` is overwritten with a `knitr::kable`. `print.biplot()` calculates `x$quality * 100` and stops with "non-numeric argument to binary operator". This occurs when the result is not assigned, for example at the end of a pipe.
- Fix: store the tables under new names (`bp$quality.time`, `bp$axis.predictivity.time`), or return data frames and let the user call `kable()`. Data frames are also easier to use in a paper than a ready kable. Same point for `evaluation()$eval.tab`: the paper now takes the numbers from the plot data because `eval.tab` is a kable.

### B3. `evaluation()` on a wrong object prints a message and continues. Confirmed. (R4-06)
- It prints "Evaluation measures can only be applied for moveplot3()." and returns the object. Use `stop()`.

### B4. `group.var = NULL`. Confirmed. (**NEW**)
- The three functions have code for `is.null(group.var)`, but `check_vars_moveEZ()` rejects NULL. Either permit NULL or remove the dead branches. Check what the help pages say.

### B5. Checks that `moveplot2()` has and `moveplot3()` does not. To check. (R5-C05)
- `moveplot2()` stops when a slice has fewer than 4 rows. `moveplot3()` has no such check.
- Hull guards (R5 code 5): in 1.3.1 both call `chull_moveEZ()`, which seems to handle small groups. Confirm that the help pages of `moveplot2()` and `moveplot3()` say so.
- `moveplot3()` handles only PCA. `moveplot2()` also handles CVA. Check what `moveplot3()` does with a CVA object (clear error or wrong result).

### B6. Title size. Confirmed. (**NEW**)
- `moveplot()` sets `plot.title` to size 30. This is good for the animation. If a user adds a title to the facet plot, it is very large. Low priority.

## C. Package data (**NEW**)

- `Africa_climate$Month` has alphabetical levels. Give it calendar-order levels (`levels = month.name`). Without this, every Month example needs a relevel first.
- Row order in `Africa_climate` is already the same in each Month slice and each Year slice. Keep it like that and say so in the help page (see A5).
- NEAF 1950 (mean SPI6 2.12, mean AP 3.31) is far from all later years. Check it against the ERA5 source before any text interprets it.

## D. Code quality and documentation (from the reviews)

- R5-C06: validation of `time.var` and `group.var`. Mostly done in 1.3.1 (`check_vars_moveEZ()` checks factor and NA with clear messages). Confirm that the help pages match, and document the NA handling.
- R5-C07, R4-07: the ggplot code is copied in the three functions, and `moveplot_func()` is unused code in the tarball. A shared internal plot function will also stop bugs like A4, where one copy differs from the others.
- R5-C08: `knitr`, `purrr` and `scales` are used at run time but are in Suggests. Move them to Imports or add `requireNamespace()` checks.
- R5-C09: no tests. See E.
- R5-C00: full check of the help pages, examples and vignette. The reviewer reports copy-paste errors, and examples and parameter descriptions that do not match the code. Read each help page against the function arguments.
- R4-09 (Maybe): the wording "squared sum of squares distances". Check if the `moveplot3()` help page has the same phrase as the paper.
- R5-02, R5-10 (Maybe): put the guidance in the help pages and the vignette, not only in the paper. When to use the fixed frame and when the dynamic frame, and how to read movement in each. The text in `case_study_draft.Rmd` ("Which axes to interpret", the guide at the end of Part A) can be the start.

## D2. Feature and API requests (from the reviews). A decision is necessary for each.

- R4-05, R5-19: function names that tell what each function does. Aliases can keep the old names, so current code does not break. Both reviewers ask for this.
- R5-24: plot `1 - CC` in `fit.plot`, so that PS and CC read in the same direction and the small CC range is visible. Do this together with A1 and B1, because all three change `evaluation()`.
- R5-22: an option for 90% or 95% ellipses next to the convex hulls. Hulls are sensitive to outliers. "Will not change" with a reason in the response is also possible.
- R5-03 (Maybe): an option to select other components than PC1 and PC2. The axis predictivity table shows why this can matter: SPI6 and, in winter, Temp are mostly on PC3. If the package does not get the option, the paper must justify the limit.
- R4-03 (Maybe): an option for a single-step alignment of each slice to one target slice, without the GPA iteration. Reviewer 4 says GPA is not necessary if the subspace is not important. This links to A3: decide what is aligned (samples, axes or the two together) and to what.

## E. Tests to add (each one comes from a bug above)

1. Facet and animation order follow the factor levels for a non-alphabetical time variable (A4).
2. `evaluation()` bias measures do not change when one slice is reflected (A1).
3. The angle between two axes is the same before and after alignment, if scaling is off (A2).
4. Shuffled rows give an error or the same result (A5).
5. A target with the wrong number of rows gives a clear error (A5).
6. `evaluation()` works with month names as time levels (B1).
7. `print()` works on the result of each function (B2).

## F. After the fixes: what to do again in the paper

1. Make all `moveplot3()` figures again (`biplot_targetNULL.png`, the 1989 target figure). With A2 and A3 fixed, the axis directions will change.
2. Make `fitplot.png`, `biasplot.png` and the comparison table again. Rewrite the interpretation (A1).
3. Correct the sentence on admissible transformations in the GPA section, and state what scaling is used (A2).
4. Run `New case study/case_study_simple.R` again and remove its temporary lines for the facet order (A4). Then check the numbers in `case_study_draft.Rmd`. The correlations and the axis predictivities will not change. The rotation angles (112 and 123 degrees) will.
5. Update the responses for R5-07, R5-C01 and the code comments.

## G. Coverage: the 23 rows of the tracker sheet "Package Changes"

| Tracker ID | Item here | State after my tests |
| --- | --- | --- |
| R4-04 | A1 | Confirmed. Also changes the 1950 result in the paper. |
| R5-C02 | A1 | Confirmed. CC is correct (it uses distances). PS leaves out `b.fact`. |
| R4-07 | A1, D | Confirmed in the source. |
| R5-07 | A2 | Confirmed: scaling is on. |
| R5-C01 | A2 | Confirmed. Also: column standardisation, and `sk.F` is always 1. |
| R5-C04 | A5 | Confirmed: unclear error for a wrong row count, and a silent change for a different row order. |
| R5-C03 | B1 | Confirmed with Month. |
| R4-06 | B3 | Confirmed. |
| R5-C05 | B5 | To check. 1.3.1 seems to guard small groups in all three functions. |
| R5-C06 | D | Mostly done in 1.3.1. Check the documentation. |
| R5-C07 | D | Not tested (package source is not in this repository). |
| R5-C08 | D | Not tested. |
| R5-C09 | E | Seven tests proposed. |
| R5-C00 | D | Not tested. B2 and B4 are examples of the inconsistencies. |
| R4-09 | D | To check in the help page. |
| R5-02 | D | Draft text exists in `case_study_draft.Rmd`. |
| R5-10 | D | Draft text exists in `case_study_draft.Rmd`. |
| R4-05 | D2 | Decision. |
| R5-19 | D2 | Decision. |
| R5-24 | D2 | Decision. Small change. |
| R5-22 | D2 | Decision. |
| R5-03 | D2 | Decision. |
| R4-03 | D2, A3 | Decision. |

Not in the tracker, found in the tests (add them to the tracker if you agree): A3, A4, B2, B4, B6, C, the column standardisation and constant `sk.F` in A2, and the effect of A1 on the 1950 result.

Reviewer comments that need no package change are not in this list (for example R4-02, the formal description of GPA, and the comments on related work). They are in the tracker sheet "All Comments".
