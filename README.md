[![Contributors][contributors-shield]][contributors-url]
[![Forks][forks-shield]][forks-url]
[![Stargazers][stars-shield]][stars-url]
[![Issues][issues-shield]][issues-url]
[![MIT License][license-shield]][license-url]

<!-- PROJECT LOGO -->
<br />
<div align="center">
  <a href="https://github.com/austinmarckx/2022UTSWMachineLearningFinalProj">
    <img src="img/test.jpg" alt="Logo" width="240" height="320">
  </a>

<h3 align="center">2022 UTSW Machine Learning Group Project</h3>

  <p align="center">
   Project 8: Alzheimer's Disease, predicting delta MMSE 
    <br />
    <a href="https://github.com/austinmarckx/2022UTSWMachineLearningFinalProj"><strong>Explore the docs »</strong></a>
    <br />
    <br />
    <a href="https://github.com/austinmarckx/2022UTSWMachineLearningFinalProj">View Demo</a>
    ·
    <a href="https://github.com/austinmarckx/2022UTSWMachineLearningFinalProj/issues">Report Bug</a>
    ·
    <a href="https://github.com/austinmarckx/2022UTSWMachineLearningFinalProj/issues">Request Feature</a>
  </p>
</div>

<!-- TABLE OF CONTENTS -->
<details>
  <summary>Table of Contents</summary>
  <ol>
    <li>
      <a href="#about-the-project">About The Project</a>
    </li>
    <li>
      <a href="#getting-started">Getting Started</a>
      <ul>
        <li><a href="#preprocessing">Preprocessing</a></li>
        <li><a href="#exploratory-plotting">Exploratory Plotting</a></li>
        <li><a href="#feature-selection">Feature Selection</a></li>
      </ul>
    </li>
    <li>
        <a href="#modeling">Modeling</a>
        <ul>
            <li><a href="#model-selection">Model Selection</a></li>
            <li><a href="#model-training">Model Training</a></li>
            <li><a href="#model-evaluation">Model Evaluation</a></li>
        </ul>
    </li>
    <li><a href="#conclusions">Conclusions</a></li>
    <li><a href="#project-contributions">Project Contributions</a></li>
    <li><a href="#acknowledgments">Acknowledgments</a></li>
  </ol>
</details>


<!-- ABOUT THE PROJECT -->
## About The Project

<figure>
  <img src="./img/projdetails.png", width = "640">
  <figcaption><b>Fig 1.</b> Project Details.</figcaption>
</figure>

<br></br>

What is the MMSE?

<figure>
  <img src="./img/mmseOverview.png", width = "640">
  <figcaption><b>Link: </b> https://oxfordmedicaleducation.com/geriatrics/mini-mental-state-examination-mmse/ </figcaption>
</figure>

<br></br>

Sample MMSE:
<figure>
  <img src="./img/mmse.png", width = "640">
  <figcaption><b>Link: </b> https://cgatoolkit.ca/Uploads/ContentDocuments/MMSE.pdf </figcaption>
</figure>

<p align="right">(<a href="#top">back to top</a>)</p>

<!-- GETTING STARTED -->
## Getting Started

We did some stuff in this project such as:

* This
* That
* Also this
* etc.

### Preprocessing

```{r}
# Read in file
df <- read_xlsx('./data/AD.training.xlsx') %>% 
transform(
  PTID = factor(PTID),
  DX = factor(DX.bl),
  PTGENDER = factor(PTGENDER),
  APOE4 = factor(APOE4)
)

df %>% head()
```

<figure>
  <img src="./img/dfHead.png", width = "1080">
  <figcaption><b>Fig 2.</b> Head of Dataset.</figcaption>
</figure>

<p align="right">(<a href="#top">back to top</a>)</p>

It is important to realize here that our relevant feature space is _extremely_ limited.  Of the 10 original columns in the dataset, 1 (`MMSE.Change`) is the response variable, and 3 (`RID`, `PTID`, `EXAMDATE`) have minimal-no predictive information.  This leaves only 6 columns for training on a dataset with 384 entries. In short, this is very little data for a very complicated problem and this will likely result in relatively high variance in the regressive models.

### Exploratory Plotting

```{r, fig.width= 18, fig.height=18, message = FALSE, warning = FALSE}
df %>% 
  mutate(asinsqrtMMSE = asin(sqrt(minmax(MMSE)))  ) %>%
  select(AGE, PTEDUCAT, MMSE, asinsqrtMMSE, everything(), MMSE.Change, -RID, -PTID, -EXAMDATE) %>% # removing factors with too many levels or irrelevant features
  ggpairs(aes(fill = DX, alpha = 0.7), progress = FALSE)

```

<figure>
  <img src="./img/pairs_colorDX.png", width = "1080">
  <figcaption><b>Fig 3.</b> Exploratory Pair Plot.</figcaption>
</figure>

```{r}
# template: https://www.datanovia.com/en/blog/elegant-visualization-of-density-distribution-in-r-using-ridgeline/
df %>%
  ggplot(aes(x = `MMSE.Change`, y = DX, group = DX, fill = factor(stat(quantile)))) + 
  stat_density_ridges(
    geom = "density_ridges_gradient",
    calc_ecdf = TRUE,
    quantiles = 4,
    quantile_lines = TRUE, 
    jittered_points = TRUE,
    scale = 0.9,
    position = position_points_jitter(width = 0.05, height = 0.1),
    point_size = 1, point_alpha = 0.5, alpha = 0.7) + 
  theme_minimal() +
  scale_fill_brewer(name = 'Quartiles')
```

<figure>
  <img src="./img/mmseBasevsChangeRidgeLine.png", width = "720">
  <figcaption><b>Fig 4.</b> MMSE Ridgeline Plot.</figcaption>
</figure>

Due to the basal distributions being quite similar in both shape and range for the majority of values, it will likely be quite difficult to get a predictor with low variance.

```{r}
df %>%
  select(PTID, `MMSE.Change`, MMSE, DX) %>%
  mutate(MMSE4mo = MMSE + `MMSE.Change`) %>%
  select(-`MMSE.Change`) %>%
  pivot_longer(cols = c(MMSE4mo, MMSE),  names_to = 'time', values_to = 'MMSE') %>%
  ggplot(aes(x = MMSE, y = time, group = PTID, color = DX)) + 
    geom_jitter(width = 0.2, height = 0.075) +
    geom_line() +
    geom_violin(aes(color = NULL, fill = time, group = time), alpha = 0.15, draw_quantiles = c(0.25, 0.5, 0.75)) + 
  theme_minimal() +
  facet_wrap(~DX)
```

<figure>
  <img src="./img/MMSE_densityLinePairs_byDiag.png", width = "720">
  <figcaption><b>Fig 5.</b> Change in MMSE by DX Ridgeline Plot.</figcaption>
</figure>

In some groups (CN/EMCI), the line/points clearly indicate subject variability. This variability seems to increase in LMCI and AD.  Note that because the MMSE score is capped at 30, we may have some censoring which could cause the measure of center to shift to the left.

<p align="right">(<a href="#top">back to top</a>)</p>

### Feature Selection

`RID`, `PTID`, `EXAMDATE` have minimal-no predictive information and we will drop these values for all models.

In order to see if our data can be condensed, we tried PCA and FAMD for dimensionality reduction.

(spoilers, neither helps too much)

* PCA

```
resPCA <- PCA(df_numerical) # Note, this automatically scales the data
resPCA$eig
fviz_screeplot(resPCA)
corrplot(resPCA$var$contrib, is.corr = FALSE, col = COL2('PiYG', 20))
fviz_pca_ind(resPCA, col.ind = "contrib", 
             gradient.cols = c("#00AFBB", "#E7B800", "#FC4E07"),
             repel = TRUE)
fviz_pca_var(resPCA, col.var = "contrib", 
             gradient.cols = c("#00AFBB", "#E7B800", "#FC4E07"),
             repel = TRUE)
```
<figure>
  <img src="./img/PCAplots3.png", width = "720">
  <figcaption><b>Fig 6.</b> PCA plot of numerical variables.</figcaption>
</figure>

* FAMD

```{r}
resFAMD <- FAMD(X)
resFAMD$eig
corrplot(resFAMD$var$contrib, is.corr = FALSE, col = COL1('YlGn', 20))
fviz_famd_var(resFAMD,"var",
             repel = TRUE)
fviz_mfa_ind(resFAMD, 
             habillage = "DX", # color by groups 
             #palette = c("#00AFBB", "#E7B800", "#FC4E07"),
             addEllipses = TRUE, ellipse.type = "confidence", 
             repel = TRUE # Avoid text overlapping
             ) 
```

<figure>
  <img src="./img/FAMDplots.png", width = "720">
  <figcaption><b>Fig 7.</b> FAMD plots variables.</figcaption>
</figure>

<p align="right">(<a href="#top">back to top</a>)</p>

## Modeling

* GLM
* RF
* Neural Net?

### GLM

#### Setup
```{r}
gauss <- glm(`MMSE.Change` ~ ., data = df_clean, family = gaussian())
summary(gauss)
plot(gauss)
```

```
Call:
glm(formula = MMSE.Change ~ ., family = gaussian(), data = df_clean)

Deviance Residuals: 
     Min        1Q    Median        3Q       Max  
-17.1360   -1.1402    0.2953    1.6626    8.7894  

Coefficients:
             Estimate Std. Error t value Pr(>|t|)    
(Intercept)   0.35168    3.73327   0.094 0.925000    
AGE           0.05543    0.02408   2.302 0.021878 *  
PTEDUCAT     -0.03558    0.06161  -0.578 0.563922    
MMSE         -0.14412    0.10269  -1.403 0.161316    
PTGENDERMale  0.29308    0.33798   0.867 0.386417    
APOE41       -0.34469    0.36527  -0.944 0.345946    
APOE42       -0.42068    0.55977  -0.752 0.452810    
DXEMCI       -0.01832    0.52151  -0.035 0.971994    
DXLMCI       -1.74341    0.45599  -3.823 0.000154 ***
DXAD         -5.25577    0.75794  -6.934  1.8e-11 ***
---
Signif. codes:  0 ‘***’ 0.001 ‘**’ 0.01 ‘*’ 0.05 ‘.’ 0.1 ‘ ’ 1

(Dispersion parameter for gaussian family taken to be 10.044)

    Null deviance: 4877.1  on 383  degrees of freedom
Residual deviance: 3756.5  on 374  degrees of freedom
AIC: 1987.5

Number of Fisher Scoring iterations: 2
```

<figure>
  <img src="./img/glmGaussianPlot.png", width = "720">
  <figcaption><b>Fig 6.</b> GLM Regression Plots.</figcaption>
</figure>


#### Evaluation


<p align="right">(<a href="#top">back to top</a>)</p>

### RF

#### Setup
```{r}
rf <- randomForest(`MMSE.Change` ~ ., data = df_clean, ntrees = 500, importance = TRUE, type = 'regression')
y_pred <- predict(rf)

rf
plot(rf)
importance(rf)
varImpPlot(rf)
```

```
Call:
 randomForest(formula = MMSE.Change ~ ., data = df_clean, ntrees = 500,      importance = TRUE, type = "regression") 
               Type of random forest: regression
                     Number of trees: 500
No. of variables tried at each split: 2

          Mean of squared residuals: 10.38104
                    % Var explained: 18.26
           %IncMSE IncNodePurity
AGE      10.349901     1086.5447
PTEDUCAT  1.117041      558.6845
MMSE     15.681850      843.7943
PTGENDER  3.466778      154.5082
APOE4     2.843596      285.1642
DX       26.222985      776.4001

```

<figure>
  <img src="./img/rfPlot.png", width = "720">
  <figcaption><b>Fig 7.</b> Random Forest Regression Plots.</figcaption>
</figure>

#### Evaluation


<p align="right">(<a href="#top">back to top</a>)</p>

### Neural Net (?)

#### Setup


#### Evaluation




## Discussion

we found out 
* This thing
* And that thing

## Conclusions

* A short sentence summary of our work
* A brief aside on Limitations
* Future directions may include



<p align="right">(<a href="#top">back to top</a>)</p>

<!-- Project Contributions -->
## Project Contributions

* Austin Marckx - Austin.Marckx@UTSouthwestern.edu
  * I did this thing
  * And that thing

* Noah Chang - WooYong.Chang@UTSouthwestern.edu
  * I did this other thing
  * And that other thing

Project Link: [https://github.com/austinmarckx/2022UTSWMachineLearningFinalProj](https://github.com/austinmarckx/2022UTSWMachineLearningFinalProj)

<p align="right">(<a href="#top">back to top</a>)</p>

<!-- ACKNOWLEDGMENTS -->
## Acknowledgments

Works cited go here...

- MMSE Overview: https://oxfordmedicaleducation.com/geriatrics/mini-mental-state-examination-mmse/
- MMSE Sample Exam: https://cgatoolkit.ca/Uploads/ContentDocuments/MMSE.pdf



<p align="right">(<a href="#top">back to top</a>)</p>


<!-- MARKDOWN LINKS & IMAGES -->
<!-- https://www.markdownguide.org/basic-syntax/#reference-style-links -->
[contributors-shield]: https://img.shields.io/github/contributors/austinmarckx/2022UTSWMachineLearningFinalProj.svg?style=for-the-badge
[contributors-url]: https://github.com/austinmarckx/2022UTSWMachineLearningFinalProj/graphs/contributors
[forks-shield]: https://img.shields.io/github/forks/austinmarckx/2022UTSWMachineLearningFinalProj.svg?style=for-the-badge
[forks-url]: https://github.com/austinmarckx/2022UTSWMachineLearningFinalProj/network/members
[stars-shield]: https://img.shields.io/github/stars/austinmarckx/2022UTSWMachineLearningFinalProj.svg?style=for-the-badge
[stars-url]: https://github.com/austinmarckx/2022UTSWMachineLearningFinalProj/stargazers
[issues-shield]: https://img.shields.io/github/issues/austinmarckx/2022UTSWMachineLearningFinalProj.svg?style=for-the-badge
[issues-url]: https://github.com/austinmarckx/2022UTSWMachineLearningFinalProj/issues
[license-shield]: https://img.shields.io/github/license/austinmarckx/2022UTSWMachineLearningFinalProj.svg?style=for-the-badge
[license-url]: https://github.com/austinmarckx/2022UTSWMachineLearningFinalProj/blob/master/LICENSE.txts