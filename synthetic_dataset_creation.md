Synthetic US Survey Sample Generation
================
Armin Satzger
2022-11-20

For the creation of our synthetic dataset, we have the choice between an
empirically driven and a hypothesis driven approach.

One way to implement a hypothesis-driven approach would, for example, be
represented by using the *simstudy* R package for all survey attributes
that allows one to parametrize variable distributions and relationships
between variables. It does, however, not seem straightforward to find a
suitable parametric distribution for some of the characteristics, like
age, in particular, as heightened age-specific death factors are not
easily represented by any common parametric distribution. As ample
empirical evidence is available for at least four out of the five
questions in the desired survey design, it, therefore, seems more
appropriate to use publicly available data to inform the synthetic data
generation process.

Our goal for the synthetic survey sample is for it to be of size
$n=2000$ and to be representative of the US population. As we want it to
be representative of the overall population, the sampling method used
does not play a major role and we could use both random choice as well
as stratified sampling blocking on certain characteristics to curate our
sample.

For this reason, we may use publicly available US survey data as the
starting point for our synthetic data generation exercise. One of the
most high-quality large-scale, freely accessible and most recent sources
of data on the US population is data of the US Census, more specifically
the most recent, 2021 edition of the Public Use Microdata Sample (PUMS)
as described on [the US Census
website](https://www.census.gov/programs-surveys/acs/microdata.html).

## Obtain US Census data to base synthetic dataset on

After registering a new US Census API key, I download 2021 micro-level
US Census data via the publicly available API [^1]:

``` r
# Define API link
link <- "https://api.census.gov/data/2021/acs/acs1/pumspr?get=SEX,AGEP,HINCP,SCHL&key=c0798cc737fff9005fc6501ac1a5c2575f76b0c6"

# Read in data at provided link as dataframe
test_data <- read.csv(link)
```

``` r
# Rename columns, remove auxiliary column X
clean_data <- dplyr::rename(subset(test_data, select=-c(X)), "sex"="X..SEX", age=AGEP, income=HINCP, education="SCHL.")

# Recode sex
# descriptions from https://api.census.gov/data/2021/acs/acs1/pums/variables/SEX.json
clean_data$sex <- as.factor(as.numeric(gsub("\\[", "", clean_data$sex)))
keys <- 1:2
values <- c("Male", "Female")
keysvals <- setNames(values, keys)
clean_data$sex <- recode(clean_data$sex, !!!keysvals)

# Recode income
# descriptions from https://api.census.gov/data/2021/acs/acs1/pumspr/variables/HINCP.json
clean_data$income <- na_if(clean_data$income, -60000) # sets values with missing value code as missing

# Recode education into six major categories: no high school, high school, vocational education, undergraduate, graduate, PhD
# descriptions from https://api.census.gov/data/2021/acs/acs1/pumspr/variables/SCHL.json
clean_data$education <- as.numeric(gsub("]", "", clean_data$education)) # remove ']' string pattern from string values
keys <- 0:24 # generate sequence vector with values from 0 to 24
values <- c(NA, rep("Did not complete high school", 15), "High school diploma", "Vocational education", rep("Some college", 3), "Bachelor's degree", rep("Master's degree", 2), "PhD") # create a vector with values corresponding to numeric keys
keysvals <- setNames(values, keys) # assign key-value pairs; create named vector
clean_data$education <- as.factor(recode(clean_data$education, !!!keysvals)) # recode edu values using mapping set up above
```

Please note that factor variables, in this case sex and education, are
marked by an asterisk (\*) next to the respective variable name. The
number of data points for each question shows the largest number of
missing values for the question about household income, followed by the
question about educational attainment. This result might reflect
hesitancy to share income information due to the social stigmata of
unemployment/low social class for very low household incomes and
potential jealousy for positive income outliers.

## Create synthetic survey dataset with socioeconomic and demographic characteristics

``` r
# Define seed in order to make replication possible
my.seed <- 19234834

# Generate synthetic data of sample size k=2000 based on adult-age (>=18 years of age) US Census sample
sds.default <- syn(filter(clean_data, age >= 18), seed=my.seed, k=2000)
```

    ## Sample(s) of size 2000 will be generated from original data of size 22184.
    ## 
    ## 
    ## Synthesis
    ## -----------
    ##  sex age income education

``` r
# Assign created synthetic dataset dataframe to synthetic_data object
synthetic_data <- sds.default$syn
```

## Create life insurance dummy variable correlated with other attributes

Based on the information provided, life insurance has an overall
prevalence of c. 60% across the adult population (previously filtered
out underage respondents from the US Census data). Now, we also make a
few additional assumptions that allow us to create the life insurance
coverage attribute in the synthetic dataset: The older respondents are,
the more likely it is they have dependents or other financial
obligations that are financially dependent on the respondent’s income,
which implies a higher utility of life insurance and thus likely
positive association between active life insurance and age.
Additionally, women might be more inclined to take out life insurance as
research has shown that women generally tend to be more risk averse than
men.[^2]

``` r
# Define distribution for life insurance status attribute
addDef <- defDataAdd(varname="insured", dist="binary", formula="0.5 + 0.002 * age + 0.02 * sex_numeric")

# Add column to dataset using specified distribution
synthetic_data <- setDF(addColumns(addDef, setDT(synthetic_data)))

# Recode insured variable to bool variable
synthetic_data$insured <- if_else(synthetic_data$insured == 1, TRUE, FALSE)
```

[^1]: Please note that directly including access credentials such as API
    keys in the code constitutes a horrible security practice; common
    software engineering best practices would require storing
    credentials in a separate (.env) file but this is omitted here since
    the data is free to access and use.

[^2]: See, e.g., Eckel, C. C., & Grossman, P. J. (2008). Men, women and
    risk aversion: Experimental evidence. Handbook of experimental
    economics results, 1, 1061-1073.
