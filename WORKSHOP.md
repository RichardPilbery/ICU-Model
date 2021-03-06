# ICU Model
> ICU Patient flow modelling using r-simmer

Written for the NHS-R Conference 2019

Based on:

>Lawton T, McCooe M. POLICY: A novel modelling technique to predict resource -requirements in critical care - a case study. Future Healthc J. 2019;6(1):17–20. doi:10.7861/futurehosp.6-1-17
 [http://futurehospital.rcpjournal.org/content/6/1/17.abstract] / [https://www.ncbi.nlm.nih.gov/pmc/articles/PMC6520084/]

Produced as a teaching package to expand use of the technique being used in the Health Foundation's [Winter Pressures Modelling Project](https://www.health.org.uk/improvement-projects/use-of-novel-modelling-techniques-and-routinely-collected-data-to-explore-responses-to)

## Meta

Tom Lawton – [@LawtonTri](https://twitter.com/lawtontri)

Distributed under the GNU GPL

## Workshop


### Assumptions

Reasonable familiarity with R as a programming language, including use of [pipe operator](https://r4ds.had.co.nz/pipes.html) `%>%` and functions.

### Step 0

Familiarisation with [r-simmer](https://r-simmer.org/)

Vignettes are in [r-simmer vignettes/](r-simmer%20vignettes/)

Take a look at [The Bank Tutorial: Part I](https://r-simmer.org/articles/simmer-04-bank-1.html).

Important concepts: `seize`,`timeout`,`release`,`add_generator`,`add_resource`

Look at "Balking and reneging customers" from [The Bank Tutorial: Part II](https://r-simmer.org/articles/simmer-04-bank-2.html#balking-and-reneging-customers)

Important concepts: `renege_in`,`renege_abort`

The important sections of these two vignettes have been copied into [0 - Bank model sections.Rmd](0%20-%20Bank%20model%20sections.Rmd) which may be easier to go through.

Keep the [r-simmer reference](https://r-simmer.org/reference/) open during subsequent steps.

### Step 1

Build a simple model of an ICU with 16 beds, and elective and emergency patients arriving.

Some suggestions for a sensible model:

* `add_resource("bed",16)`
* Emergency intervals: `round(rexp(1,1/1))`
* Elective intervals: `round(rexp(1,1/3))`
* Patient stay: `timeout(function() {round(runif(1,0,30))})`
* Suggest putting the model environment into a wrapper for later Monte-Carlo use

**Solution:** [1 - Partmodel-BedsOnly.R](1%20-%20Partmodel-BedsOnly.R)

### Step 2

Update model to include nurses as a resource. Our ICU has 12 nurses; patients require 1 nurse when level 3, and can share a nurse when level 2 or below.

Some suggestions:

* `add_resource("halfnurse",24)`
* 50% split between starting L3/L2 - `seize("halfnurse", function() { if (runif(1)<0.5) 2 else 1 })`
* At initial level - `timeout(function() {round(runif(1,0,15))})`
* At lower level (after releasing half a nurse if L3) - `timeout(function() {round(runif(1,0,15))})`

**Solution:** [2 - Partmodel-AddedNurses.R](2%20-%20Partmodel-AddedNurses.R)

### Step 3

Split single patient trajectory into elective vs emergency. Elective patients return a week later if they are rejected.

Some suggestions:

* Split out a common patient trajectory for once a patient has been admitted (of either type) - use `join()`
* Elective: `timeout(0.1)` to make emergency patients take priority
* Use `rollback()` to allow elective patients to return

**Solution:** [3 - Partmodel-ElectiveAndEmergency.R](3%20-%20Partmodel-ElectiveAndEmergency.R)

### Step 4

Generate ICNARC L3/2/1/0 days at the start of an admission (randomly) and then drive trajectories based on these

Some suggestions:

* Split out a "generator" trajectory to handle the generation
* `set_attribute` to store the days once generated
* If >0 L3 days, patient will need a whole nurse to be admitted, and can drop to 1:2 later
* (Optional) - correct stays using an offset, for the way ICNARC days are calculated (eg 8pm-8am counts as 2 days)
* Some sensible values: L3 - `rpois(1,5)`, L2 - `rpois(1,5)`, L1 - `rpois(1,3)`, L0 - `rpois(1,1)`

**Solution:** [4 - Partmodel-DaysAsAttributes.R](4%20-%20Partmodel-DaysAsAttributes.R)

### Step 5

Create a generator function for time gaps between patients so we can later model changes by day etc

Some suggestions:

* Use a [closure](http://adv-r.had.co.nz/Functional-programming.html#closures) to keep track of the current date and when the last patient was generated
* Return inter-arrival time gaps (which may be zero), and -1 to finish
* Some sensible values: `rpois(1,mean)` where mean is 0.7 for emergency and 0.3 for elective

**Solution:** [5 - Partmodel-PoissonGenerator.R](5%20-%20Partmodel-PoissonGenerator.R)

### Step 6

Drive the model with real data from [ICNARC Test Data.csv](ICNARC%20Test%20Data.csv)

Some suggestions:

* To read data in:
```r
patients<-read.csv("ICNARC Test Data.csv",header=TRUE,sep=",",colClasses=c("Admission.Date"="character"))
patients$Admission.Date<-as.Date(patients$Admission.Date,format="%d/%m/%Y")
```
* Calculate totals by weekday (or business day) and admission type
* In generator, use mean for current weekday/type as input to the `rpois`
* In patient days generator trajectory, select a real patient of the appropriate type (use `set_attribute` and `get_attribute` to store patient types)
* Use `mclapply` to run a Monte-Carlo simulation (environment needs to be in a wrapper if it isn't already)

**Solution:** [6 - Partmodel-DataDriven.R](6%20-%20Partmodel-DataDriven.R)

### Extra credit

* Add repatriations and transfers in, handled sensibly. Repatriations are lower priority but will usually be re-requested the following day if rejected. Transfers in are lowest priority.
* Make pretty graphs to demonstrate the model's power
* Tweak the model parameters (eg beds, nurses) and see what the effects would be
* Compare what would happen if "delayed discharges" were removed earlier (L1 and below)
* Simulate an increase in elective/emergency workload

## Acknowledgements

![Health Foundation Logo](https://www.health.org.uk/themes/custom/health_foundation/assets/images/logo.png)

The Winter Pressures Modelling Project is part of the Health Foundation’s Applied Analytics programme. The Health Foundation is an independent charity committed to bringing about better health and health care for people in the UK.

Massive thanks to Iñaki Ucar, creator of r-simmer, for his support of the work - both in terms of advice and modifications to r-simmer to make it more suitable for this type of work.
