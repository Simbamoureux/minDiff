# minDiff

minDiff is an R-package that can be used to assign elements to a
specified number of groups and minimize differences between created
groups. This functionality is implemented in the method
`create_groups()`.

I created this package to assign stimuli in psychological experiments to
different conditions so that conditions are as equal as possible, a
priori. But I am sure that a wide range of applications is possible (for
an example, see section 'Usage').

## Installation

Install the `devtools` package first via `install.packages("devtools")`.

To install the minDiff package, type in your R console

`devtools::install_github("m-Py/minDiff")`

## Usage

Load the package via

```S
library("minDiff")
```

To reproduce the following example, install the "MASS" package that
contains the data set that is used.

```S
library("MASS")
data(survey)       # load data set
head(survey, n=10) # look at the data
``` 

The `survey` data set contains some demographic information on a student
sample. Imagine you wish to assign students to three different
dormitories and want to create a similar groups of students in each
house. As a first step, we might want to match average student age.

To do so, we pass our data set to the function `create_groups()`. We
also specify that we want to create three groups whose ages should be
equal on average (specified by passing the function `mean` to the
argument `equalize`).

```S

equal <- create_groups(survey, criteria_scale=c("Age"), 
                       sets_n = 3, repetitions=10, equalize=list(mean))

```

By passing the column `survey$Age` to the argument `criteria_scale` we
tell the function that age is a continuous variable that we want to
compute the mean for in each of the newly created groups. The function
returns a data.frame that is saved into the object `equal`. `equal` is a
shuffled version of the original data set `survey`, which has one
additional column: `newSet` - this is the group assignment variable that
was created.

Let's have a look at this:

```S
> table(equal$newSet)
 1  2  3 
79 79 79 
```

The `survey` data set has 237 entries, which can be assigned to three
groups of equal size. If the data set had not been a multiplier of the
group number, `create_groups()` would have created groups that are of
similar size.

Let's see how successful we were in creating groups of age:

```S
> tapply(equal$Age, equal$newSet, mean)
       1        2        3 
20.35449 20.57806 20.19099 
```

Not so bad! But how did it work? In the function call above, we
specified another parameter, `repetitions=10`. This means that the
function randomly assigned all cases (i.e. students) to three groups ten
times, and returned the most equal group assignment. What is considered
most equal is determined by the parameters `criteria_scale` and
`equalize`.

By varying the parameter `repetitions` we can increase our chances of
creating equal groups. Let's see what happens if we do only one trial -
in this case, the data set is simply shuffled once and no optimization
is conducted:

```S
equal <- create_groups(survey, criteria_scale=c("Age"), 
                       sets_n = 3, repetitions=1, equalize=list(mean))

> tapply(equal$Age, equal$newSet, mean)
       1        2        3 
20.71423 19.54434 20.86497                
```

If we conduct 10,000 repetitions (which is still very fast if we only
consider one variable), the groups will be very similar with regards to
age. Note that it is possible to pass a data set that has been optimized
previously; in this case, the program does not start all over, but only
tries find more similar groups than the previous best assignment:

```S
equal <- create_groups(equal, criteria_scale=c("Age"), 
                       sets_n = 3, repetitions=10000, equalize=list(mean))
                     
> tapply(equal$Age, equal$newSet, mean)
       1        2        3 
20.37028 20.38194 20.37133
```

Nice! How small differences between sets can become depends on the
original data.

### Considering more than one criterion

More than one criterion can be passed to the function. Let's imagine I
also want students to be of equal heights in all dormitories:

```S
equal <- create_groups(survey, criteria_scale=c("Age", "Height"), 
                       sets_n = 3, repetitions=1000, equalize=list(mean))
                      
> tapply(equal$Age, equal$newSet, mean)
       1        2        3 
20.37877 20.35233 20.39244

> tapply(equal$Height, equal$newSet, mean, na.rm=TRUE)
       1        2        3 
172.5658 172.3694 172.2292
```

Note that there were missing values in the variable
`survey$Height`. This is given out as a warning by `create_groups()`,
but it will still return a result (and simply disregards the missing
value in the variable height).

### Considering categorical criteria

We may not only wish to minimize differences with regards to age or
height, but we might want to create equal gender ratios in all
dormitories. Let's check in what ratios our previous group assignment
resulted:

```S
> table(equal$newSet, equal$Sex)

    Female Male
  1     36   43
  2     38   41
  3     44   34

```

We can see that gender ratios are very different between
dormitories. `create_groups()` offers the possibility to consider
categorical variables when creating groups. These are passed via the
`criteria_nominal` parameter. Let's try this out:

```S

equal <- create_groups(survey, criteria_scale=c("Age"), 
                       criteria_nominal=c("Sex"), tolerance_nominal=c(2),
                       sets_n = 3, repetitions=100, equalize=list(mean))

```

By specifying the parameter `tolerance_nominal = 2`, we tell the
function that we tolerate deviations in the frequency between new sets
of no more than 2. Did that work?

```S
> table(equal$newSet, equal$Sex)

    Female Male
  1     40   39
  2     38   40
  3     40   39
```

Note that in this case, we could decrease our tolerance for deviations
to 1, but it is impossible to assign the same number of female and male
students to all dormitories in this case. If a tolerance of 0 is passed,
the function will never stop executing, so be careful with low tolerance
values if you are not sure how categories can be assigned to groups.

### Use more than one categorical variable

It is possible to pass two categorical criteria `create_groups()`. There
is no limit for scale criteria, but only two categorical variables can
be passed (that is because two variables can already make the program
run really slow).

Here is an example where I use two categorical and two scale variables:

```S

equal <- create_groups(survey, criteria_scale=c("Age", "Height"),
                       criteria_nominal=c("Sex", "Smoke"),
                       tolerance_nominal=c(2, 3, Inf), sets_n = 3,
                       repetitions=20, equalize=list(mean))

> table(equal$newSet, equal$Sex)

    Female Male
  1     39   40
  2     40   38
  3     39   40

> table(equal$newSet, equal$Smoke)

    Heavy Never Occas Regul
  1     4    63     6     6
  2     4    62     6     6
  3     3    64     7     5
```

Note that the parameter `tolerance_nominal` expects a vector of length 3
if two categorical variables are passed to `criteria_nominal`. These
values indicate tolerances for deviations in the first, the second and
the combined categorical levels. In doubt, use large tolerance values
when starting to optimize your groups and see how fast the function is
running. In the upper case I was not interested to assign an equal
number of smoking females and males to each dormitories and considered
both categorical variables independently from each other. So, I set the
tolerance for deviations in the combined groups to infinity.

### Use other equalizing functions

I could be interested not only in equalizing mean age between groups,
but also the standard deviation of ages to achieve similar distributions
of age between dormitories. This is possible by passing another function
to the `equalize` parameter.

```S
equal <- create_groups(survey, criteria_scale=c("Age"), 
                      criteria_nominal=c("Smoke"), 
                      tolerance_nominal=c(2),
                      sets_n = 3, repetitions=500, equalize=list(mean, sd))

> tapply(equal$Age, equal$newSet, mean)
       1        2        3 
20.21943 20.41986 20.48425 

> tapply(equal$Age, equal$newSet, sd)
       1        2        3 
6.599697 6.995727 5.855852 
````

### Performance

Simple group assignments are fast and effective. The most simple case is
to have only one criterion and two groups - the groups will probably
become very similar very fast. If more groups are required, more
criteria are specified and more equalizing functions are considered, the
function will run slower and results may be less optimal. I suggest to
try out different settings, and start with simpler requirements. Note
that using categorical variables will make the function very slow if the
tolerance level is low.

## Feedback / Bug reports

Any feedback or reports on bugs is greatly appreciated; just open an
issue or write an email to martin.papenberg at uni-duesseldorf.de.
