
# 1. Wage analysis in Italy provinces


```r
df1 <- read_csv("./dataset/ps_2.csv")
```


## (a) Creating dummies for gender and age


```r
# Process the data
df1 = df1 %>% mutate(sex_dummy = if_else(sex == "F", 1, 0), age = year - birth)
```


## (b) Decompose the within province and between province variance

The wihin variance and between variacnce are shown as follows separately and in order.

This table briefly the source of variance are mainly within provinces. That is, the wage variation/inequality is not mainly due to the difference among provinces. Instead, it is from the variation withing provinces.


```r
within_variance_all = (df1 %>% group_by(prov) %>% summarise(within_variance = var(log_dailywages) * 
    n()))$within_variance
within_variance_all = sum(within_variance_all)

btw_variance_all = df1 %>% group_by(prov) %>% summarise(mean = mean(log_dailywages), 
    var = (mean - mean(df1$log_dailywages))^2, number = n())


btw_variance_all = sum(btw_variance_all$var * (btw_variance_all$number))

var_decomp = data.frame(`within variance` = within_variance_all, `among variance` = btw_variance_all)

stargazer(var_decomp, summary = F, header = F, type = "text", rownames = F)
```

```
## 
## ==============================
## within.variance among.variance
## ------------------------------
## 507,255.700       3,062.185   
## ------------------------------
```

Based on the figure, we can find the vairance of daily wage most result from within variance. We could conclude that more than 99% of the wage inequality is within provinces. Recently, after 1990, within variance increased and between variance decreased. Until 2000, the trend seemed to be mitigated.


```r
within_variance_yearly = df1 %>% group_by(prov, year) %>% summarise(within_variance = sum((log_dailywages - 
    mean(log_dailywages))^2))

btw_variance_yearly = c(1985:2001)
# Variance decompostion for each year
for (i in 1985:2001) {
    
    df_variance_yearly = df1 %>% filter(year == i) %>% group_by(prov) %>% summarise(mean = mean(log_dailywages), 
        var = (mean - mean((df1 %>% filter(year == i))$log_dailywages))^2, number = n())
    
    btw_variance_yearly[i - 1984] = sum(df_variance_yearly$var * (df_variance_yearly$number))
    
}
# Transform the data
within_variance_yearly = within_variance_yearly %>% group_by(year) %>% summarise(within_variance = sum(within_variance)) %>% 
    arrange(year) %>% mutate(btw_variance = btw_variance_yearly) %>% mutate(total_variance = btw_variance + 
    within_variance)
# Plot the trend of variation
ggplot(data = within_variance_yearly, aes(x = year)) + geom_point(aes(x = year, y = (within_variance/total_variance) - 
    0.99, color = "Within Variance")) + geom_line(aes(x = year, y = (within_variance/total_variance) - 
    0.99, color = "Within Variance")) + geom_line(aes(x = year, y = btw_variance/total_variance, 
    color = "BTW")) + geom_point(aes(x = year, y = btw_variance/total_variance, color = "BTW")) + 
    scale_y_continuous(limits = c(0.001, 0.01), labels = scales::percent, sec.axis = dup_axis(~. + 
        0.99, name = "Within ", labels = scales::percent), name = "Between") + theme_classic() + 
    theme(legend.title = element_blank()) + labs(title = "Variance Decompsition for each year")
```

<img src="./figure/ps2/figureunnamed-chunk-6-1.png" style="display: block; margin: auto;" />
## (c) How a province relate to the daily wage

This table is the result of for province fixed effects. To interpret the table, we can say the dailywage of labor in VR is higher than BL about 0.05 in average, based on the same age and gender. We can find VR has the largest place effect, which is 0.05 higher than BL, However, with this table, we only can observe the place province effects on wages in 1995. Thus, we refer to the next figure to see how the place effect on wage changes. 


```r
model1 = lm(data = df1, log_dailywages ~ as.factor(prov) + as.factor(year) + age + 
    I(age^2) + sex_dummy)
stargazer(model1, type = "text", omit = c(7:22), header = FALSE, covariate.labels = c("PD", 
    "RO", "TV", "VE", "VI", "VR", "Age", "Age Squared", "Sex dummy", "BL"), title = "Province fixed effect")
```

```
## 
## Province fixed effect
## ====================================================
##                           Dependent variable:       
##                     --------------------------------
##                              log_dailywages         
## ----------------------------------------------------
## PD                              0.018***            
##                                 (0.001)             
##                                                     
## RO                             -0.109***            
##                                 (0.001)             
##                                                     
## TV                              -0.002*             
##                                 (0.001)             
##                                                     
## VE                              0.010***            
##                                 (0.001)             
##                                                     
## VI                              0.006***            
##                                 (0.001)             
##                                                     
## VR                              0.050***            
##                                 (0.001)             
##                                                     
## Age                             0.044***            
##                                 (0.0001)            
##                                                     
## Age Squared                    -0.0005***           
##                                (0.00000)            
##                                                     
## Sex dummy                      -0.270***            
##                                 (0.0004)            
##                                                     
## BL                              3.896***            
##                                 (0.003)             
##                                                     
## ----------------------------------------------------
## Observations                   3,033,744            
## R2                               0.189              
## Adjusted R2                      0.189              
## Residual Std. Error       0.369 (df = 3033718)      
## F Statistic         28,193.550*** (df = 25; 3033718)
## ====================================================
## Note:                    *p<0.1; **p<0.05; ***p<0.01
```
We can see the place effect of VR is still the largest among most of the time. Then, we can say that VR drives up wages the most.
Also, we find that place effect was decreasing over the time.

```r
df1 = df1 %>% arrange(prov)
place = cbind(c(unique(df1$prov))[2:7], model1$coefficients[2:7], c(rep("total", 
    6)))
# Run the regression for each year
for (i in 1985:2001) {
    
    df1_year = df1 %>% filter(year == i)
    model_year = lm(data = df1_year, log_dailywages ~ as.factor(prov) + age + I(age^2) + 
        sex_dummy)
    model_year$coefficients[2:7] = model_year$coefficients[2:7] + model_year$coefficients[1]
    model_year_coef = cbind((unique(df1$prov))[1:7], model_year$coefficients[1:7], 
        c(rep(i, 7)))
    
    place = rbind(place, model_year_coef)
    
}
# Create the table for place fixed effect in each year
place = as.data.frame(place) %>% filter(V3 != "total")
colnames(place) = c("Province", "Place_Effect", "Year")
place$Place_Effect = as.numeric(as.character(place$Place_Effect))
place$Year = as.numeric(as.character(place$Year))
# Plot
ggplot(data = place, aes(x = Year)) + geom_point(aes(y = Place_Effect, color = Province)) + 
    geom_line(aes(y = Place_Effect, color = Province)) + labs(y = "Place Effect", 
    title = "Province effect over time") + theme_classic()
```

<img src="./figure/ps2/figureunnamed-chunk-8-1.png" style="display: block; margin: auto;" />


## (d) Visualization

This figure is a visualization of the line graph. We can find place effect decreased for all provinces here. It clearly shows there is less place effect in 2000 than 1990.


```r
# Import the map dataframe
italy_map <- map_data("italy")
italy_map = italy_map  %>%
  filter(region %in% 
           c("Belluno","Padova","Rovigo",
             "Treviso","Venezia","Vicenza","Verona")) %>%
  mutate(region = 
           case_when(
             region == "Belluno" ~ "BL",
             region == "Padova"  ~ "PD",
             region == "Rovigo" ~ "RO",
             region == "Treviso" ~ "TV",
             region == "Venezia" ~ "VE",
             region == "Vicenza" ~ "VI",
             region == "Verona" ~ "VR"
             
           ))
map_mean = italy_map %>%
  group_by(region) %>%
  summarize(long = mean(long),lat = mean(lat))
place = place %>%
  filter( Year %in% c(1990,2000)) %>%
  left_join(map_mean,by = c("Province" = "region"))
# Plot
ggplot() + 
  geom_map(data=italy_map, map=italy_map,
                      aes(long, lat, map_id=region),
                      color="#b2b2b2", size=0.1, fill=NA)+
                   geom_map(data=place, map=italy_map,
                      aes(fill=Place_Effect, map_id=Province),
                      color="#b2b2b2", size=0.1)+
    geom_text(data = place, aes(long,lat,label = Province), 
               color = "#BD9355",size = 4, hjust = 1, vjust = -0.5)+
    facet_wrap(~Year)+
    theme_classic()+
    theme(
      axis.text.x = element_blank(),
      axis.text.y = element_blank(),
      axis.line.x = element_blank(),
      axis.line.y = element_blank(),
      axis.ticks.x= element_blank(),
      axis.ticks.y= element_blank()
    )+
    labs(x = "", y="", fill = "Place Effect")+
    scale_fill_continuous( low = '#70c1b3', high = '#1d3557' ,
    space = "Lab", na.value = "grey50", guide = "colourbar", 
    breaks = c(3.75,3.85,3.95) )
```

<img src="./figure/ps2/figureunnamed-chunk-9-1.png" style="display: block; margin: auto;" />

# 2. Instrument variable and Omitted Variable Bias


```r
# Import data
df4 = read.table("./dataset/colonies.out", header = T)
```

## (a) The relationship between institution quality and gdp


```r
model = lm(data = df4, lgdp ~ inst + colony + tropics)
stargazer(model, type = "text", header = FALSE)
```

```
## 
## ===============================================
##                         Dependent variable:    
##                     ---------------------------
##                                lgdp            
## -----------------------------------------------
## inst                         4.479***          
##                               (0.422)          
##                                                
## colony                        -0.108           
##                               (0.142)          
##                                                
## tropics                       -0.259*          
##                               (0.151)          
##                                                
## Constant                     5.059***          
##                               (0.383)          
##                                                
## -----------------------------------------------
## Observations                    97             
## R2                             0.754           
## Adjusted R2                    0.746           
## Residual Std. Error       0.530 (df = 93)      
## F Statistic           95.092*** (df = 3; 93)   
## ===============================================
## Note:               *p<0.1; **p<0.05; ***p<0.01
```

This table suggests that institution quality and gdp have significantly positive relationship. However, this does not mean institution quality has positive causal effect on log gdp.

## (b) Conditional mean of island with different background


```r
df4_summary = df4 %>% group_by(colony, tropics) %>% summarize(`mean of lgdp` = round(mean(lgdp), 
    3), `mean of institution quality` = round(mean(inst), 3)) %>% ungroup() %>% mutate(colony = if_else(colony == 
    1, "Yes", "No"), tropics = if_else(tropics == 1, "Yes", "No"))
stargazer(df4_summary, type = "text", summary = FALSE, header = FALSE)
```

```
## 
## =========================================================
##   colony tropics mean of lgdp mean of institution quality
## ---------------------------------------------------------
## 1   No     No       8.931                0.866           
## 2   No     Yes      8.346                0.783           
## 3  Yes     No       8.593                0.811           
## 4  Yes     Yes      7.355                0.595           
## ---------------------------------------------------------
```

The condiational means of lgdp and inst for the combination of colony and tropics are shown in the table. For example, the first row shows the average gdp and the average institution quality of the islands which was colonized and is in tropics area.

## (c) OLS on covariates


```r
model1 = lm(data = df4, lgdp ~ colony + tropics + col_trop)
model2 = lm(data = df4, inst ~ colony + tropics + col_trop)
stargazer(model1, model2, type = "text", header = FALSE)
```

```
## 
## ==========================================================
##                                   Dependent variable:     
##                               ----------------------------
##                                    lgdp          inst     
##                                    (1)            (2)     
## ----------------------------------------------------------
## colony                            -0.339        -0.054    
##                                  (0.232)        (0.038)   
##                                                           
## tropics                           -0.585        -0.082    
##                                  (0.417)        (0.069)   
##                                                           
## col_trop                          -0.652        -0.133*   
##                                  (0.468)        (0.077)   
##                                                           
## Constant                         8.931***      0.866***   
##                                  (0.147)        (0.024)   
##                                                           
## ----------------------------------------------------------
## Observations                        97            97      
## R2                                0.468          0.492    
## Adjusted R2                       0.450          0.476    
## Residual Std. Error (df = 93)     0.780          0.128    
## F Statistic (df = 3; 93)        27.228***      30.074***  
## ==========================================================
## Note:                          *p<0.1; **p<0.05; ***p<0.01
```

## (d) instrument regression

The result in the following table shows IV estimation. The coefficient of inst ($\gamma$) is the same as we did in (b) and (c)


```r
model_2SLS = ivreg(data = df4, lgdp ~ inst + colony + tropics | col_trop + colony + 
    tropics)
stargazer(model_2SLS, type = "text", header = FALSE, title = "2 stage least square")
```

```
## 
## 2 stage least square
## ===============================================
##                         Dependent variable:    
##                     ---------------------------
##                                lgdp            
## -----------------------------------------------
## inst                          4.888**          
##                               (2.395)          
##                                                
## colony                        -0.072           
##                               (0.250)          
##                                                
## tropics                       -0.182           
##                               (0.469)          
##                                                
## Constant                      4.699**          
##                               (2.107)          
##                                                
## -----------------------------------------------
## Observations                    97             
## R2                             0.752           
## Adjusted R2                    0.744           
## Residual Std. Error       0.533 (df = 93)      
## ===============================================
## Note:               *p<0.1; **p<0.05; ***p<0.01
```

## (e) 2SLS regression


```r
fit_inst = fitted.values(model2)
df4 = cbind(df4, fit_inst)
model_sec = lm(data = df4, lgdp ~ fit_inst + colony + tropics)
stargazer(model2, model_sec, type = "text", title = "second stage", header = FALSE)
```

```
## 
## second stage
## ==========================================================
##                                   Dependent variable:     
##                               ----------------------------
##                                    inst          lgdp     
##                                    (1)            (2)     
## ----------------------------------------------------------
## fit_inst                                         4.888    
##                                                 (3.507)   
##                                                           
## colony                            -0.054        -0.072    
##                                  (0.038)        (0.366)   
##                                                           
## tropics                           -0.082        -0.182    
##                                  (0.069)        (0.687)   
##                                                           
## col_trop                         -0.133*                  
##                                  (0.077)                  
##                                                           
## Constant                         0.866***        4.699    
##                                  (0.024)        (3.085)   
##                                                           
## ----------------------------------------------------------
## Observations                        97            97      
## R2                                0.492          0.468    
## Adjusted R2                       0.476          0.450    
## Residual Std. Error (df = 93)     0.128          0.780    
## F Statistic (df = 3; 93)        30.074***      27.228***  
## ==========================================================
## Note:                          *p<0.1; **p<0.05; ***p<0.01
```

The table above shows the coefficient of institution quality is the same as (d), but the standard error is different. It's because when we are doing 2SLS separately in two stages, we misidentify the error term. Thus, we care about the variance in (d) more. That is, one unit of institution quality increases the log gdp by 4.888 siginifcantly, which is larger than the estimates obtained from simple oLS.

# 3. IV with Heterogeneous Effects


```r
# Import data
df5 <- read_csv("./dataset/iv_ps2.csv")
```

## (a) Overidentification and wald estimate


```r
df5_wald_Z1 = df5 %>% group_by(Z1) %>% summarise(Y = mean(Y), Treatment = mean(T)) %>% 
    ungroup()

Wald_Z1 = (df5_wald_Z1$Y[2] - df5_wald_Z1$Y[1])/(df5_wald_Z1$Treatment[2] - df5_wald_Z1$Treatment[1])

df5_wald_Z2 = df5 %>% group_by(Z2) %>% summarise(Y = mean(Y), Treatment = mean(T)) %>% 
    ungroup()

Wald_Z2 = (df5_wald_Z2$Y[2] - df5_wald_Z2$Y[1])/(df5_wald_Z2$Treatment[2] - df5_wald_Z2$Treatment[1])

wald = data_frame(round(Wald_Z1, 2), round(Wald_Z2, 2))

stargazer(wald, summary = FALSE, type = "text", covariate.labels = c("Wald estimate (Z1)", 
    "Wald estimate (Z2)"), rownames = FALSE, header = FALSE)
```

```
## 
## =====================================
## Wald estimate (Z1) Wald estimate (Z2)
## -------------------------------------
## 0.98                      0.51       
## -------------------------------------
```

Overidentification are showns as follow:


```r
model_sargan = ivreg(data = df5, Y ~ T | Z1 + Z2)
sargan = summary(model_sargan, diagnostic = TRUE)$diagnostics
stargazer(sargan, type = "text", header = FALSE, title = "sargan test")
```

```
## 
## sargan test
## ============================================
##                  df1  df2  statistic p-value
## --------------------------------------------
## Weak instruments  2  9,997  656.501     0   
## Wu-Hausman        1  9,997   8.555    0.003 
## Sargan            1         12.032    0.001 
## --------------------------------------------
```

We reject the null hypothesis because the p-value of sargan test is 0.001, 
so we reject the null hypothesis. However, since this problem is about
heterogeneous treatment effects, rejecting Sargan test means that either one of
the IVs is invalid, or they are identifying distinct LATEs.

## (b) ratio of the compliers of treatments


```r
df5 = df5 %>% mutate(XT = X * T)
# complier
complier1 = ivreg(XT ~ T | Z1, data = df5)
complier2 = ivreg(XT ~ T | Z2, data = df5)
# output
stargazer(complier1, complier2, summary = FALSE, rownames = FALSE, type = "text", 
    header = FALSE)
```

```
## 
## ============================================================
##                                     Dependent variable:     
##                                 ----------------------------
##                                              XT             
##                                      (1)            (2)     
## ------------------------------------------------------------
## T                                  0.881***      0.168***   
##                                    (0.034)        (0.032)   
##                                                             
## Constant                          -0.172***      0.187***   
##                                    (0.017)        (0.017)   
##                                                             
## ------------------------------------------------------------
## Observations                        10,000        10,000    
## R2                                  0.219          0.193    
## Adjusted R2                         0.219          0.193    
## Residual Std. Error (df = 9998)     0.393          0.400    
## ============================================================
## Note:                            *p<0.1; **p<0.05; ***p<0.01
```

By Abadie(2002), the mean of compliers is the coefficient of T in the following Table. The coefficient of T in column(1) and column (2) is the mean for complier complied with Z1 and Z2 separately.  They are 0.881 and 0.168 separately.

## (c) the CDFs of potential outcomes, Yi(1) and Yi(0)


```r
test = seq(-3.4, 3.8, by = 0.01)
Z1_treated = seq(-3.4, 3.8, by = 0.01)
for (i in 1:721) {
    df5_test = df5 %>% mutate(Dep_var = if_else(Y <= test[i], 1, 0), Dep_T = Dep_var * 
        T, untreated = 1 - T)
    model1 = ivreg(Dep_T ~ T | Z1, data = df5_test)
    Z1_treated[i] = model1$coefficients[2]
}
for (i in 1:721) {
    Z1_treated[i] = if_else(Z1_treated[i] <= 1, Z1_treated[i], 1)
}
Z2_treated = seq(-3.4, 3.8, by = 0.01)
for (i in 1:721) {
    df5_test = df5 %>% mutate(Dep_var = if_else(Y <= test[i], 1, 0), Dep_T = Dep_var * 
        T, untreated = 1 - T)
    model1 = ivreg(Dep_T ~ T | Z2, data = df5_test)
    Z2_treated[i] = model1$coefficients[2]
}

for (i in 1:721) {
    Z2_treated[i] = if_else(Z2_treated[i] <= 1, Z2_treated[i], 1)
}
Z1_untreated = seq(-3.4, 3.8, by = 0.01)
for (i in 1:721) {
    df5_test = df5 %>% mutate(Dep_var = if_else(Y <= test[i], 1, 0), untreated = 1 - 
        T, Dep_T = Dep_var * untreated)
    model1 = ivreg(Dep_T ~ T | Z1, data = df5_test)
    Z1_untreated[i] = -model1$coefficients[2]
}
for (i in 1:721) {
    Z1_untreated[i] = if_else(Z1_untreated[i] <= 1, Z1_untreated[i], 1)
}
Z2_untreated = seq(-3.4, 3.8, by = 0.01)
for (i in 1:721) {
    df5_test = df5 %>% mutate(Dep_var = if_else(Y <= test[i], 1, 0), untreated = 1 - 
        T, Dep_T = Dep_var * untreated)
    model1 = ivreg(Dep_T ~ T | Z2, data = df5_test)
    Z2_untreated[i] = -model1$coefficients[2]
}

for (i in 1:721) {
    Z2_untreated[i] = if_else(Z2_untreated[i] <= 1, Z2_untreated[i], 1)
}

ggplot() + geom_point(aes(x = test, y = Z1_treated, color = "Z1:Y(1)"), size = 0.2, 
    alpha = 0.2) + geom_point(aes(x = test, y = Z2_treated, color = "Z2:Y(1)"), size = 0.2, 
    alpha = 0.2) + geom_point(aes(x = test, y = Z1_untreated, color = "Z1:Y(0)"), 
    size = 0.2) + geom_point(aes(x = test, y = Z2_untreated, color = "Z2:Y(0)"), 
    size = 0.2) + labs(x = "Y", y = "Probability", title = "CDF of potential outcome for compliers with instrument") + 
    theme_classic() + guides(colour = guide_legend(override.aes = list(size = 3))) + 
    scale_color_discrete(breaks = c("Z1:Y(0)", "Z2:Y(0)", "Z2:Y(1)", "Z1:Y(1)")) + 
    theme(legend.title = element_blank(), legend.key = element_blank())
```

<img src="./figure/ps2/figureunnamed-chunk-20-1.png" style="display: block; margin: auto;" />


The CDFs is the potential outcomes, Yi(1) and Yi(0), for individuals that comply with each instrument.







