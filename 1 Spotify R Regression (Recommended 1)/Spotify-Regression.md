---
title: "Spotify Regression"
author: "Tonatiuh De Leon"
output: 
  html_document:
    theme: default
    code_folding: hide
    keep_md: true
editor_options: 
  chunk_output_type: console
---
 

 

```r
Sys.setenv(SPOTIFY_CLIENT_ID = '857bab10601a4bc2af19d359336d3af7')
Sys.setenv(SPOTIFY_CLIENT_SECRET = 'f36a14960bc445fba551b36f3b65cd65')
url <- "https://kworb.net/spotify/songs.html"
url1 <- "https://kworb.net/spotify/artists.html"
url2 <- "https://kworb.net/spotify/listeners.html"
#access_token <- get_spotify_access_token()
 
#all_artists_info <- search_spotify("all", type = "artist")
webpage <- read_html(url)
table <- html_table(webpage)[[1]]
webpage1 <- read_html(url1)
table1 <- html_table(webpage1)[[1]]
webpage2 <- read_html(url2)
table2 <- html_table(webpage2)[[1]]
 
df <- as.data.frame(table) |> 
  rename(title = `Artist and Title`) |> 
  separate(title, into = c("artist", "title"), sep = " - ") |> 
  rename(streams = Streams,
         daily = Daily) |> 
  mutate(streams = as.numeric(gsub(",", "", streams)),
         daily = as.numeric(gsub(",", "", daily))) |> 
  drop_na()
 
 
df1 <- as.data.frame(table1) |>  
  rename(artist_streams = Streams,
         artist = Artist,
         daily1 = Daily) |> 
  mutate(artist_streams = as.numeric(gsub(",", "", artist_streams))) |> 
  dplyr::select(artist, artist_streams, daily1) |> 
  drop_na()
 
df2 <- as.data.frame(table2) |>  
  rename(monthly_listeners = Listeners,
         peak_monthly_listeners = PkListeners,
         months_sincepk = Peak,
         daily_average = `Daily Trend`,
         artist = Artist) |> 
  mutate(monthly_listeners = as.numeric(gsub(",", "", monthly_listeners)),
         peak_monthly_listeners = as.numeric(gsub(",", "", peak_monthly_listeners)),
         daily_average = as.numeric(gsub(",", "", daily_average)))|> 
  dplyr::select(artist, monthly_listeners, peak_monthly_listeners, months_sincepk, daily_average) |> 
  drop_na()
 

dta <- read_csv("https://github.com/Tonadeleon/Regression-Projects/raw/main/Datasets/dataset.csv")|> 
  #filter(popularity > 20) |> 
  rename(title = track_name,
         artist = artists,
         title_id = track_id,
         album = album_name,
         loud = loudness,
         words = speechiness,
         acoustics = acousticness,
         instrumental = instrumentalness,
         live = liveness) |> 
  inner_join(df1, by="artist", relationship =
  "many-to-many") |> 
  inner_join(df2, by="artist", relationship =
  "many-to-many")
 
 
dt <- inner_join(dta, df, by = c("title", "artist"), relationship =
  "many-to-many") |> 
  dplyr::select(-track_genre) |> 
  mutate(explicit = as.factor(explicit),
         daily = daily,
         #streams = ifelse(streams>3825666400, 942193161, streams),
         #times = as.factor(time_signature),
         #energy = ifelse(energy>.5,1,0),
         #duration = ifelse(between(duration_ms, 200000, 250000), 1, 0),
         #loud = ifelse(loud>-9, 1, 0),
         dance = ifelse(danceability>.5,1,0),
         words = ifelse(between(words, .025, .06), 1, 0),
         instrumental = ifelse(instrumental>0,1,0),
         #popularity = ifelse(popularity>76,1,0),
         live = ifelse(live<.37,1,0),
         key = case_when(key %in% c(0, 1, 6, 8, 11) ~ 1,TRUE ~ 0),
         explicit = case_when(explicit == "TRUE" ~ 1,TRUE ~ 0), 
         #genre = case_when(genre %in% c("pop", "chill", "indie", "garage", "ambient", "folk", "indie-pop", "power-pop", "reggae", "soul", "synth-pop", "swedish", "country", "metal", "german", "funk", "alternative", "alt-rock") ~ 1,TRUE ~ 0),
         peak_last_year = ifelse(months_sincepk<13,1,0),
         y = ifelse(popularity >79 ,1 ,0),
         popularity0 = round(popularity / 5) * 5,
         popularity0 = case_when( popularity0 > 84 ~ 1, popularity == 80 ~ 2, TRUE ~ 3 ),
         popularity1 = round(popularity / 10) * 10,
         popularity1 = case_when( popularity1 == 0 ~ 1, popularity1 == 70 ~ 2, popularity1 == 80 ~ 3,  popularity1 == 90 ~ 4, TRUE ~ 5 ),
         times = case_when(time_signature %in% c(4) ~ 1,TRUE ~ 0),
         energy = round(energy*10),
         popular_artist = case_when(artist_streams > 75000 ~ 1, between(artist_streams, 50000,75000) ~ 2, between(artist_streams, 25000,50000) ~ 3, artist_streams < 25000 ~ 4, TRUE ~ 0 ),
         diff = peak_monthly_listeners - monthly_listeners,
         current_hit = ifelse(diff>20000000,1,0),
         current_trend = ifelse(daily_average<0,0,1)) |> 
  

   distinct(across(c(daily1, monthly_listeners, artist_streams, streams, months_sincepk, explicit, daily, popular_artist))) |> 
  
   dplyr::select(daily, daily1, monthly_listeners, artist_streams, streams, months_sincepk, explicit,  popular_artist)
 

 
 
dt$daily1 <- log(dt$daily1)
dt$monthly_listeners <- log(dt$monthly_listeners)
dt$artist_streams <- log(dt$artist_streams)
dt$streams <- log(dt$streams)
dt$daily <- log(dt$daily)


 
 
mylm <- lm((daily1) ~ 
             
             explicit +
             
             ###
             
             (monthly_listeners) + 
             
             (artist_streams) + 
             
             (streams)
             
             
, dt)


 
b <-(coef(mylm))

#summary(mylm)
```
 
# {.tabset .tabset-fade .tabset-pills}
 
##  Introduction

### {.tabset .tabset-fade .tabset-panel}

#### Actual Model Representation
 
<br>
 

```r
ggplot(dt, aes(y=(daily1), x=(monthly_listeners)))+
  geom_point(show.legend = F, col="orange")+
  #geom_smooth( col="skyblue", method = "lm", formula = "(y)~(x)")+
  
  annotate("text", x = log(50000000-5000000), y = log(.75), 
           label = "Subset being measured:", color = "grey35", hjust = -0.1, size=3.2) +
  
  stat_function(fun = function(x) 
    (b[1] + b[2] * 0 + b[4] * 8.156 + b[5] * 20.17) + 
    (b[3]) * x, linewidth = 1, col = "skyblue1") +
  annotate("text", x = log(50000000-1000000), y = log(.38), 
           label = "2nd Quartile", color = "skyblue1", hjust = -0.1, size=3.2) +

  stat_function(fun = function(x) 
    (b[1] + b[2] * 0 + b[4] * 9.030 + b[5] * 20.55) + 
    (b[3]) * x, linewidth = 1, col = "skyblue3") +
  annotate("text", x = log(50000000-500000), y = log(.48), 
           label = "Median", color = "skyblue3", hjust = -0.1, size=3.2) +

  stat_function(fun = function(x) 
    (b[1] + b[2] * 0 + b[4] * 9.916 + b[5] * 20.86) + 
    (b[3]) * x, linewidth = 1, col = "steelblue4") +
  annotate("text", x = log(50000000-1000000), y = log(.6), 
           label = "3rd Quartile", color = "steelblue4", hjust = -0.1, size=3.2) +
  
  scale_x_continuous(breaks = c(log(4500000), log(10000000), log(25000000), log(50000000), log(100000000)), 
                     labels = c("4.5 M", "10 M", "25 M", "50 M" ,"100 M")) +
  scale_y_continuous(breaks = c(log(.5), log(1.5), log(6), log(20), log(70)), 
                     labels = c(".5 M", "1.5 M", "6 M", "20 M" ,"70 M")) +
  theme_classic() +
  labs( 
    title = "Artist Monthly Followers vs. Artist Daily Streams", 
    subtitle = "These lines represent a 2D version of the actual high dimensional model",
    x= "Artist Monthly Followers",
    y = "Artist Daily Streams",
    caption= "Log scale") +
  theme(
    plot.title = element_text(size = 14, color = "grey15"),
    axis.text.x = element_text(color = "grey35", size = 10),
    axis.text.y = element_text(color = "grey35", size = 10),
    plot.subtitle = element_text(color = "grey35", size = 11),
    axis.title = element_text(color = "grey15", size = 12))
```

<img src="Spotify-Regression_files/figure-html/graphs actual-1.png" style="display: block; margin: auto;" />

<br>

$$ \text{Regression Equation :} \\\ \\  \text{Artist Daily Streams}_{Y_i} = \\\ \ \text{b0} + \text{Monthly followers}_{b_1} + \text{Explicit (or not)}_{b_2} + \text{All time artist streams}_{b_3} + \\\ \ \text{All time streams per song}_{b_4} $$

<br>

You may be wondering why the least square lines of this model look as if they're not fitting the data very well. Maybe I did a bad job graphing the model? or maybe the model is not that good? It was interesting to me at first. Considering that we're used to the **simple idea** (see the other tab) that the line should be directly in the middle of the points; I was shocked when I realized that even when these lines look as if they're not fitting the data well enough, they're actually a correct representation of this model.

This is because this model has many dimensions to it. Differently from the simple idea where only two dimensions are considered, here we are looking at 3 and some subgroups that could make it 4 dimensional. Because of this, there is not a "one line fits all scenarios" case in this model, but actually there are infinite least squares lines according to different scenarios. A couple examples are shown in the graph above. The 3 least square lines shown in the graph portray subsets of the data where all time artist streams, and all time streams per song meet their respective quartiles as examples (see graph legend). In other words, what if my artist has "x" amount of followers, and "z" amount of all time streams, and has a song with "q" reproductions. Well there will certainly be a line for that type of artist and if many fall into that category we would be able to predict their daily streams.  

This is a high dimensional regression. The actual model (1st graph) and their near to infinite possible scenarios could be overwhelming to understand, that's why from this point on in this analysis, all details for any scenario being considered will be disclosed in the different tabs of the analysis.
 
This analysis will explain the relationship between artist monthly followers and their daily streams on Spotify, we could estimate their daily revenue as well; Spotify does not share all of its information publicly, (no revenue data) only some variables were available to perform a regression model on this data. This is why more research was needed in order to find variables. By looking up related Spotify data I came across with data that not only had information on how the song was made (duration, decibel levels, etc..), but it also had Monthly followers for more than 3000 artists as well as daily streamings, and other interesting variables which led me to continue an analysis on it.(See model selection tab) 

If the correlation is proven true, then an artist can focus their efforts on getting more followers so their streams go up and thus their [earnings](https://dittomusic.com/en/blog/how-much-does-spotify-pay-per-stream#:~:text=Spotify%20pays%20artists%20between%20%240.003,holders%20and%2030%25%20to%20Spotify.).

Thus, this analysis will consider the next hypothesis:

<br>

$$
\left.\begin{array}{ll}
H_0: \beta_1 = 0 \\  
H_a: \beta_1 \neq 0
\end{array}
\right\} \ \text{Slope Hypotheses} \\ \text{alpha level:} \ .05
$$

<br>

The null hypothesis being a slope of zero will help us test if the correlation is non existent. IF the slope of this model was zero, then no correlation would be found between these two variables. A T-test will be performed to test for a significant difference between zero and the actual slope of the model. If the results are significant at the **alpha level of .05**, Then it would be safe to assume that correlation has been found between these two variables and in the model as well.

Such hypothesis can be tested by using the slope coefficient of the model summary. However, it's crucial to understand that even when there are almost infinite lines in this model, all of them share the same slope. So, for each specific scenario that we try to analyze there will be a single slope (as seen in the first graph). There's only one slope when we test monthly followers as the main predictor because all of these lines are only shifting in their intercepts.

There would be different slopes if we tested a null hypothesis on the rest of the variables. Since, Monthly followers is the best predictor in this model, we will stick to it and test the null at the **.05 alpha level.**

To start the hypothesis testing, a corresponding T value is needed for the comparison between our selected line slope, and the slope that is being hypothesized which is of 0, meaning no correlation. After that, a P value for the slope in the model can be calculated while is tested against the null hypothesis. For this we will use the artist monthly followers slope.
 
<br>
 

```r
tvalue <- ((b[3]) - 0)/(0.03272)
result <- data.frame(a = round(pt(-abs(tvalue), 876 ) * 2, 5), row.names = NULL)
kable(result, align = "c", col.names = 'P. Value when 0 streams increase per new monthly follower is the null ')
```



| P. Value when 0 streams increase per new monthly follower is the null  |
|:----------------------------------------------------------------------:|
|                                   0                                    |

<br>

As we can see, there is a significant difference between the model's estimated slope for this correlation and zero. It is then safe to assume that, it is reasonable to explain the amount of monthly streams per artist (which can lead to more earnings) by their monthly followers. 

Spotify's follower count increases their monthly streams. What I mean by this is that the slope of these lines actually mean streams per one follower gained in a month. So, in this case there is an increase of **3** streams per follower in a monthly basis. This slope is being tested to the 0 or no additional stream per new monthly follower, and the results indicate that the slope of the model is of significant value.

Keep on reading to see more on how the data was worked up in order to get to these results.
 
<br><br><br><br>

#### Simple Idea
 
<br>
 

```r
ggplot(dt, aes(y=(daily1), x=(monthly_listeners)))+
  geom_point(show.legend = F, col="orange")+
  geom_smooth( col="skyblue", method = "lm", formula = "(y)~(x)") +
  
  scale_x_continuous(breaks = c(log(4500000), log(10000000), log(25000000), log(50000000), log(100000000)), labels=c("4.5 M","10 M","25 M","50 M" ,"100 M")) +
  
   scale_y_continuous(breaks = c(log(.5), log(1.5), log(6), log(20), log(70)), labels=c(".5 M","1.5 M","6 M","20 M" ,"70 M"))+
 
  theme_classic() +
  labs( 
    title = "Artist Monthly Followers vs. Artist Daily Streams", 
    subtitle = "March 2023, Spotify Measurements",
    x= "Artist Monthly Followers",
    y = "Artist Daily Streams",
    caption= "Log scale") +
  theme(
    plot.title = element_text(size = 13, color = "grey30"),
    axis.text.x = element_text(color = "grey45"),
    axis.text.y = element_text(color = "grey45"),
    plot.subtitle = element_text(color = "grey35", size = 10),
    axis.title = element_text(color = "grey35", size = 10))
```

<img src="Spotify-Regression_files/figure-html/graphs simple-1.png" style="display: block; margin: auto;" />
 
<br>

While this is not the actual model being used, it gives an idea of how the model would look like if it was 2D only, it also portrays how linear the data is and looks when transformed with log scales on x and y. However, a better prediction can be achieved when using the rest of the variables shown in the actual model.

<br><br><br><br>

###
 
## Model Selection

<br>

### {.tabset .tabset-fade .tabset-panel}
 
#### Data Collection
 
<br>
 
I found many interesting data variables for Spotify, some of them I found in [Kaggle](https://www.kaggle.com/code/adrianograms/spotify-regression/input) and the rest I found in [Kworb's Project](https://kworb.net/) on the Spotify section. Here are the variables I was able to find that are available in Spotify, other than artist's name, albums, and songs;

**Kaggle:**

1. <span style="color: #0047AB;">**key** - if certain Keys were present in the song</span>

2. <span style="color: #0047AB;">**loud** - song's decibel level</span>

3. <span style="color: #0047AB;">**mode** - major or minor scale</span>

4. <span style="color: #0047AB;">**words** - are there too many words in the song?</span>

5. <span style="color: #0047AB;">**acoustics** - level of acoustics</span>

6. <span style="color: #0047AB;">**instrumental** - level of instrumentality</span>

7. <span style="color: #0047AB;">**live** - crowd noises present</span>

8. <span style="color: #0047AB;">**valence** - level of joy</span>

9. <span style="color: #0047AB;">**tempo** - beats per minute</span>

10. <span style="color: #0047AB;">**time_signature** - 3/4, 4/4, etc..</span>

11. <span style="color: #0047AB;">**track_genre** - genre</span>

<br>

**Kworb's Project:**

<br>

12. <span style="color: #0047AB;">**daily** - Monthly streams per song</span>

13. <span style="color: #0047AB;">**daily1** - daily streams per artist</span>

14. <span style="color: #0047AB;">**monthly_listeners** - number of followers per artist</span>

15. <span style="color: #0047AB;">**peak_monthly_listeners** - max number of followers the artist had</span>

16. <span style="color: #0047AB;">**months_sincepk** - number of months since peak monthly listeners</span>

17. <span style="color: #0047AB;">**daily_average** - amount of followers the artist is getting or losing</span>

<br>

More variables were included in the final data set, and after deep data analysis more were found. By pure logic we can infer that number of followers will be important, as well as number of streams. But to be sure about this, multiple pairs plot were used. Here are a couple of them.

<br><br><br><br>
 
#### Model Selection
 
<br>
 
##### {.tabset .tabset-fade .tabset-pills}
 
###### Original Pairs Plot

<br>

Take a look to the following pairs plot to understand more about the process of selecting this final model. Consider which variables have a larger correlation; Since no monetary values were available, I looked for other types of correlation and tried to keep my model business/industry related. I looked for variables which were related in order to get good insights for spotify artists.

Originally, not many correlations were visible. A little on daily and monthly_listeners can be visible but, it's not too obvious. But was enough to get me started on trying to find something there. At the same time, the relation between streams and daily looked good to me. This is why I tried both models. Take a look at my alternative model if you're curious about it.

<br>


```r
dto <- inner_join(dta, df, by = c("title", "artist"), relationship = "many-to-many") |> 
  mutate(explicit = as.factor(explicit),
         track_genre = as.factor(track_genre)) |> 
  dplyr::select(daily, daily1, everything()) |> 
  dplyr::select(-c("...1", title_id, artist, album, title))
pairs(dto, panel=panel.smooth)
```

<img src="Spotify-Regression_files/figure-html/original pairs plot-1.png" style="display: block; margin: auto;" />
 
<br><br><br><br>

###### Wrangled Pairs Plot

<br>

After wrangling the data, and creating new variables such as current_trend, peaked_last_year and others in where I create some on-off switches, as well as multiple categorical variables I got many good insights as to what to do next. This pairs plot helped me view the correlation between daily1 and monthly_listeners and that got me my first raw R^2^ of around .7 Then I tried transforming the data.


```r
dta1 <- read_csv("https://github.com/Tonadeleon/Regression-Projects/raw/main/Datasets/dataset.csv")|> 
  #filter(popularity > 59) |> 
  rename(title = track_name,
         artist = artists,
         title_id = track_id,
         album = album_name,
         loud = loudness,
         words = speechiness,
         acoustics = acousticness,
         instrumental = instrumentalness,
         live = liveness) |> 
  inner_join(df1, by="artist", relationship =
  "many-to-many") |> 
  inner_join(df2, by="artist", relationship =
  "many-to-many")


dtal <- inner_join(dta1, df, by = c("title", "artist"), relationship =
  "many-to-many") |> 
  mutate(explicit = as.factor(explicit),
         genre = as.factor(track_genre),
         track_genre = as.factor(track_genre),
         daily = daily,
         #streams = ifelse(streams>3825666400, 942193161, streams),
         #times = as.factor(time_signature),
         #energy = ifelse(energy>.5,1,0),
         #duration = ifelse(between(duration_ms, 200000, 250000), 1, 0),
         #loud = ifelse(loud>-9, 1, 0),
         dance = ifelse(danceability>.5,1,0),
         words = ifelse(between(words, .025, .06), 1, 0),
         instrumental = ifelse(instrumental>0,1,0),
         #popularity = ifelse(popularity>76,1,0),
         live = ifelse(live<.37,1,0),
         key = case_when(key %in% c(0, 1, 6, 8, 11) ~ 1,TRUE ~ 0),
         explicit = case_when(explicit == "TRUE" ~ 1,TRUE ~ 0), 
         genre = case_when(genre %in% c("pop", "chill", "indie", "garage", "ambient", "folk", "indie-pop", "power-pop", "reggae", "soul", "synth-pop", "swedish", "country", "metal", "german", "funk", "alternative", "alt-rock") ~ 1,TRUE ~ 0),
         peak_last_year = ifelse(months_sincepk<13,1,0),
         y = ifelse(popularity >79 ,1 ,0),
         popularity0 = round(popularity / 5) * 5,
         popularity0 = case_when( popularity0 > 84 ~ 1, popularity == 80 ~ 2, TRUE ~ 3 ),
         popularity1 = round(popularity / 10) * 10,
         popularity1 = case_when( popularity1 == 0 ~ 1, popularity1 == 70 ~ 2, popularity1 == 80 ~ 3,  popularity1 == 90 ~ 4, TRUE ~ 5 ),
         times = case_when(time_signature %in% c(4) ~ 1,TRUE ~ 0),
         energy = round(energy*10),
         popular_artist = case_when(artist_streams > 75000 ~ 1, between(artist_streams, 50000,75000) ~ 2, between(artist_streams, 25000,50000) ~ 3, artist_streams < 25000 ~ 4, TRUE ~ 0 ),
         diff = peak_monthly_listeners - monthly_listeners,
         current_hit = ifelse(diff>20000000,1,0),
         current_trend = ifelse(daily_average<0,0,1)) |>
  dplyr::select(daily, daily1, everything()) |> 
  dplyr::select(-c("...1", title_id, artist, album, title))

pairs(dtal, panel=panel.smooth)
```

<img src="Spotify-Regression_files/figure-html/alternative pairs plot-1.png" style="display: block; margin: auto;" />

<br><br><br><br>

###### Variables Selected

<br>

After considering multiple variables within all the options available in the pairs plots; and after getting all the data that I got. I found the most significant model as compared to others I tried in the process of finding correlations in this data set.

These are the variables I used.

<br>

**Response:**

<br>

1. <span style="color: #0047AB;">**daily1** - Max count of daily streams per artist</span>

<br>

**Explanatory:**

<br>

2. <span style="color: #0047AB;">**explicit** - 1-0 categorical variable. Songs in 0 group (no curse words) tend to have higher streams</span>

3. <span style="color: #0047AB;">**monthly_listeners** - Monthly followers count updated in march 2023 (Main explanatory variable)</span>

4. <span style="color: #0047AB;">**artist_streams** - Monthly artist streams updated in march 2023 </span>

5. <span style="color: #0047AB;">**streams** - Song specific max streams count </span>

6. <span style="color: #0047AB;">**daily** - Song specific daily streams </span>

<br>

When considering all this variables in the model its significance got really high up. It was also maintained even when validations where run. 

Other than my alternative model which I added in this html, I could've also analyzed some plots like `loud~energy` `energy~acoustics` `loud-acoustics`. I did not venture into those ones; While I was trying to predict more on a industry level type of scenario probably business related, I cannot deny that these graphs are also useful scenarios because they may increase a song's quality. But that's pay for another check.


```r
dtwhat <- inner_join(dta, df, by = c("title", "artist"), relationship =
  "many-to-many") |> 
  dplyr::select(-track_genre) |> 
  mutate(explicit = as.factor(explicit))|> 
  dplyr::select(-c("...1", title_id, artist, album, title)) 

g11 <- ggplot(dtwhat, aes((loud),(energy)))+
  geom_point(col="steelblue")+
  theme_minimal()+
  #geom_smooth(se=F)+
  labs(title = "",
       x = "Loudness (decibel level)",
       y = "Energy level (expert rating)")

g22 <- ggplot(dtwhat, aes((energy),(acoustics)))+
  geom_point(col="purple")+
  theme_minimal()+
  #geom_smooth(se=F,col="purple")+
  labs(title = "",
       x = "Energy level (expert rating)",
       y = "Acousticness level (expert rating)")

g33 <- ggplot(dtwhat, aes((acoustics),(loud)))+
  geom_point(col="forestgreen")+
  theme_minimal()+
  #geom_smooth(se=F,col="grey50")+
  labs(title = "",
       x = "Acousticness level (expert rating)",
       y = "Loudness (decibel level)")

grid.arrange(g11,g22,g33, ncol=3, top = "Other Possible Models")
```

<img src="Spotify-Regression_files/figure-html/unnamed-chunk-1-1.png" style="display: block; margin: auto;" />


<br><br><br><br>

###### Final Dataset Used

<br>


```r
datatable(dt)
```

```{=html}
<div class="datatables html-widget html-fill-item" id="htmlwidget-8e40b8c7a0e4734d855d" style="width:100%;height:auto;"></div>
<script type="application/json" data-for="htmlwidget-8e40b8c7a0e4734d855d">{"x":{"filter":"none","vertical":false,"data":[["1","2","3","4","5","6","7","8","9","10","11","12","13","14","15","16","17","18","19","20","21","22","23","24","25","26","27","28","29","30","31","32","33","34","35","36","37","38","39","40","41","42","43","44","45","46","47","48","49","50","51","52","53","54","55","56","57","58","59","60","61","62","63","64","65","66","67","68","69","70","71","72","73","74","75","76","77","78","79","80","81","82","83","84","85","86","87","88","89","90","91","92","93","94","95","96","97","98","99","100","101","102","103","104","105","106","107","108","109","110","111","112","113","114","115","116","117","118","119","120","121","122","123","124","125","126","127","128","129","130","131","132","133","134","135","136","137","138","139","140","141","142","143","144","145","146","147","148","149","150","151","152","153","154","155","156","157","158","159","160","161","162","163","164","165","166","167","168","169","170","171","172","173","174","175","176","177","178","179","180","181","182","183","184","185","186","187","188","189","190","191","192","193","194","195","196","197","198","199","200","201","202","203","204","205","206","207","208","209","210","211","212","213","214","215","216","217","218","219","220","221","222","223","224","225","226","227","228","229","230","231","232","233","234","235","236","237","238","239","240","241","242","243","244","245","246","247","248","249","250","251","252","253","254","255","256","257","258","259","260","261","262","263","264","265","266","267","268","269","270","271","272","273","274","275","276","277","278","279","280","281","282","283","284","285","286","287","288","289","290","291","292","293","294","295","296","297","298","299","300","301","302","303","304","305","306","307","308","309","310","311","312","313","314","315","316","317","318","319","320","321","322","323","324","325","326","327","328","329","330","331","332","333","334","335","336","337","338","339","340","341","342","343","344","345","346","347","348","349","350","351","352","353","354","355","356","357","358","359","360","361","362","363","364","365","366","367","368","369","370","371","372","373","374","375","376","377","378","379","380","381","382","383","384","385","386","387","388","389","390","391","392","393","394","395","396","397","398","399","400","401","402","403","404","405","406","407","408","409","410","411","412","413","414","415","416","417","418","419","420","421","422","423","424","425","426","427","428","429","430","431","432","433","434","435","436","437","438","439","440","441","442","443","444","445","446","447","448","449","450","451","452","453","454","455","456","457","458","459","460","461","462","463","464","465","466","467","468","469","470","471","472","473","474","475","476","477","478","479","480","481","482","483","484","485","486","487","488","489","490","491","492","493","494","495","496","497","498","499","500","501","502","503","504","505","506","507","508","509","510","511","512","513","514","515","516","517","518","519","520","521","522","523","524","525","526","527","528","529","530","531","532","533","534","535","536","537","538","539","540","541","542","543","544","545","546","547","548","549","550","551","552","553","554","555","556","557","558","559","560","561","562","563","564","565","566","567","568","569","570","571","572","573","574","575","576","577","578","579","580","581","582","583","584","585","586","587","588","589","590","591","592","593","594","595","596","597","598","599","600","601","602","603","604","605","606","607","608","609","610","611","612","613","614","615","616","617","618","619","620","621","622","623","624","625","626","627","628","629","630","631","632","633","634","635","636","637","638","639","640","641","642","643","644","645","646","647","648","649","650","651","652","653","654","655","656","657","658","659","660","661","662","663","664","665","666","667","668","669","670","671","672","673","674","675","676","677","678","679","680","681","682","683","684","685","686","687","688","689","690","691","692","693","694","695","696","697","698","699","700","701","702","703","704","705","706","707","708","709","710","711","712","713","714","715","716","717","718","719","720","721","722","723","724","725","726","727","728","729","730","731","732","733","734","735","736","737","738","739","740","741","742","743","744","745","746","747","748","749","750","751","752","753","754","755","756","757","758","759","760","761","762","763","764","765","766","767","768","769","770","771","772","773","774","775","776","777","778","779","780","781","782","783","784","785","786","787","788","789","790","791","792","793","794","795","796","797","798","799","800","801","802","803","804","805","806","807","808","809","810","811","812","813","814","815","816","817","818","819","820","821","822","823","824","825","826","827","828","829","830","831","832","833","834","835","836","837","838","839","840","841","842","843","844","845","846","847","848","849","850","851","852","853","854","855","856","857"],[13.35753509936667,12.13966037001079,13.90152035020198,12.76500248397031,14.02491941876061,14.78033999382297,13.68719638907167,14.05137920078162,13.92802730659703,13.99849272983736,14.15663623622599,13.57350523660886,12.71282096839042,13.34786574452383,12.61206428166479,13.7154968338179,12.58155266054776,14.23864988612521,13.37503310906877,13.51492154820281,13.63967631298581,13.67655189287551,12.54003459114859,13.1003202019537,12.96545222430755,13.48656611626374,13.70860774246214,12.89338785314719,13.40061461903943,12.92793118197177,12.82637490210897,13.04872544860907,12.60449299751163,12.90911817539425,13.15242108429846,13.60803610266464,12.97294879786084,13.35739755419986,13.63307339448813,13.1159629382233,13.20300283006707,13.44118940284909,13.3014764592095,12.0894826479208,13.34443755714452,12.98456690223243,13.62962786785523,12.10513566493213,12.4196142453001,13.50841050423055,13.24872462587261,13.45267381191173,13.96640938481556,12.82393577801476,12.69052759955509,12.73541183080982,12.88272960670851,12.84553947442059,13.31412688502314,13.21277430270929,12.86865752050748,12.58767188253636,13.45723290179674,12.28060945429176,12.68069061232666,12.58413229794774,12.29483873211606,13.3594587478786,13.5037092751623,12.85934269754338,12.54960561072196,13.30039574877985,12.60271225658903,13.05763923904095,12.49138691695229,13.14984623996561,12.70163741252811,12.49077403014232,14.03186874064431,13.85620710461627,12.05151021427826,13.09106464235722,13.27778467822216,13.6963046354844,12.55666965619678,12.96923550766179,13.11474644206516,13.67830040699826,12.68454964912897,13.64342483581786,13.66076136573086,13.64576202836062,13.27550154481385,13.34714239062135,13.04172365822155,13.47096179382338,12.54426126546531,12.86408475532179,13.92154034960696,13.22296498741195,13.18006419966978,12.99704502240556,12.7721276570945,12.77527660539494,12.71080977195561,12.66056598470569,12.67463774058156,13.56680686161221,14.25637112353721,13.81067488492064,13.56764773745441,13.61884339322291,12.23140407842164,13.21316522812972,12.45456420437266,13.21316522812972,13.10185245932945,12.39491042555193,12.75307203753146,12.78598673621373,12.27840726050082,12.15215485753863,12.34774580384632,13.07995888138883,12.32060596212033,12.6242993108075,13.33098113839686,13.17001384474694,12.85565187738618,13.86853934551199,14.19632156940699,12.96292675154783,14.28414199618638,14.15315496446872,13.91612482799687,13.4869314881796,12.55470634514984,14.02933724174505,12.91961296527434,13.14952708067517,12.87842510683341,13.81743370752612,13.69227908147407,12.28769871441012,12.47700395400984,13.21427141283662,12.42176450179923,12.69269061409495,12.65177717849784,12.61059063859979,13.03602958898153,12.37765760411044,13.16762350880919,14.51411681063827,12.02213177836057,14.17479559265064,14.16989708547434,13.49225381080883,13.93862071262628,13.95739546265072,12.66205975065253,12.62054374435324,11.53740342037705,12.31195402551462,12.3042197066697,12.85045491160191,12.85770227722802,12.14475559654137,12.75631031029895,13.49358342656908,12.93101399212103,13.45638117578343,13.26797945518987,12.29343770996502,11.74729378222487,13.49980754016263,12.27679202574283,13.3265507910489,12.83687090602554,13.33049724489595,13.38873805095882,13.17524421098443,12.35355686811293,13.85340731815541,12.52007649057508,12.63908482496764,13.62097632915214,13.16283330995804,13.32946372193726,13.12440329535621,13.37624716870991,13.28612543389946,12.75649487333208,12.81750259217977,12.90837033221826,13.90097056553575,13.77533731129495,13.80554607701099,12.81301085435999,12.66788342988914,12.76649953300684,13.80319908180012,12.98360922703617,13.75406484914815,13.1820765251309,12.97805873256257,13.75535919914124,13.16802866422624,12.18185167170075,13.75296045860516,12.79706804581919,13.65215480784153,12.1401518361389,12.51734794598268,13.93544062044149,13.61191224788412,12.87005874474713,13.17728707390023,13.84038266663938,12.5708546383628,13.59093848680127,13.59093848680127,12.54599962194546,12.2382396938242,12.41940017697109,12.2846481235023,13.79356040638762,13.18818108004028,12.92909411254187,13.43472586725017,13.68120629532408,13.10022410227167,10.89690597832573,13.24985733131858,12.74847251489475,12.95391158913568,13.66104266116236,13.60944533818482,11.98150307793789,12.43020770513754,11.9466151523068,12.79882197601204,12.94086068524919,13.29455210335967,13.10156661612067,12.4848173988591,11.64059722342143,13.15648457162349,12.71774890740097,12.68005584576874,12.78231796458573,12.27995952181543,12.97674783609865,12.97099149979417,12.49751015748337,12.7622683057766,13.16869910333164,13.41554778421171,13.23130387726484,12.82283272022163,12.08598194927205,13.71089865091798,13.09726093084349,13.34430140236355,13.16308296668135,11.98447789420582,13.53417364567233,13.16467162690028,14.4739700158281,12.14072846660417,13.93266406217398,14.02458195938841,12.12196251041079,13.20646001065256,12.74847251489475,12.13442625657766,14.21768157985623,11.75419965835672,13.39421937558235,12.68051951395245,13.0267881624813,13.71874411832813,12.82940608206519,12.5118536889972,13.43023917713331,12.74697732453216,12.19132949745596,12.70734175088869,11.90999543229001,12.50456853730776,12.01587414692566,14.33161681461419,11.74226577664434,12.29393699197124,13.26175307659254,13.22532421613918,12.24872017275119,13.2718483325034,12.73275193321933,14.12017584572645,13.13844733430251,12.83728856974847,12.14227518307411,13.58354697426884,13.24454207131737,12.6768603423146,12.48783598679836,13.28828281860299,13.07182137967012,12.6289420824779,13.29655363124359,12.75597858797922,12.87293488391562,12.75631031029895,13.3941370800866,12.35213210813821,13.44068764559745,12.74017426920834,12.85272028643607,13.25512872669256,12.63742639866416,11.44870689264358,13.38173239135353,11.47200971605517,12.00413392424075,12.48907252359774,11.3019945350842,12.07747762039943,12.52904705207126,12.90214803469962,11.66945046672017,13.2873946483278,12.84810020083174,12.57254394807292,13.40406711517158,13.50565104056779,13.52129384786515,12.48064288233495,12.67318459790537,12.22434959977496,12.22619384952639,12.63050670018594,13.48174684996548,13.11792956255838,12.86265179147205,12.32392678976939,12.43548450985713,12.95507071883013,13.87014645961047,13.18459799659681,11.77918214415306,11.37862806475107,12.94175714859379,13.69310447748486,11.61531055844651,12.49271292550457,12.45599443787699,13.84320646103762,11.91584065814016,12.92756672922997,12.00613890067543,12.1702900153904,11.99326920058193,12.15423709888728,11.45906044547802,11.46621114474514,12.01313695761639,13.24726589816403,11.50008335695693,12.01254250661683,13.41475981973082,14.15599331312277,13.00740808989696,13.39058736319899,12.36173789210766,12.54760704774365,13.56053959556029,13.03902089813448,13.31578476913491,13.42764123209533,11.84493059605867,12.46464897559329,12.5600725146446,13.19047426005345,13.60539304819655,12.72781743760541,13.05662520363787,13.06359590501784,12.55216276736873,10.35593166183829,13.1239381368196,12.53110107948131,12.97263988311305,13.61285689772638,13.31578476913491,12.67191136356177,14.90636281532092,13.36451551535766,13.5908909177667,13.74358266923929,14.1192042574131,13.60618361618248,13.79132544157592,13.33618198626927,13.0319440180158,12.59767544764519,12.99782230758937,13.4179258180318,12.65172274531536,12.86987855018801,13.04538297608079,12.23925110250255,12.73610954530804,12.3020734958576,12.76505680545919,13.00536203614526,12.53331671904374,12.09325103164073,13.61484986245212,12.5929425970559,13.9065105531632,12.47704589991869,12.42280770626322,11.94109747561346,13.34749535049699,12.21445739506457,12.30740556128555,13.36283591084031,12.79773789011928,12.54723742635258,13.52149379700627,13.29223103870752,13.35171322972797,12.98623203641274,13.60689225264159,14.77743600461923,13.94541362617354,14.20435564889562,14.00343638147806,13.20497324863042,12.810353092627,12.4892872537073,13.19360438660157,13.09047634694165,13.54050644613434,13.00376901270953,13.18189186927801,13.44948056310097,12.88945985211793,13.23375805085835,12.86710977217188,13.33285928338117,13.91047613294463,13.79884851386988,13.08259083201965,12.94371267477343,13.6668898927151,13.92328276315965,12.2964936198974,12.64219950727586,12.78708922279615,13.38271947300363,12.97230298759045,14.05865985319999,12.89624289943767,12.94943754225074,13.05627271899134,12.60474137717129,12.29284190987957,13.62733349108352,12.64993440940086,13.27207048488535,12.56885987135661,12.98322773885997,12.64993440940086,12.28923663079091,13.01494519063704,12.50044975611445,12.8274149689605,13.32448265952166,14.12550167285143,12.48493090237216,11.65985297756508,13.71167998246203,12.52734784710029,13.81718515504359,13.40712418711999,12.23137482839239,12.13097187302092,12.29163547387962,12.96979244911228,12.71279988237,13.88744416790489,14.08737646162494,13.8609579985962,13.98183568469446,12.67325041078059,12.07923877435337,12.11640501655204,12.52034674505418,13.67376955854697,13.44201784189938,12.6105505998957,12.15650417856168,11.84188382873385,13.47424485099899,11.47828225902665,12.66489208928708,13.70310416721286,12.47203044550779,12.2408556419526,13.39445098580775,12.13508134995189,11.07516441206124,12.30124645239977,12.9123235954671,12.18766424584682,13.49575973598699,14.103340119539,12.40672106801701,13.14325756559384,12.64949805742473,13.78447799026372,12.99041473157502,13.6496684658141,13.61856701912751,12.15822079101052,13.40436283433709,13.70692373224968,12.91716533785739,13.6616096802672,12.78098512594869,12.75172569222075,13.18423334110762,12.95624031375202,13.18790764261174,12.63464537094967,12.83442527217895,13.19738364757335,12.99755270565838,13.48170915149876,12.7007871798819,14.15317066022224,12.25971767242579,13.15892286031583,13.9096327596747,12.68805733932265,12.35373805566843,13.8155555569518,11.1960891750881,14.31922092824466,13.71951274179224,13.95634409988454,12.79564660116512,13.01411870656664,12.8165960820728,12.84199846909282,12.4875341603689,12.87403021501646,12.5451735669872,9.681093720112464,12.54538014470745,10.72544562667554,12.79173761678687,9.986080850839981,10.59033996253332,12.87045763147245,12.39339003567475,12.34431604419689,12.38409748463935,13.33568282989777,12.83832001016126,12.6153537966092,12.62751608416053,12.75314147715373,12.12235402328678,12.21594376532544,12.12456970556955,12.68775777954975,11.95558359082048,12.4392576126581,12.22498762408079,12.72732472440795,12.14927475492921,12.49893253989599,11.92111352913304,11.98345395645168,13.54549288070647,13.62579815603727,12.61030365912352,12.61882778334969,12.71505657441679,13.56719277810134,13.13824263205499,14.34969424567001,13.33393138993011,13.09125447437709,12.85142566574416,13.08614327790376,12.86240282171281,13.4834055710104,13.26808318855372,13.93966613467031,13.31261838102241,12.69867217861281,12.7101545816628,12.64075345495629,12.7751520829666,13.66361732817823,13.3479152264402,13.11458117446227,12.74172355601176,12.28963223862911,11.9577138450749,11.907902430964,13.55133535473657,12.11240461665765,12.36327295621687,13.66751622223593,11.76310035533547,12.92603700944914,12.38906481074936,11.67027103569401,13.03571993271931,12.44573894139705,12.33271410699741,13.26040209675274,12.30059160330998,12.08137639356561,11.45457575358176,12.466693082722,12.73711551650798,13.7984895184633,13.47006094501609,13.18125473327694,12.58836124118712,12.83192283720959,13.97602097174411,13.13154509611917,14.1687486025248,12.09620755504745,12.44133447303462,12.72014399622108,14.20999156105885,13.13796109802738,12.55044927425661,12.4825104879021,13.26430034400281,12.51530948585352,12.79144807239964,12.27761626110169,13.75337980249777,13.08850872118076,12.60147394968067,12.49166879395431,14.34771227635955,13.48009795972796,14.29783052873232,13.87930567784759,13.95450437932305,13.85182898663837,13.62640242436529,14.32324039472279,12.91894647010749,13.85064601473253,13.45618925837034,14.03069693476886,13.06717830935386,13.93363220295271,13.52599881268156,13.24758534215771,13.8974508487031,13.55108006030646,13.79298781450421,13.73668810547646,13.8973301475861,13.76077767074313,13.55070872503922,11.14833355007164,13.72696891981295,13.46541109097228,13.50578735463561,10.84898751099427,12.31637443297189,11.88116626578012,13.73274723610723,13.14889235163058,13.02999367017001,13.89485579169017,13.63465272939729,14.29046192693394,13.25509019644631,13.41658402669569,14.23214835185556,13.51650056621994,14.28238742899241,13.17268950423609,13.90903186596382,13.81722808217479,12.70844362933949,13.56249783124775,13.76432357371507,13.11129029650314,13.77653165557241,13.38101614740481,14.19600446467981,13.48690231889985,13.491329079575,11.77476651566218,13.67694365124948,13.72180189504689,13.86809541792181,13.72311123749525,14.38966021567673,13.98079191843382,13.20932608654303,13.33369666125514,13.77714699735077,13.39673222079398,13.09018310210283,12.70282469956481,13.63781826955398,13.491329079575,11.77476651566218,13.54723103278733,14.1725435378005,14.0975990152795,12.8522199334279,12.57938639362125,12.42615952999448,12.79672464587951,12.09131814423048,12.54297336176945,13.15676480412166,13.48689676275009,12.42966809472316,13.3269665124987,12.8616996661762,12.79482551017762,13.13219686974898,12.45647332313472,12.61200431144998,13.17908769703902,12.4663115761518,12.82891806266344,13.94552251463323,13.32470649177585,13.27435630238064,13.05330050346726,12.71892920554826,12.91708675760604,12.65702086824075,12.38234678684505,13.59933641381311,13.03744145235248,12.43727960005394,13.74218366530391,12.11350769413584,12.3278477024561,11.88607397160185,12.55933878609474,13.87198540489225,12.76563414627779,13.04453155725017,13.3511867678497,13.46026316601673,12.09886041650887,12.89180490337163,13.47642290411924,13.11770454186356,13.16705754537521,13.48777147609429,13.99781128120788,13.81654901857736,14.05550650656976,13.80285481077037,13.46461743844677,14.05275502245601,13.02864153552321,12.80630103463602,13.07255109104594,12.89042879798576,13.2962343281954,13.92139463653911,13.74785357948907,13.64641238557653,13.23295618073981,12.64041347473663,12.95192612172105,13.52324734141084,13.06133017212904,14.2422014631884,13.4045272525722,12.42975605109685,12.66674537703443,13.35359572601254,12.76336573809595,12.42013912458991,12.64937289194136,12.81295635117018,13.64067729379217,13.13134690441958,13.80942407283583,13.54500151465312,13.92048569937498,13.2581626963387,13.69314516432392,13.21497537493198,12.57895946399618,13.57980974319902,12.67521340260877,13.72869916097731,12.36673209727026,13.84743452216889,14.41426024209768,14.23753906580297,13.95280772614718,12.53253062068327,12.2949805562955,12.19778842594248,11.87790879845456,12.78094010460575,13.49566060257812,13.66569878478599,13.29299858066269,13.83804670070256,14.30012936330866,12.80287686314884,13.55843269581712,12.37402478835683,13.40445635995111,13.71899304963208,13.02398950929605,13.63559051522569,13.25145819295903,13.91133878537694,13.29134468071114,12.55939146397479,13.97526607274629,13.26634942913269,12.50879602817642,12.88399704526835,12.75435011122647,13.39152143478682,12.9948254168018,13.52090457043967,13.27293107458006,13.00859875709147,13.19832391820687,13.861633358356,12.93259657341342,13.301178794869,13.10503205391981,13.93954600648633,13.12608643820445,13.35345446323418,14.58326090947535],[0.4034631054374914,0.4034631054374914,2.299379962045097,0.8394096875381973,2.299379962045097,2.299379962045097,2.299379962045097,0.598836501088704,1.941328239550202,0.3947411447451887,1.338678507178227,2.010225430991154,1.338678507178227,1.338678507178227,2.010225430991154,2.299379962045097,1.067465559671273,1.917804576078714,-0.2119563619236453,1.917804576078714,0.4007875185570534,0.6248683398066509,0.8006553882752305,0.8006553882752305,0.6991292522374928,-0.01106094735942495,1.941328239550202,2.010225430991154,0.7802418874108791,0.01291622526654623,0.412109650826833,0.412109650826833,0.4285303810391604,-0.3812604194113469,0.131028262406404,0.1612681475961223,0.001998002662673058,1.941328239550202,2.010225430991154,-0.02326862693935433,0.308219723669329,2.010225430991154,-0.1369658550731574,0.4285303810391604,0.6580380034463774,0.2859305394129745,1.212238548742969,-0.03355678352884275,-0.5978370007556204,-0.02020270731751947,2.010225430991154,2.010225430991154,1.44243837173793,1.917804576078714,1.941328239550202,-0.8416471888783893,0.8394096875381973,1.212238548742969,1.17588192415941,0.8394096875381973,2.010225430991154,1.44243837173793,-0.0253178079842899,0.856116008838085,0.09075436326846412,1.088225195781623,1.338678507178227,1.055356779997153,-0.2876820724517809,1.551596912744251,1.088225195781623,1.17588192415941,1.212238548742969,0.01192857086527381,1.212238548742969,1.551596912744251,0.856116008838085,-0.8867319296326107,2.53978932795659,2.53978932795659,0.1864795669426184,1.576708087811182,1.461865547966965,2.53978932795659,1.134301131076617,1.221714485802093,0.952043901579973,0.7447904137117837,0.6280751838162305,1.576708087811182,1.756822899347159,2.53978932795659,2.53978932795659,2.53978932795659,2.53978932795659,2.53978932795659,-0.04604393850140685,2.53978932795659,1.756822899347159,1.576708087811182,1.756822899347159,1.756822899347159,1.461865547966965,1.133335724726495,1.756822899347159,1.133335724726495,1.461865547966965,1.975052775942304,1.975052775942304,1.975052775942304,1.975052775942304,1.015230679729058,0.3307415619122278,0.6146449524049877,0.06015392281974714,0.6146449524049877,-0.3624056186477175,-0.02429269256904459,0.6146449524049877,-0.2218943319137778,0.1873090983049937,-0.2395270305647338,-0.9063404010209869,0.6227247162633995,-0.8794767587514388,-0.4431669752921759,0.2776317365982796,0.6339278208999741,-0.02737119679613201,2.66569926056168,2.66569926056168,2.66569926056168,2.66569926056168,2.66569926056168,2.66569926056168,2.66569926056168,2.66569926056168,2.66569926056168,2.66569926056168,2.66569926056168,-0.4780358009429998,1.365070724668264,1.416581053724763,1.416581053724763,1.416581053724763,2.66569926056168,2.286354079883628,2.66569926056168,-0.6014799920341214,0.1629688282781397,-0.6674794338113675,1.365070724668264,-0.1578240851935672,1.506740514195758,0.8046885552928528,3.056215708933502,1.798072831284647,1.798072831284647,1.798072831284647,3.056215708933502,0.5429055172294024,0.5429055172294024,0.259282597930083,0.03440142671733232,0.02761516703297339,-0.2797139028026041,0.04114194333117521,1.080787703755558,0.07973496801885352,1.874107210576808,0.2231435513142098,2.505852414303463,0.9995283304442107,-0.9014021193804044,0.9756910751556026,2.993830466264395,0.259282597930083,2.993830466264395,-0.363843433417345,2.505852414303463,0.9995283304442107,2.993830466264395,1.072268313328508,2.627851925773098,2.336503311197506,1.065055505139267,3.343462331010917,0.7866375236472842,1.644805056271392,3.343462331010917,2.394890763778072,2.700353994467739,2.700353994467739,2.153504869979893,2.153504869979893,2.153504869979893,1.754749643594154,2.073926360991726,2.394890763778072,1.065055505139267,1.612433421413899,2.073926360991726,1.440308949426129,3.431823575203916,2.700353994467739,2.394890763778072,3.431823575203916,1.651155505785969,0.03343477608623742,2.154317076324943,2.154317076324943,2.154317076324943,1.644805056271392,3.343462331010917,1.754749643594154,2.073926360991726,1.416338482468267,3.062596213807498,3.062596213807498,3.062596213807498,3.062596213807498,3.062596213807498,3.062596213807498,2.394890763778072,2.153504869979893,1.644805056271392,3.062596213807498,2.336503311197506,3.062596213807498,2.336503311197506,2.336503311197506,1.416338482468267,1.793591124057025,2.024721329479603,1.325747861251857,3.343462331010917,3.343462331010917,2.700353994467739,1.491779868965868,2.073926360991726,2.073926360991726,0.8006553882752305,0.8024499154503955,2.700353994467739,0.9861898592538223,1.848297320188313,1.554559250924111,1.325747861251857,2.096544449579498,1.793591124057025,1.06608909696255,2.405953625971079,-0.3298939212610904,0.4731237565819792,3.343462331010917,1.308332819650179,1.75526836064564,0.03343477608623742,1.754749643594154,0.9861898592538223,0.4014570867106256,1.656512319820029,1.656512319820029,1.656512319820029,-0.3354727362881295,1.416338482468267,1.896869953630672,2.394890763778072,3.431823575203916,1.793591124057025,1.222009166812035,2.700353994467739,1.644805056271392,1.7509374747078,1.325747861251857,1.325747861251857,3.431823575203916,2.073297706919346,3.343462331010917,2.700353994467739,2.073926360991726,2.153504869979893,1.491779868965868,2.394890763778072,1.091251934261817,2.154317076324943,-0.02737119679613201,2.436678827474744,0.1638180852293949,3.343462331010917,0.4718768736274159,3.431823575203916,1.065055505139267,3.343462331010917,1.74798172188216,1.586783221869294,1.793591124057025,2.336503311197506,1.7509374747078,0.9745596399981308,1.11350090116186,-0.5762534290884459,0.1151128071005047,0.5877866649021191,0.1151128071005047,-0.6733445532637656,0.9745596399981308,0.08250122151174377,1.11350090116186,-0.579818495252942,0.5877866649021191,-0.6033064765601558,-0.6424540662444271,-0.3710636813908321,1.692491467569386,-0.5869869847315546,0.01587334915629016,1.612433421413899,1.754749643594154,1.650963659262599,0.1996701951285677,0.4780957991430718,1.650963659262599,1.091251934261817,0.9685036033210893,-0.4307829160924542,0.4780957991430718,0.1996701951285677,-0.118783535989967,1.498506351726819,0.7499999921527281,0.01783991812833102,0.7499999921527281,0.08892620919440149,3.717466910965849,0.7807000775678068,1.651730824624352,1.651730824624352,3.717466910965849,3.717466910965849,3.717466910965849,3.717466910965849,3.717466910965849,3.717466910965849,0.669878553620591,0.7807000775678068,1.498506351726819,3.717466910965849,3.717466910965849,0.6820862332005203,0.6820862332005203,0.1638180852293949,3.717466910965849,0.521765563804325,0.1638180852293949,0.1638180852293949,0.4718768736274159,2.611906340549308,-0.1996711951290677,3.717466910965849,1.651155505785969,3.717466910965849,3.717466910965849,3.717466910965849,0.7499999921527281,1.651155505785969,0.6820862332005203,2.611906340549308,0.521765563804325,3.717466910965849,2.703305630071453,2.703305630071453,2.703305630071453,2.703305630071453,2.703305630071453,2.703305630071453,2.703305630071453,2.703305630071453,2.703305630071453,2.703305630071453,2.703305630071453,2.703305630071453,2.703305630071453,1.350667183476739,1.420212579323351,1.350667183476739,1.350667183476739,1.379269746182926,1.350667183476739,-0.5430045221302259,1.379269746182926,2.026831591407539,1.420212579323351,2.026831591407539,2.703305630071453,1.350667183476739,1.3015531326648,-0.1707883209802816,1.400936637856761,0.3604677421774286,0.8775499035577244,1.400936637856761,0.8510052651755257,1.239822457219633,0.8510052651755257,0.3653373170173851,-0.1438703704197019,1.239822457219633,-0.709276562489829,-0.6597124044737079,0.3694924476493469,0.8808707866428946,-0.1221676339742075,-0.6198967188203526,1.239822457219633,0.6611403844248589,0.6611403844248589,0.8775499035577244,0.3052763808527321,0.3555743384946994,0.880041599199034,-0.4034671054454912,0.3555743384946994,0.3555743384946994,0.4094571293777018,1.663736685841479,0.9238619973704731,0.6392188385343897,0.6392188385343897,1.433416468188625,0.8355144218468673,-0.02429269256904459,0.8355144218468673,1.102272249499597,1.102272249499597,2.460101530869029,2.460101530869029,2.460101530869029,2.460101530869029,2.460101530869029,2.460101530869029,2.460101530869029,2.460101530869029,0.555033878430311,0.6575200029167941,-0.2561834053924099,0.06485097231961627,1.055356779997153,-0.3368723166425527,0.5777363290486176,1.562136639316087,1.085864715845607,1.562136639316087,1.995516306742075,1.562136639316087,1.562136639316087,1.995516306742075,1.995516306742075,0.5104255437446509,-0.6462635946610948,1.562136639316087,1.562136639316087,1.995516306742075,1.995516306742075,-0.2890162954649176,0.3927175352856618,0.3386128011203239,-0.2890162954649176,1.085864715845607,0.5376622777195503,0.2342812957246657,1.109552228706444,-0.27049724769768,0.264669298142708,0.2342812957246657,1.109552228706444,1.109552228706444,0.9477893989335261,-0.2536027587989182,0.9477893989335261,3.242397019909546,2.570702009950987,2.570702009950987,3.159889859720275,-0.780886094867952,3.242397019909546,3.698359377313068,0.9477893989335261,0.9477893989335261,0.128393214768399,3.242397019909546,1.402167710276181,2.274597056453877,3.453979567101125,3.14806701985727,3.453979567101125,2.309362077273069,1.254475786499043,1.864545138953703,3.242397019909546,2.153388786631919,3.453979567101125,1.245018773761937,0.128393214768399,0.128393214768399,2.111787717719047,-1.010601411345396,3.453979567101125,3.453979567101125,3.453979567101125,3.453979567101125,3.453979567101125,1.121677561599106,-0.5395680926316447,3.453979567101125,-0.148500008318444,0.3770656335864664,0.9604989865384754,1.246169853025724,1.171242979703017,1.171242979703017,-0.5464528014091419,1.640161085076941,1.725263477939691,0.3350426438116185,0.5538851132264376,0.1705863005755337,1.171242979703017,1.640161085076941,-0.6311117896404926,0.7742662318447373,-0.02224560894731974,0.178146185383474,0.1275133202989595,-0.5464528014091419,-0.1613431504087629,0.9604989865384754,0.1705863005755337,0.171429115627531,-0.3739664410487935,2.036664512323778,1.725263477939691,1.982242078302704,-0.2718087232954908,0.1889660995126232,1.982242078302704,-0.2131932204610416,0.7742662318447373,1.982242078302704,1.612433421413899,0.6507614640635074,0.1848184369925419,0.7756484020716891,0.1133286853070033,0.688134638736401,0.02176149178151271,0.2223432311434407,0.8977193462887197,0.5738004229273791,0.7371640659767196,0.7371640659767196,0.5894519442211802,0.7371640659767196,0.2859305394129745,0.03246719013750141,-0.09211528890780563,1.879770346550768,1.353771169414331,1.879770346550768,2.461979145144296,2.461979145144296,1.610836933347808,1.610836933347808,2.461979145144296,1.610836933347808,2.461979145144296,1.610836933347808,2.461979145144296,2.461979145144296,2.461979145144296,1.610836933347808,2.461979145144296,1.610836933347808,1.610836933347808,1.610836933347808,2.461979145144296,1.402167710276181,3.226645562132561,3.226645562132561,2.676146652089202,2.676146652089202,1.781035505865079,2.570702009950987,1.974775229451485,2.089763090751708,3.638112337060283,0.9597332897169919,0.8082599876604499,1.213725095768614,1.037091432019859,1.213725095768614,3.638112337060283,1.049772123248636,3.638112337060283,2.431066254642586,3.638112337060283,-0.2107210313156525,3.638112337060283,2.917608556772412,2.385546613708393,2.175773926127659,2.385546613708393,3.638112337060283,2.385546613708393,3.638112337060283,1.249041767693514,2.385546613708393,1.696165915943925,2.917608556772412,2.525728644308256,2.525728644308256,3.638112337060283,1.696165915943925,3.638112337060283,3.638112337060283,0.5080216964332565,3.638112337060283,3.638112337060283,3.638112337060283,2.385546613708393,3.638112337060283,1.520606698727485,1.375991468123994,1.375991468123994,1.375991468123994,1.375991468123994,1.069183477977298,2.210579447198021,0.3520643313810491,2.210579447198021,2.210579447198021,2.210579447198021,2.210579447198021,0.8219800524029137,0.2783890255401882,2.210579447198021,2.210579447198021,1.956991381923272,0.8219800524029137,-0.709276562489829,0.2783890255401882,1.956991381923272,1.956991381923272,1.956991381923272,1.239822457219633,3.568461700634797,0.910272659548592,2.263740309170478,0.5359084041334539,2.614105754925668,2.614105754925668,0.9146894505071812,2.080316159090497,-0.3051673867928005,3.568461700634797,4.224978009857304,4.224978009857304,4.224978009857304,4.224978009857304,1.367111541703117,3.568461700634797,2.989311705751068,4.224978009857304,3.568461700634797,2.080316159090497,3.568461700634797,3.568461700634797,3.091860300648211,3.091860300648211,2.263740309170478,2.614105754925668,2.437028569543537,2.989311705751068,2.989311705751068,2.989311705751068,2.989311705751068,2.614105754925668,2.263740309170478,2.989311705751068,2.642835896118608,1.526708264835104,-0.3739664410487935,-0.2943710606025776,3.568461700634797,3.568461700634797,4.224978009857304,0.5810976767513224,2.056940276333322,2.642835896118608,4.224978009857304,2.437028569543537,1.360207026356572,3.21221368199465,3.21221368199465,3.717466910965849,3.21221368199465,3.21221368199465,3.717466910965849,3.717466910965849,2.642835896118608,2.642835896118608,2.37815627984112,2.614105754925668,3.091860300648211,3.091860300648211,1.159393760927969,2.263740309170478,4.224978009857304,3.21221368199465,0.3300229129413059,4.224978009857304,2.263740309170478,3.717466910965849,3.717466910965849,2.642835896118608,1.724194149732289,0.427226599889677,0.1371498381472337,1.333684410847593,1.333684410847593,1.333684410847593,1.333684410847593,0.6103089022509408,1.433177947018741,1.433177947018741,-0.1684186516249633,1.433177947018741,1.004667842491568,0.01783991812833102,1.004667842491568,0.1388919988666187,1.17588192415941,1.916775542544323,1.916775542544323,1.916775542544323,1.916775542544323,1.916775542544323,1.916775542544323,0.6455313266182821,0.6455313266182821,1.916775542544323,0.4977403842173352,1.22436349397367,1.005765738301758,-0.1165338162559515,-0.7507762933965817,3.192531849528599,3.192531849528599,3.192531849528599,1.459544822859483,1.459544822859483,0.1831545430978465,0.1930966299619132,1.971299383060133,1.971299383060133,1.971299383060133,-0.3038114543816646,1.971299383060133,1.344951397585508,1.344951397585508,1.344951397585508,-0.1935847490726654,0.2507587183471831,2.701965057446262,2.701965057446262,2.701965057446262,2.701965057446262,2.701965057446262,2.701965057446262,1.111199404173581,2.701965057446262,-0.25489224962879,2.008884948216702,2.008884948216702,2.008884948216702,2.701965057446262,2.701965057446262,1.111199404173581,-0.07904320734045285,0.001998002662673058,1.111199404173581,1.131079478805611,2.379731302170699,2.701965057446262,1.299919145373145,0.824613943302251,-0.7465479572870606,2.008884948216702,2.379731302170699,-0.6812186096946715,0.10345870836823,-0.4541302800894454,1.097611788334526,1.129787906136559,1.129787906136559,0.04210117601863533,2.284115576710384,2.379731302170699,2.701965057446262,1.093934699116999,0.3155404005801773,1.299919145373145,2.284115576710384,1.299919145373145,0.9305880365749795,0.7738050835773999,1.074661067845596,1.070212814146412,0.1061601958283907,1.199663532737838,1.199663532737838,2.504627574400201,2.504627574400201,1.310762298507454,1.310762298507454,0.9925107577855641,2.604391939491223,2.604391939491223,2.604391939491223,-0.2131932204610416,0.2429461786103894,2.604391939491223,2.604391939491223,2.604391939491223,2.66569926056168,2.604391939491223,2.604391939491223,1.466952264137345,1.466952264137345,-0.4700036292457356,2.052198805285962,1.466952264137345,-0.3147107448397002,0.0944006754214843,0.706556867469863,2.175773926127659,0.2844267797311082,0.538829820175588,1.727931806481889,1.727931806481889,1.727931806481889,0.1484200051182732,0.128393214768399,0.830297018707179,-0.2731219211204512,-0.3783364407199117,0.9313763692921958],[16.50820391063406,16.50820391063406,17.58799331944035,16.65679594115833,17.58799331944035,17.58799331944035,17.58799331944035,16.64517136392878,17.29342827278237,16.65297485390803,17.16455820842611,17.38799360211192,17.16455820842611,17.16455820842611,17.38799360211192,17.58799331944035,16.4270988295864,17.16698730270146,16.06121858577142,17.16698730270146,16.6681485455484,16.69147294212456,16.55668239780023,16.55668239780023,16.48432223242121,16.14489540781375,17.29342827278237,17.38799360211192,16.14768746236379,16.19265844823484,16.34970748493958,16.34970748493958,16.39347037271681,15.87948014891669,16.28915629095641,16.29786637380654,15.85118292791877,17.29342827278237,17.38799360211192,16.23922074017155,16.10223432252978,17.38799360211192,16.17148465281611,16.39347037271681,16.42373902415162,16.10758217757015,16.82734701033055,15.5404893498557,15.60139346061559,16.05411974485656,17.38799360211192,17.38799360211192,16.94521893774304,17.16698730270146,17.29342827278237,15.63322850490393,16.65679594115833,16.82734701033055,16.783752460098,16.65679594115833,17.38799360211192,16.94521893774304,16.05083267519573,16.41123612419876,15.85394805730416,16.60756502478123,17.16455820842611,16.3932377758922,16.06090034232012,16.34317125465674,16.60756502478123,16.783752460098,16.82734701033055,15.89251225063171,16.82734701033055,16.34317125465674,16.41123612419876,15.48435220606005,17.51913063332623,17.51913063332623,16.24861913686369,17.05446748115592,16.43353152382913,17.51913063332623,16.20027536591207,16.99322920714529,16.23255592791678,16.5961833285557,16.11962138643177,17.05446748115592,16.95687191978669,17.51913063332623,17.51913063332623,17.51913063332623,17.51913063332623,17.51913063332623,15.82457255288153,17.51913063332623,16.95687191978669,17.05446748115592,16.95687191978669,16.95687191978669,16.43353152382913,16.8712620465836,16.95687191978669,16.8712620465836,16.43353152382913,17.08622222801272,17.08622222801272,17.08622222801272,17.08622222801272,16.72477941346262,16.31999660752066,16.6363924756669,16.07930378183829,16.6363924756669,15.90728115848519,15.93822696341942,16.6363924756669,15.90163310692973,15.94354903085954,15.81917236549006,15.53418127904306,16.3323109840842,15.39047047674987,15.70749794356875,16.26255558395185,16.64927463269345,16.03759152851603,17.9112257637585,17.9112257637585,17.9112257637585,17.9112257637585,17.9112257637585,17.9112257637585,17.9112257637585,17.9112257637585,17.9112257637585,17.9112257637585,17.9112257637585,15.70765518626488,16.8013756607072,17.16805890049839,17.16805890049839,17.16805890049839,17.9112257637585,17.43717134254946,17.9112257637585,15.74770000725018,16.06250947109877,15.62989970037754,16.8013756607072,16.08734433895957,17.46298375837013,16.2775083761948,17.30764797819802,17.46817442301304,17.46817442301304,17.46817442301304,17.30764797819802,16.49909899482858,16.49909899482858,16.16830642463169,15.84292616852274,16.06162497704164,15.82091608643974,16.08679495550128,16.3372115019665,16.11443997711764,17.09584062747553,16.17539729898421,17.33046941816969,16.97530167723621,15.42560425532443,16.63963667917066,17.59175706955604,16.16830642463169,17.59175706955604,15.91545080814896,17.33046941816969,16.97530167723621,17.59175706955604,16.2037370566323,17.72767717308006,17.89965430163045,16.93675361991894,18.15819429902114,16.63816565348163,17.36372549625615,18.15819429902114,17.75318125146029,18.09401889571956,18.09401889571956,17.82869746758156,17.82869746758156,17.82869746758156,17.4915067278191,17.67455181278565,17.75318125146029,16.93675361991894,17.57417065985459,17.67455181278565,17.34773725466386,18.31273084540104,18.09401889571956,17.75318125146029,18.31273084540104,17.53246015827729,16.22750487950578,17.78961382172637,17.78961382172637,17.78961382172637,17.36372549625615,18.15819429902114,17.4915067278191,17.67455181278565,17.28350593747289,18.1326628817962,18.1326628817962,18.1326628817962,18.1326628817962,18.1326628817962,18.1326628817962,17.75318125146029,17.82869746758156,17.36372549625615,18.1326628817962,17.89965430163045,18.1326628817962,17.89965430163045,17.89965430163045,17.28350593747289,17.45413354841692,17.21401508380921,17.16999260650562,18.15819429902114,18.15819429902114,18.09401889571956,17.26754901511176,17.67455181278565,17.67455181278565,16.67757645835571,16.42207089092639,18.09401889571956,16.75380596714986,17.72979209099776,17.5348873300867,17.16999260650562,17.64990962979162,17.45413354841692,17.04338128711397,17.64977766564146,15.9767145584668,16.52324217499063,18.15819429902114,17.20527614494743,17.57144887791464,16.22750487950578,17.4915067278191,16.75380596714986,16.63713185546574,17.57649662749578,17.57649662749578,17.57649662749578,16.0213590643193,17.28350593747289,17.51911740468018,17.75318125146029,18.31273084540104,17.45413354841692,17.09943336218326,18.09401889571956,17.36372549625615,17.34209579262536,17.16999260650562,17.16999260650562,18.31273084540104,17.92535876518373,18.15819429902114,18.09401889571956,17.67455181278565,17.82869746758156,17.26754901511176,17.75318125146029,17.0323923560819,17.78961382172637,16.11526404574557,17.85933882210461,16.26243882258991,18.15819429902114,16.67077365661313,18.31273084540104,16.93675361991894,18.15819429902114,17.47009720283679,17.30188599265977,17.45413354841692,17.89965430163045,17.34209579262536,17.02906219446224,17.1020246946509,15.79975256371853,16.44179342619259,16.84393008456503,16.44179342619259,15.64246791603756,17.02906219446224,16.38964582623315,17.1020246946509,15.65402717380036,16.84393008456503,15.77187826423293,15.74983022471022,15.47159729591874,17.41197898317171,15.65936994617078,16.33536325264384,17.57417065985459,17.4915067278191,16.77462345196099,16.38266954930886,16.19277705866644,16.77462345196099,17.0323923560819,16.57908861834866,15.81638240031091,16.19277705866644,16.38266954930886,15.91976421728859,16.49880190594892,16.55283861569764,16.14945923137452,16.55283861569764,16.02688674940298,18.49659028801472,16.54368960765651,17.23574632071249,17.23574632071249,18.49659028801472,18.49659028801472,18.49659028801472,18.49659028801472,18.49659028801472,18.49659028801472,16.57306768217586,16.54368960765651,16.49880190594892,18.49659028801472,18.49659028801472,16.51603907532764,16.51603907532764,16.26243882258991,18.49659028801472,16.71930581619246,16.26243882258991,16.26243882258991,16.67077365661313,18.09743395822774,15.39265624375387,18.49659028801472,17.53246015827729,18.49659028801472,18.49659028801472,18.49659028801472,16.55283861569764,17.53246015827729,16.51603907532764,18.09743395822774,16.71930581619246,18.49659028801472,17.5256348586799,17.5256348586799,17.5256348586799,17.5256348586799,17.5256348586799,17.5256348586799,17.5256348586799,17.5256348586799,17.5256348586799,17.5256348586799,17.5256348586799,17.5256348586799,17.5256348586799,16.99066284958375,16.7607528459707,16.99066284958375,16.99066284958375,16.89742428922013,16.99066284958375,15.6651770774008,16.89742428922013,16.63674336227545,16.7607528459707,16.63674336227545,17.5256348586799,16.99066284958375,17.2137576584905,16.00754904407977,17.15058245325872,16.47596392259824,17.00006488092107,17.15058245325872,16.81751084640558,17.10564631459523,16.81751084640558,16.29511333371338,15.99150911546162,17.10564631459523,15.70019372524939,15.65173880411117,16.5100679390011,16.71515898751296,15.85496920610514,15.55004337573013,17.10564631459523,16.50819497441087,16.50819497441087,17.00006488092107,16.3355392282417,16.12943234709556,16.87709564484535,15.57380389794792,16.12943234709556,16.12943234709556,16.41047925387463,17.25628564033412,16.73435061001401,16.50901799042852,16.50901799042852,16.93772362070095,16.84591702296428,16.21807346583997,16.84591702296428,16.59614302983429,16.59614302983429,17.77392932039844,17.77392932039844,17.77392932039844,17.77392932039844,17.77392932039844,17.77392932039844,17.77392932039844,17.77392932039844,16.42709427766973,16.57562549248508,16.04458656097733,15.90494884031141,16.3932377758922,15.71238201196891,15.8847099524667,17.26734015221422,16.97744728611724,17.26734015221422,17.49157664539654,17.26734015221422,17.26734015221422,17.49157664539654,17.49157664539654,16.47961319058057,15.74919340010857,17.26734015221422,17.26734015221422,17.49157664539654,17.49157664539654,16.12611272814166,16.11524057911677,15.99381595764773,16.12611272814166,16.97744728611724,16.63461740095079,16.1298334921734,16.85905466013064,15.51842624510072,15.87159301313747,16.1298334921734,16.85905466013064,16.85905466013064,16.9959013222205,15.81379329019397,16.9959013222205,18.16811703278531,17.8174268985483,17.8174268985483,17.87026087861337,15.57002669107318,18.16811703278531,18.15493615562233,16.9959013222205,16.9959013222205,16.30581743358333,18.16811703278531,17.20147023320649,17.45330530119876,18.05847957279464,18.07099367073268,18.05847957279464,17.64422873411443,16.93049634105336,17.12064334198719,18.16811703278531,17.42853789286083,18.05847957279464,16.45652931985581,16.30581743358333,16.30581743358333,17.43909766021497,15.41392068730195,18.05847957279464,18.05847957279464,18.05847957279464,18.05847957279464,18.05847957279464,16.78125418340622,15.73578035618145,18.05847957279464,16.03975273947901,16.31840847330061,16.82581365953373,16.9878233986554,17.02128724247799,17.02128724247799,15.55747595928012,17.01786009287673,16.79534157397254,16.25361477922674,16.37736536300291,16.0301172975067,17.02128724247799,17.01786009287673,15.62877267921549,16.53638964916043,16.14116001183678,15.82969305232321,15.8645140259567,15.81430953500902,15.80637836644775,16.82581365953373,16.0301172975067,16.25638067714253,15.61083383594204,17.14850725427717,16.79534157397254,16.92767272987494,15.63179969967941,15.8699002819223,16.92767272987494,15.98866782716653,16.53638964916043,16.92767272987494,17.57417065985459,16.72429127092142,16.30312246191697,16.90987811340199,15.65924495588109,16.14348041420106,15.91372306751168,16.06179034370768,15.95242765069074,15.87133566401752,16.48428666315504,16.48428666315504,16.09484340054813,16.48428666315504,16.2478631188973,15.84270545622691,15.62499517121636,16.90799106773273,17.239029320303,16.90799106773273,17.09513184440631,17.09513184440631,16.60738240983959,16.60738240983959,17.09513184440631,16.60738240983959,17.09513184440631,16.60738240983959,17.09513184440631,17.09513184440631,17.09513184440631,16.60738240983959,17.09513184440631,16.60738240983959,16.60738240983959,16.60738240983959,17.09513184440631,17.20147023320649,17.59831671258896,17.59831671258896,17.88159807379292,17.88159807379292,16.52119857129647,17.8174268985483,17.39352992926325,17.48118156756532,17.99432588644486,16.51555399027237,16.31031687725617,17.12901600383646,16.70634537529853,17.12901600383646,17.99432588644486,16.66910756002427,17.99432588644486,17.99111119911949,17.99432588644486,15.78282485232137,17.99432588644486,17.74549612625891,17.62855002706755,16.93041432504297,17.62855002706755,17.99432588644486,17.62855002706755,17.99432588644486,16.82242359319967,17.62855002706755,17.17564987557729,17.74549612625891,17.57826634669174,17.57826634669174,17.99432588644486,17.17564987557729,17.99432588644486,17.99432588644486,16.18922909644783,17.99432588644486,17.99432588644486,17.99432588644486,17.62855002706755,17.99432588644486,16.49389374630373,17.20989172861805,17.20989172861805,17.20989172861805,17.20989172861805,16.22538326856183,17.89238895423733,16.30236173260446,17.89238895423733,17.89238895423733,17.89238895423733,17.89238895423733,16.79640971273471,16.29049110660556,17.89238895423733,17.89238895423733,17.74902184760174,16.79640971273471,15.51565717052384,16.29049110660556,17.74902184760174,17.74902184760174,17.74902184760174,17.10564631459523,18.47195247410741,16.9841658181278,17.60528203889385,16.68584920867248,18.02268622636427,18.02268622636427,16.92893012925859,17.75930236837986,15.98500199108626,18.47195247410741,18.37763034129063,18.37763034129063,18.37763034129063,18.37763034129063,16.74549189570296,18.47195247410741,18.14710296335901,18.37763034129063,18.47195247410741,17.75930236837986,18.47195247410741,18.47195247410741,18.29685761611028,18.29685761611028,17.60528203889385,18.02268622636427,17.96075813008951,18.14710296335901,18.14710296335901,18.14710296335901,18.14710296335901,18.02268622636427,17.60528203889385,18.14710296335901,17.7113216983861,17.3102809447495,15.88862890028913,16.05008208550893,18.47195247410741,18.47195247410741,18.37763034129063,16.38146794565412,17.53158320293223,17.7113216983861,18.37763034129063,17.96075813008951,16.93427721026068,17.95398230255052,17.95398230255052,18.49659028801472,17.95398230255052,17.95398230255052,18.49659028801472,18.49659028801472,17.7113216983861,17.7113216983861,17.80646488447384,18.02268622636427,18.29685761611028,18.29685761611028,16.93544711785734,17.60528203889385,18.37763034129063,17.95398230255052,16.31189590800747,18.37763034129063,17.60528203889385,18.49659028801472,18.49659028801472,17.7113216983861,17.41159804142207,16.74492293801199,16.28910943182061,16.73221493791337,16.73221493791337,16.73221493791337,16.73221493791337,16.36827365597707,16.87261065944155,16.87261065944155,15.87477809089437,16.87261065944155,16.69418292184326,15.82739582967944,16.69418292184326,16.05550233002486,16.783752460098,17.34101592291797,17.34101592291797,17.34101592291797,17.34101592291797,17.34101592291797,17.34101592291797,16.28674212862896,16.28674212862896,17.34101592291797,16.14018135632174,16.43291434697108,16.24605191891613,16.11295676949597,15.77097308033889,17.95748600948167,17.95748600948167,17.95748600948167,16.99610519938291,16.99610519938291,16.50548246026107,16.1140411425365,17.11792945553555,17.11792945553555,17.11792945553555,15.6053815785284,17.11792945553555,17.0321370149002,17.0321370149002,17.0321370149002,16.28822020919684,16.33989918485008,17.91158104658484,17.91158104658484,17.91158104658484,17.91158104658484,17.91158104658484,17.91158104658484,16.89690342160591,17.91158104658484,15.97625393735753,17.24728618969196,17.24728618969196,17.24728618969196,17.91158104658484,17.91158104658484,16.89690342160591,15.96963868115686,16.32990432457659,16.89690342160591,17.19069926056704,16.94044183412938,17.91158104658484,17.06504368829448,16.93994365068662,15.66901778276374,17.24728618969196,16.94044183412938,15.67178026667998,16.43304302379941,15.92062423234798,17.16515471672355,17.03701025746069,17.03701025746069,16.18857965016155,17.79009501892314,16.94044183412938,17.91158104658484,16.74849263038285,16.19583111437431,17.06504368829448,17.79009501892314,17.06504368829448,16.97337630673808,17.1364475415934,17.2451562549554,16.27104984385191,16.03131184807174,16.63673086037114,16.63673086037114,17.81460777507458,17.81460777507458,17.0379131165897,17.0379131165897,16.93158911258581,17.42357646154831,17.42357646154831,17.42357646154831,15.77806302366168,16.43565167016315,17.42357646154831,17.42357646154831,17.42357646154831,17.9112257637585,17.42357646154831,17.42357646154831,17.29980885850905,17.29980885850905,15.96533926531767,17.61042416579097,17.29980885850905,15.99664595129798,15.99924150436737,16.43312498004467,16.93041432504297,16.22542871985319,16.26634245142236,17.3381945677688,17.3381945677688,17.3381945677688,16.31347491692197,16.19778223368843,16.88750047718164,16.009171571119,16.17306525443296,16.85085096961281],[8.657459354476865,8.657459354476865,9.465664528319838,8.527895714549373,9.465664528319838,9.465664528319838,9.465664528319838,7.84129636426282,9.346513272744158,8.009230133263724,8.994954154594799,9.676875492895563,8.994954154594799,8.994954154594799,9.676875492895563,9.465664528319838,8.99715952767618,9.146004496971164,7.34096524739886,9.146004496971164,8.154959971700073,8.298614206599607,8.368669980972477,8.368669980972477,7.949091499830517,7.50069530912505,9.346513272744158,9.676875492895563,8.382632698020542,7.678835250598505,8.035278911144667,8.035278911144667,8.506819570988677,7.388327859577107,7.59749666640725,7.585077909195111,7.588273043141941,9.346513272744158,9.676875492895563,7.479751257977932,7.801268563166303,9.676875492895563,7.427382058701196,8.506819570988677,8.30112455377399,8.140956733968203,9.037604723205819,7.746041225208321,7.307135243434353,7.108325969443955,9.676875492895563,9.676875492895563,8.821481612031421,9.146004496971164,9.346513272744158,6.90032776315334,8.527895714549373,9.037604723205819,8.696192569212887,8.527895714549373,9.676875492895563,8.821481612031421,7.502958815211956,8.630343289348893,7.875119281040293,8.659022560408832,8.994954154594799,8.618141989744455,6.85393247228463,8.99299249325855,8.659022560408832,8.696192569212887,9.037604723205819,7.72616853837448,9.037604723205819,8.99299249325855,8.630343289348893,7.326531401149974,9.910239705358661,9.910239705358661,8.266883972029259,9.196129879507067,8.973807583542957,9.910239705358661,8.633588598094025,8.82217476094608,8.189855111084171,8.179171755405095,8.33076722303719,9.196129879507067,9.064262023039797,9.910239705358661,9.910239705358661,9.910239705358661,9.910239705358661,9.910239705358661,7.592466980720981,9.910239705358661,9.064262023039797,9.196129879507067,9.064262023039797,9.064262023039797,8.973807583542957,8.653907014732059,9.064262023039797,8.653907014732059,8.973807583542957,8.637408821469814,8.637408821469814,8.637408821469814,8.637408821469814,8.470751267970165,8.241070715311141,8.222634193691498,7.498149651967252,8.222634193691498,7.217076526537399,7.425298290495652,8.222634193691498,7.478508811328016,8.13882755318979,7.839919360012582,6.85751406254539,8.328740879891958,6.977374620862194,7.349937970025464,7.921970951011497,8.062842414578226,7.534335215269969,9.98764026409912,9.98764026409912,9.98764026409912,9.98764026409912,9.98764026409912,9.98764026409912,9.98764026409912,9.98764026409912,9.98764026409912,9.98764026409912,9.98764026409912,7.562733193424579,8.757877991507106,9.183544549189019,9.183544549189019,9.183544549189019,9.98764026409912,9.904457082390131,9.98764026409912,7.752335163302292,7.854497618312121,7.113304961895299,8.757877991507106,7.132497551660044,8.492162521690991,8.655022833318114,9.255046015360033,9.014605254264806,9.014605254264806,9.014605254264806,9.255046015360033,8.132442059905177,8.132442059905177,8.492408574593606,8.3301886867179,7.733114231850586,7.587867877271622,7.986538940016061,8.678716625572909,7.448333860897476,8.855962787086133,8.236791221261745,9.435058682705662,8.185266607642872,6.934299834204249,8.721145768336486,9.780251551450796,8.492408574593606,9.780251551450796,7.410105084267084,9.435058682705662,8.185266607642872,9.780251551450796,8.872136637176821,10.41004005952894,9.921749778336602,9.030854790001436,10.80247951983195,9.219082051912938,9.572598699777137,10.80247951983195,10.0482976013198,10.39556145660906,10.39556145660906,10.12367874993935,10.12367874993935,10.12367874993935,9.757183507195171,10.10556271581692,10.0482976013198,9.030854790001436,9.467993556713555,10.10556271581692,9.272450948039086,10.43312707967774,10.39556145660906,10.0482976013198,10.43312707967774,9.960472618018786,7.639594186653069,10.08470850388605,10.08470850388605,10.08470850388605,9.572598699777137,10.80247951983195,9.757183507195171,10.10556271581692,9.191830109217086,10.02476226297274,10.02476226297274,10.02476226297274,10.02476226297274,10.02476226297274,10.02476226297274,10.0482976013198,10.12367874993935,9.572598699777137,10.02476226297274,9.921749778336602,10.02476226297274,9.921749778336602,9.921749778336602,9.191830109217086,9.989642312734334,9.342043844973093,9.569084174350692,10.80247951983195,10.80247951983195,10.39556145660906,9.614244262017392,10.10556271581692,10.10556271581692,8.866200816445671,8.842546700589027,10.39556145660906,9.26618180787014,9.880085856354508,9.597994468204893,9.569084174350692,9.584569338562131,9.989642312734334,9.233099409096697,10.48332324764115,7.929738471414313,9.061202108972461,10.80247951983195,9.284789000385688,9.787122261404862,7.639594186653069,9.757183507195171,9.26618180787014,8.472635059552028,9.293403130087711,9.293403130087711,9.293403130087711,7.202363311504651,9.191830109217086,9.334255677881742,10.0482976013198,10.43312707967774,9.989642312734334,8.681995511991254,10.39556145660906,9.572598699777137,9.446479118785204,9.569084174350692,9.569084174350692,10.43312707967774,9.874058741845316,10.80247951983195,10.39556145660906,10.10556271581692,10.12367874993935,9.614244262017392,10.0482976013198,9.228494582556225,10.08470850388605,8.608933369978953,10.20657288335387,8.722319400863455,10.80247951983195,8.980411351346952,10.43312707967774,9.030854790001436,10.80247951983195,9.225140310587783,9.300610599805156,9.989642312734334,9.921749778336602,9.446479118785204,8.322612801226343,8.586066055702613,7.122382670528007,8.032912122872359,7.906768202427492,8.032912122872359,7.149602815090119,8.322612801226343,7.457955450222149,8.586066055702613,7.058070039641005,7.906768202427492,6.950335726517255,6.889387621623952,7.22190886863971,8.748701659062075,8.157456422013103,7.941259995745106,9.467993556713555,9.757183507195171,8.748907905146114,8.647062888207767,8.877549351357443,8.748907905146114,9.228494582556225,8.518052821828119,7.566931939970929,8.877549351357443,8.647062888207767,8.389291635763685,9.114996191593844,9.281935188798878,7.55924678427325,9.281935188798878,7.927721106992244,10.55596897637006,8.6143380508026,9.480550698515923,9.480550698515923,10.55596897637006,10.55596897637006,10.55596897637006,10.55596897637006,10.55596897637006,10.55596897637006,8.407869276301488,8.6143380508026,9.114996191593844,10.55596897637006,10.55596897637006,8.907544661714354,8.907544661714354,8.722319400863455,10.55596897637006,8.350807973920636,8.722319400863455,8.722319400863455,8.980411351346952,10.16810312705616,7.70634297200008,10.55596897637006,9.960472618018786,10.55596897637006,10.55596897637006,10.55596897637006,9.281935188798878,9.960472618018786,8.907544661714354,10.16810312705616,8.350807973920636,10.55596897637006,10.40044951836295,10.40044951836295,10.40044951836295,10.40044951836295,10.40044951836295,10.40044951836295,10.40044951836295,10.40044951836295,10.40044951836295,10.40044951836295,10.40044951836295,10.40044951836295,10.40044951836295,9.355349393886158,8.855805992536562,9.355349393886158,9.355349393886158,8.975301349248907,9.355349393886158,8.526271855349442,8.975301349248907,9.475914862764256,8.855805992536562,9.475914862764256,10.40044951836295,9.355349393886158,8.288509320837514,7.422493239568043,8.915136750125551,8.418168119938422,8.740720669021316,8.915136750125551,8.568741453296814,8.74682763126081,8.568741453296814,7.868981134850038,8.002326078490526,8.74682763126081,7.018222732008108,7.229766311757508,7.854536417854291,8.650342010868975,7.567138829389274,7.732544568593798,8.74682763126081,8.24845062878385,8.24845062878385,8.740720669021316,7.352312886896398,8.737099508537463,8.231322810193722,7.432128715293397,8.737099508537463,8.737099508537463,7.99964502188942,9.323936878558722,8.761315149115639,8.321299954592188,8.321299954592188,9.206000970377085,8.115042584275903,7.632885505395133,8.115042584275903,8.349697190940843,8.349697190940843,9.870845552120199,9.870845552120199,9.870845552120199,9.870845552120199,9.870845552120199,9.870845552120199,9.870845552120199,9.870845552120199,8.157456422013103,8.270985700503475,7.369096497649219,7.915640204017289,8.618141989744455,7.270312886079025,8.051532119604621,9.196616630756372,8.725312441031333,9.196616630756372,9.430559651773146,9.196616630756372,9.196616630756372,9.430559651773146,9.430559651773146,8.140548749116267,7.095478885065086,9.196616630756372,9.196616630756372,9.430559651773146,9.430559651773146,7.361947057180535,8.100707580024608,7.996788501203253,7.361947057180535,8.725312441031333,7.740186026644468,8.049554762876783,9.002516374769804,7.418540901948107,8.262791144199255,8.049554762876783,9.002516374769804,9.002516374769804,9.359018964347545,7.993890046139134,9.359018964347545,10.83122254829454,10.35496118185185,10.35496118185185,10.58583917013487,7.47862182483325,10.83122254829454,11.53912913046686,9.359018964347545,9.359018964347545,8.680807520744496,10.83122254829454,9.235832664737119,10.14296104166755,10.79243791961572,10.53554891119323,10.79243791961572,9.329358200517506,9.773566776640083,9.54052143200899,10.83122254829454,9.564189326565852,10.79243791961572,9.333451410804058,8.680807520744496,8.680807520744496,9.359122380528822,7.194887200128437,10.79243791961572,10.79243791961572,10.79243791961572,10.79243791961572,10.79243791961572,7.67205965385841,8.167493861213931,10.79243791961572,8.229137715924885,8.102798331486911,8.15461514411045,8.816275038818892,9.100904898276326,9.100904898276326,7.287765817617833,8.88798403559038,8.61235790609557,7.369726736530608,7.939693434382254,7.955775781534187,9.100904898276326,8.88798403559038,7.314419667665783,8.288810942053825,7.899598653256963,7.438618796484636,7.407803029576249,7.496263905429513,6.997687386482744,8.15461514411045,7.955775781534187,7.527955351786017,6.842790000129778,8.862256986106123,8.61235790609557,8.372098052798325,8.033917881737253,7.331191272411538,8.372098052798325,7.680960551712661,8.288810942053825,8.372098052798325,9.467993556713555,7.534816074414223,7.04377108289958,8.425604079842328,7.660962215969746,7.468513271496337,7.881295531656987,8.823544896627043,8.222848976090907,7.504831990791349,9.180355278885605,9.180355278885605,8.271625078538326,9.180355278885605,7.928802344032635,8.002861428320958,8.197043022159876,7.842318055643741,7.763233871459534,7.842318055643741,10.60541854224312,10.60541854224312,9.515940944930309,9.515940944930309,10.60541854224312,9.515940944930309,10.60541854224312,9.515940944930309,10.60541854224312,10.60541854224312,10.60541854224312,9.515940944930309,10.60541854224312,9.515940944930309,9.515940944930309,9.515940944930309,10.60541854224312,9.235832664737119,10.03994840011446,10.03994840011446,10.67074681238422,10.67074681238422,9.238461247692838,10.35496118185185,10.06471314603535,9.525742177978104,11.33787778924385,8.743451885630593,8.50584908971558,8.506354666235243,8.646324919696072,8.506354666235243,11.33787778924385,8.088316150297414,11.33787778924385,9.982474318283785,11.33787778924385,7.452054362805667,11.33787778924385,10.19863915843395,10.18309082921989,9.088760136071414,10.18309082921989,11.33787778924385,10.18309082921989,11.33787778924385,8.743404033779912,10.18309082921989,9.959239143931937,10.19863915843395,10.52879476818543,10.52879476818543,11.33787778924385,9.959239143931937,11.33787778924385,11.33787778924385,7.277109500077943,11.33787778924385,11.33787778924385,11.33787778924385,10.18309082921989,11.33787778924385,8.90477942063711,8.958141266769733,8.958141266769733,8.958141266769733,8.958141266769733,8.476517019596669,9.692142870000351,7.736045053484376,9.692142870000351,9.692142870000351,9.692142870000351,9.692142870000351,8.156997772155558,8.107750181938849,9.692142870000351,9.692142870000351,9.574420827301616,8.156997772155558,7.082128665926917,8.107750181938849,9.574420827301616,9.574420827301616,9.574420827301616,8.74682763126081,11.13068066663416,8.122074375362217,9.99601203771172,8.209634888959664,10.91075362782705,10.91075362782705,7.950537682689576,10.02997922436941,7.872988544369307,11.13068066663416,11.40406996005754,11.40406996005754,11.40406996005754,11.40406996005754,9.307049572694215,11.13068066663416,10.90704896111217,11.40406996005754,11.13068066663416,10.02997922436941,11.13068066663416,11.13068066663416,10.41174688619472,10.41174688619472,9.99601203771172,10.91075362782705,10.21482145224664,10.90704896111217,10.90704896111217,10.90704896111217,10.90704896111217,10.91075362782705,9.99601203771172,10.90704896111217,9.838591634206455,9.318297513481275,7.402207588630183,7.792885526606668,11.13068066663416,11.13068066663416,11.40406996005754,7.962241509607692,9.047233034106032,9.838591634206455,11.40406996005754,10.21482145224664,8.853993945498251,10.25208596895518,10.25208596895518,10.55596897637006,10.25208596895518,10.25208596895518,10.55596897637006,10.55596897637006,9.838591634206455,9.838591634206455,8.21451930114522,10.91075362782705,10.41174688619472,10.41174688619472,8.752771114510624,9.99601203771172,11.40406996005754,10.25208596895518,8.217033597453344,11.40406996005754,9.99601203771172,10.55596897637006,10.55596897637006,9.838591634206455,9.435585712982174,7.109143898493318,7.694757035013707,9.194840872929232,9.194840872929232,9.194840872929232,9.194840872929232,8.279215151143571,9.034485858705137,9.034485858705137,7.592214831144517,9.034485858705137,8.480549953717398,8.093828843420241,8.480549953717398,7.7085004701401,8.696192569212887,9.301249974335988,9.301249974335988,9.301249974335988,9.301249974335988,9.301249974335988,9.301249974335988,8.129764445794171,8.129764445794171,9.301249974335988,8.111388025494968,9.233529415748881,8.299012305681773,7.334590412826494,6.93809047877821,10.13865057967454,10.13865057967454,10.13865057967454,9.528685006234781,9.528685006234781,7.815045628658993,7.725374187052922,9.283646594242914,9.283646594242914,9.283646594242914,8.359907602893491,9.283646594242914,9.04337628326326,9.04337628326326,9.04337628326326,7.222566018822171,6.984623723238714,10.41585898093029,10.41585898093029,10.41585898093029,10.41585898093029,10.41585898093029,10.41585898093029,9.389707568943701,10.41585898093029,7.715881567865732,9.733049111028903,9.733049111028903,9.733049111028903,10.41585898093029,10.41585898093029,9.389707568943701,8.13678160637218,7.47584931053852,9.389707568943701,8.569823591479436,8.569463008849919,10.41585898093029,8.725945686435079,8.993539165405553,7.298445101508147,9.733049111028903,8.569463008849919,7.370671350725978,7.547132416332254,7.734121303328305,8.545761211733295,8.314660764006138,8.314660764006138,7.849206806514792,10.05412922585201,8.569463008849919,10.41585898093029,8.562663565460994,8.373299726041628,8.725945686435079,10.05412922585201,8.725945686435079,8.038770218757747,8.013740321979659,8.306422776165169,8.913227300461799,7.908240089652304,9.309262203724961,9.309262203724961,9.331513211306236,9.331513211306236,8.7846980664292,8.7846980664292,9.193255248918369,9.735624959299791,9.735624959299791,9.735624959299791,7.819716652695063,8.078098908327595,9.735624959299791,9.735624959299791,9.735624959299791,9.98764026409912,9.735624959299791,9.735624959299791,9.086012213658792,9.086012213658792,7.173575104576783,9.62136996955676,9.086012213658792,7.818148655590208,7.49681890599876,8.53233792874269,9.088760136071414,7.618447638757831,7.845650934608677,9.075184803949726,9.075184803949726,9.075184803949726,7.194887200128437,7.762255709899841,8.105277321812611,7.416258207378482,6.923530198097499,7.105375873065261],[21.37971729196712,20.57866750647796,21.14167230722138,20.31888700312864,20.73135940051916,21.86093638454719,20.29866815280757,20.88054972087354,21.46750086569128,21.30527904860642,21.54839636621646,21.11325422746425,20.29335809008931,20.55948610355757,20.05910303736902,20.16728275947119,20.51563493613235,21.28699961658899,20.77901727607259,20.42763539439344,21.27346857105808,21.07313758141382,20.06120941661975,20.51309569337041,20.40426253609209,20.45933335176336,21.02533606589655,20.82239030310285,20.4121311765891,20.47953209381348,20.33806283144183,20.69796212322177,20.50753191142797,20.58669667611547,20.44415711077584,20.71968579724481,20.39517190669014,20.44727863743543,20.96484234543777,20.85037284936139,20.6506357778333,21.09647975147814,20.81536121754405,20.33366572116828,20.8912202564173,20.66164519758164,20.94002049386913,20.13674202191363,20.17981523943574,20.27454377063613,20.77264201984951,20.70569840831145,20.84023669665496,20.15647578566679,20.26931844249457,20.27950994216927,20.53683615374458,20.51225099827607,20.74195141901118,20.49663791224767,20.45292117917904,20.14410329615115,20.63657074338845,20.13187116701508,20.26494954156551,20.28710226953923,20.14394338471521,20.19028686139514,20.64211584479295,20.21499387314689,20.140948797931,20.66943902644182,20.22080783511463,20.37205766863714,20.3015807299979,20.07548247542906,20.23121478115891,20.86249293283842,21.49097739866821,21.31327841987607,20.18354591289188,20.53283932838743,20.4841087239766,20.62516676308985,20.22426960422635,20.25837853216262,20.13797130051089,20.90704523890086,20.15528170110517,21.11781036465484,21.07961444394304,20.10428146389384,20.41263582580859,20.39498440907734,20.25726493170178,20.22329780066863,20.05243391066449,20.09166270481245,20.69801624919917,20.24846760715133,20.23340392432772,20.1281952660925,20.24952235917692,20.50662339256776,20.22564541774798,20.47567718518942,20.11230115491991,20.3586700601581,21.07033217240523,20.15428291356132,20.31310006852536,20.70704414384541,20.04094126412126,21.02154887878772,20.06647009364029,21.02154887878772,20.72602580173721,20.05231044892171,20.33771807896527,20.10373675500031,20.40457786464555,20.22292682760058,20.16379755888087,20.18812474371264,20.16190675460591,20.16861031577196,20.52354773351326,20.25655624041294,20.22731563675304,20.48874070674454,21.2466151033016,20.30603917058759,21.31986475684613,21.41953329587063,20.4579729809379,21.00507982557228,20.17659023351667,21.35312926082251,20.80834221227726,21.18171757870028,20.48741315695593,20.80102084706499,20.95966069736057,20.20751234448892,20.26132713616063,20.27896413866029,20.21926536541602,20.31807569853062,20.59508568031352,20.33398274787077,20.4225588041153,20.13968149191666,20.21164409636351,21.67615661911162,20.26962795057191,20.72190263120353,21.25714572042314,20.64405185528459,21.09708568741133,20.20607927188751,20.13945747310019,20.22232826156602,20.06894696756016,20.61167096579459,20.2130985864052,20.44906236664207,20.41526978295915,20.25665690319633,20.38206473042377,20.7379062462265,20.26220754103971,20.59829431756477,20.36254131165302,20.1735540693859,20.03579579091934,20.39010662658796,20.24303165197218,20.46526161466587,20.58227758060555,20.06704771686228,20.29293794654483,20.56142625193602,20.168766117088,21.01042581112861,20.10763663499672,20.36632646662153,21.56493463944422,21.32376306211534,21.33585942335254,21.10953096205184,21.23875962048025,21.48187069000112,20.70293437542206,21.31550033223591,20.42378597141357,21.03712433961089,21.34461125155211,21.47748228539425,20.90972984277211,20.13722061185741,21.16669871719473,21.52887769350408,21.17021439180706,20.62964205996517,20.72884580293941,20.62092256925886,21.07806112363578,21.45982731997108,20.05664214020146,21.38719610815278,20.57099243992063,21.47543641545959,20.22844561562517,20.72256726300044,21.60719485333322,21.40081866874184,20.70864866470942,20.26104663189826,21.03343347116953,20.26593976914871,20.97837464335009,20.97837464335009,20.55364562705296,20.16039869508366,20.68809192961774,20.56885724719658,20.1394508771712,20.65703811818418,20.11607285976494,20.83503205402114,20.9183475217635,21.26242803478391,20.05413209790456,20.35222901751363,20.54196995037288,21.37118620318322,21.14680870392079,21.4926837463577,20.40609024041575,20.75635490047631,20.63141974535439,21.08867356092306,20.71166722434292,21.20575625449582,21.32904670462534,20.91270388633145,20.34011861815028,20.57685929241525,20.92490751249116,20.94514592963281,21.02170507160608,20.15600481202145,21.05038032249723,21.09853824638314,20.80408081526527,20.55031984348217,20.61064212916435,20.60992600885733,21.08833703042283,20.38315589970788,20.1886899827418,21.12689201793072,20.42754618955696,20.76682775859578,20.72100240531659,20.09781656764089,20.39563057439893,20.81649187370536,21.4981291758722,20.31041599619659,21.13979603716874,21.71667329635546,20.36439303806444,20.6152210674561,20.54196995037288,20.88020143215162,21.5192422726256,20.16195691521858,21.07142176761507,20.93024394389188,21.04990973327331,21.30649600788981,20.7927345332108,20.92252410039593,21.12971950826491,20.39741629776255,20.5373136814979,20.52792245865362,20.78362714974713,21.02513301512947,20.15508133924969,21.48253298936628,20.29859147482132,20.71512317124168,20.59939133806719,20.36398570101144,20.80314241451341,20.68851465478177,20.16087680512247,21.25328626332058,20.78550826000466,20.58423629200276,21.13852340998223,20.93469677947546,20.71626411500017,20.24641453259736,20.18722086530604,20.50187530070303,20.47206538616982,20.08679858388119,20.6052004923196,20.35475167378472,20.10854517873581,20.26627190934841,20.83236682221739,20.75418027135962,21.05863545090262,20.50987715431458,20.75968452713585,20.45877996175368,20.63889543931691,20.09540651357664,20.38303332233311,20.17003447387876,20.08841906590106,20.28650912242834,20.2700990877875,20.60457801034012,20.54982615010749,20.56183360971496,20.38284836972736,20.50517726019297,20.9752710862222,20.20566564083049,21.66491156179348,21.29407342298507,21.41647269722549,20.44777808792335,20.53386827054133,20.75241962902707,20.29963862378928,20.59850936329124,21.23537992368736,20.73507620220908,20.0935748532724,20.106723154339,20.06081318626069,20.72210020586238,21.12136521722888,20.67754085594962,20.25540394135092,20.21326353249338,20.83648313480006,21.10615960003443,20.21437536623004,20.44100489955259,20.93198985752282,21.18750835028708,20.06808333297906,20.35385150622286,20.28924220926215,20.5023246785834,20.07326013673844,20.06581979463446,20.25598994005385,20.15402479375752,20.25243214489385,20.77771115956947,20.03654223098487,20.09779865603421,21.52610134142745,21.09758459432452,20.22454466614012,21.22054087521714,20.70008553363968,21.05065316908215,21.13319207407135,21.53651196226275,21.1125371674199,21.38540039763891,20.07120506074736,20.14746788774115,20.13405560565431,20.92533675082686,20.70693922520215,20.52973082935506,20.44433456299512,20.5533193418397,20.32038889265091,20.29444953963709,20.51888359520993,20.47861534807401,20.5000368982368,20.94572354352288,21.1125371674199,20.23077804760291,21.56026520125368,20.61148206740226,21.01018465794589,21.56480415496658,21.70812829339761,21.01136034433521,20.91732795121418,20.26262207832591,20.68442149807544,20.41740650253627,20.80596172459235,20.70722025943167,20.44578883649588,20.66693423684881,20.65842395566647,20.2937273554547,20.42585633138458,20.26349433171771,20.59725681166814,20.23951297795181,20.12478178689954,20.06384369595147,20.6383930395644,20.43650794530795,20.86418986383973,20.10430465063924,20.38944405596239,20.09386280646988,20.84376085850905,20.29419551813708,20.26448343913232,20.88669887946554,20.21413193474556,20.45544354702016,20.72559032722437,20.72284093468823,20.41593432178509,20.14543416486679,20.36010547665631,21.59214622915531,21.54497807854616,21.29358804210688,21.29639350404686,20.79726314983477,20.20802281248321,20.17043358844334,20.61186324353293,20.46171853927065,21.14294399081665,20.34424303810891,20.27352811433308,20.48538791218675,20.07842081578665,20.2586684571632,20.43802362428122,20.89439960062149,21.39947960663914,21.18360775052311,20.83012677550661,20.6934361384958,21.24188850738754,21.20853291175233,20.03059833332309,20.26135874530081,20.20561132933864,20.93295538285385,20.41569434279193,20.94410858744721,20.46745034319438,20.36212921450645,20.31426870257079,20.31725401141551,20.08084022657088,20.60640492337974,20.43065528345949,20.89583355264849,20.11257930985618,20.42142806095082,20.43065528345949,20.4555860594376,20.7841606584909,20.80526895560416,20.8632672975913,21.38722137590809,21.54108134703097,20.09154887362375,20.32104793894025,21.228504217096,20.96422481571578,21.35742105692294,21.67170132673069,21.15782426033309,20.74794876070936,20.57829063240372,20.76955364903324,20.55250402569778,21.56455931965415,21.25992801952721,21.56207722502081,21.63827096718881,20.66628175649775,20.54178283726515,20.3221393344628,20.8180320051669,21.17785335497665,21.50644308246105,20.22329718076905,20.72519080400792,20.27926471166716,20.62317994154435,20.03469511640016,20.09934850912448,21.11262571798201,20.44762543041369,20.30982855786106,20.28080444911744,20.10346647584018,20.18498215752481,20.37954470323988,21.39734932249246,20.04728097367889,20.89414264759637,21.85591885476012,20.35434088712771,20.60150655800202,20.44805679712687,21.23878609126951,20.45488128777667,20.8632183498575,20.87353344970326,20.31161064775624,20.28399025387703,20.82668557341007,20.58142552119884,20.81057362466871,20.64822405585188,20.21539325033773,20.19085395225509,20.75848101846582,20.39080534391021,20.03451781986197,20.08563698891692,20.15298095560489,20.21228506614574,20.49695101371075,20.05785645403229,20.72433835059798,20.30732527233073,20.0360517248753,20.49125355443918,20.45674407495054,20.03098094850574,20.19892493859299,20.03603258260101,21.15305663049802,20.56392582579992,21.35692050170975,20.22779929954884,20.26261941377922,20.31171772784551,20.9151369380187,20.22628034024478,20.19969060288116,20.05203076109143,20.24056500714087,20.0896975381956,20.72536853273136,20.23475043077056,20.03106673281994,20.25578920415399,20.92817785201032,20.28243918793126,20.0911008105159,20.41806225953944,21.37358949052657,20.46211974240856,20.77972732587949,20.5322834415307,20.34130856571262,20.20648444273699,20.39184255626874,20.35257563942498,20.97194305272622,20.17173173652233,20.18130074344197,20.05823759533774,20.55553135950493,20.12452107338969,20.2385536127228,20.28420490007973,20.15892269440715,20.47764706443644,20.73883188593809,20.11845662489767,20.25971427626824,20.29144855377304,20.68602949586391,21.15726656058867,21.34952485866416,21.17212236409463,20.27501395696898,20.19642391767445,20.30749382522557,20.0431305789495,20.41888449527758,21.02003590344682,20.34677978684698,20.87325623851886,20.26668731195598,20.57580897809383,20.34140199979045,21.07849014992864,21.06413024242822,21.01365539385206,20.12599030944427,20.67474334140054,20.0694016176023,20.4115033335495,20.44527714220616,21.33246078274788,20.23554010766192,20.17287066665852,20.32730049285489,20.16776016963123,20.59592671373026,20.73367640741763,20.19906057366407,20.16634557880412,20.54640153342121,20.2824198405525,20.66098554914617,20.41623911207195,20.4115065910494,20.24848249424789,20.50006540526454,20.30788516723562,21.23711672597929,20.8773752224925,20.62960428301133,20.12602750059186,20.07796584275872,21.26427883002446,20.95440268775277,21.67143806297989,20.26649116116039,20.49556075013022,20.20210295232949,21.2229097463036,20.94174544576977,20.28382125205752,20.04181047590819,20.86249165650297,20.07178918551938,20.51499431054996,20.05919634873845,20.86766476404542,20.64593360120609,20.47402181375411,20.06375222895895,22.20892547619821,20.8683695639431,21.23643961703419,21.224420953321,21.84180088817783,22.10741877410378,20.80233001678694,21.9769087287671,20.81308796947576,21.59222573938403,20.39411892867395,21.1355950643268,20.59924993099764,21.0544161989092,20.99747198487949,20.70768846929527,21.23822407448879,21.35970389130983,21.34133365047059,21.75773456646919,21.45444829224739,21.20405896279277,21.10932903606009,20.30689970701159,21.01855912993195,21.20812189207058,21.44678265306962,20.46072705405146,20.4555612993694,20.50339841201491,21.49268242090513,21.30852656658062,20.5696670864448,21.63770489713419,21.53545082516254,21.99013063306164,20.92789295389558,21.21810371631242,21.59144212348173,20.68820160651822,20.91165774598098,20.64140563529753,21.12632385767616,21.21919130955408,20.71322332631482,21.07461688305766,21.28376206460958,20.39164850657291,20.48208221771524,20.24183341595061,21.16490110872746,20.38229907666888,21.0422765423805,20.04307999021112,21.5267746008524,21.24247736393209,20.51099435615677,21.67488390877689,21.62028297586504,21.41579970246713,20.90847454191415,20.87887930523392,20.90038575866294,20.40586722137265,20.83408887234003,20.63851560164804,21.01577812136941,21.0422765423805,20.04307999021112,20.90023923195115,21.80545613312767,20.69166854394366,20.07759421779659,20.42667922583679,20.23661702447549,20.63591925941972,20.05678080718689,20.1718228904218,20.52317395505047,20.88338413307835,20.21808888767034,20.45429808607543,20.44520344398899,20.18505216286783,20.43439410913523,20.07711550113846,20.2128679900042,20.55620743136482,20.21998692167282,20.45777588500347,20.84387659438204,20.40404086487301,20.71748438249988,20.50719995473677,20.04051179413668,20.15604022813122,20.08863579973526,20.43362959501527,20.58304264121027,20.22026263312132,20.03791593841931,20.95416682062689,20.23260187242214,20.60278844715068,20.3125562249377,20.31609897367395,21.1695151046191,20.30366019405957,20.50636538064576,20.76600699317716,20.70935819650221,20.43881407209518,20.17884281869253,20.65722095515043,20.55670806228288,20.06919568806421,20.86606350510071,20.62521088720513,21.62636912911551,21.85655227874571,20.99425077105074,20.9944634865524,21.58759469519424,20.80946692832664,20.45229466438217,20.72471691761942,20.83951961232458,21.33148232620636,21.31322221298868,21.59084285518872,21.36526254870822,20.96115728780058,20.6247067082187,20.88038178715357,21.01745346605146,21.22211877286919,21.49510125090902,20.95922653734584,20.39866445154054,20.25854092592014,21.2171784114296,20.83087319556108,20.09273640965493,20.24705579892865,20.55231099557806,21.01825708442834,21.06135055995525,21.1963049563019,20.9588661705858,20.46878342848255,21.03626727602502,21.66373546235795,20.27898422510133,20.42115149585351,20.94661095994368,20.63925870741014,20.37951133205748,20.27236403890839,21.1394869479665,21.32741929770016,21.43014947785771,21.34208737381775,20.27784105147816,20.05183309618042,20.0488835748608,20.14981158969332,20.10252454224138,20.20301545097519,20.50309358694113,20.04245162570881,21.61927809536756,21.07329763507283,20.46996961105839,20.83250329494169,20.69578838065088,20.77925073587589,20.56793904894166,20.42701818308658,20.15529052459652,20.85685637707169,20.76908047577066,20.27750856893084,20.31922601813072,20.83660313293063,20.84585609935064,20.24285651174456,20.59988542944151,20.66903586632207,20.19242412790371,20.0750561552292,20.33663613264221,20.21254717079358,20.29867070925078,20.52909839872592,21.05797672043838,20.19384347071279,20.30305244610532,20.23960821961646,21.28674014150866,20.66608437389965,20.57404229896206,20.31055787860933],[282,282,66,416,66,66,66,471,129,410,178,81,178,178,81,66,341,177,821,177,290,332,440,440,543,756,129,81,958,734,644,644,398,1062,688,679,1285,129,81,817,778,81,919,398,611,737,231,1330,1770,947,81,81,288,177,129,1248,416,231,309,416,81,288,1018,488,944,423,178,681,1054,636,423,309,231,981,231,636,488,545,62,62,85,54,543,62,715,108,769,474,694,54,262,62,62,62,62,62,1249,62,262,54,262,262,543,324,262,324,543,210,210,210,210,401,550,389,708,389,1125,1032,389,204,791,937,1656,621,1439,1367,468,475,706,29,29,29,29,29,29,29,29,29,29,29,579,279,138,138,138,29,78,29,295,722,1356,279,905,94,666,119,92,92,92,119,352,352,388,770,600,1070,822,592,778,208,663,114,253,1657,418,59,388,59,921,114,253,59,601,25,23,107,3,257,58,3,5,3,3,8,8,8,69,45,5,107,51,45,61,4,3,5,4,48,525,7,7,7,58,3,69,45,85,14,14,14,14,14,14,5,8,58,14,23,14,23,23,85,22,157,98,3,3,3,77,45,45,119,129,3,205,48,37,98,38,22,160,18,1101,227,3,78,57,525,69,205,340,57,57,57,676,85,59,5,4,22,205,3,58,66,98,98,4,11,3,3,45,8,77,5,162,7,675,6,428,3,309,4,107,3,81,112,22,23,66,240,158,1260,13,214,13,1314,240,611,158,1465,214,1331,1464,1784,105,749,594,51,69,318,338,350,318,162,440,1043,350,338,513,257,259,516,259,463,1,251,100,100,1,1,1,1,1,1,339,251,257,1,1,321,321,428,1,317,428,428,309,13,1259,1,48,1,1,1,259,48,321,13,317,1,67,67,67,67,67,67,67,67,67,67,67,67,67,129,389,129,129,251,129,1064,251,387,389,387,67,129,152,480,165,422,117,165,291,132,291,680,958,132,1117,1575,348,326,1101,1077,132,488,488,117,628,629,333,1858,629,629,592,71,114,325,325,189,219,631,219,468,468,41,41,41,41,41,41,41,41,514,417,964,874,681,1682,778,135,201,135,77,135,135,77,77,475,1240,135,135,77,77,646,805,924,646,201,493,615,202,1719,1158,615,202,202,61,756,61,5,22,22,22,924,5,3,61,61,105,5,29,45,6,8,6,50,103,99,5,91,6,275,105,105,89,1094,6,6,6,6,6,390,826,6,614,617,110,132,67,67,1361,134,337,711,521,856,67,134,1183,304,753,1159,951,1078,1106,110,856,463,1002,71,337,290,1504,844,290,844,304,290,51,410,421,314,1492,808,1222,685,760,584,13,13,671,13,331,69,40,317,87,317,63,63,173,173,63,173,63,173,63,63,63,173,63,173,173,173,63,29,53,53,29,29,344,22,64,66,3,404,440,159,332,159,3,462,3,3,3,750,3,34,56,239,56,3,56,3,270,56,128,34,40,40,3,128,3,3,892,3,3,3,56,3,475,143,143,143,143,683,23,424,23,23,23,23,314,636,23,23,16,314,1662,636,16,16,16,132,1,150,58,224,2,2,82,13,530,1,1,1,1,1,116,1,3,1,1,13,1,1,4,4,58,2,21,3,3,3,3,2,58,3,18,66,847,968,1,1,1,473,26,18,1,21,167,22,22,1,22,22,1,1,18,18,31,2,4,4,145,58,1,22,360,1,58,1,1,18,69,406,552,258,258,258,258,161,268,268,1215,268,354,1081,354,835,309,104,104,104,104,104,104,715,715,104,788,268,717,674,1112,21,21,21,137,137,549,975,170,170,170,901,170,63,63,63,538,597,18,18,18,18,18,18,165,18,820,118,118,118,18,18,165,836,705,165,144,231,18,214,183,1345,118,231,1178,598,831,171,228,228,783,30,231,18,331,553,214,30,214,224,180,151,419,841,327,327,32,32,145,145,107,102,102,102,1043,475,102,102,102,29,102,102,117,117,784,56,117,150,793,519,239,875,672,105,105,105,742,860,238,937,883,348],[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,1,1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,1,1,0,0,0,0,0,0,0,0,0,0,0,0,1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,1,0,0,0,1,0,0,0,0,0,0,1,0,0,0,0,0,1,0,0,0,0,0,0,0,0,0,1,1,0,0,0,0,0,0,0,0,0,1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,1,0,0,0,1,0,0,0,0,0,0,0,0,0,0,0,1,0,0,0,0,0,1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,1,0,0,1,0,0,1,1,0,0,0,0,0,0,0,1,0,0,0,0,0,0,0,0,1,0,0,0,0,0,1,0,0,0,0,0,0,1,0,0,1,0,0,0,0,0,0,0,0,0,1,1,1,0,0,0,0,0,0,0,1,0,0,0,0,0,0,1,1,0,0,1,0,0,0,0,0,0,0,0,0,0,0,0,1,0,0,0,0,0,0,0,1,0,0,0,0,0,0,1,1,0,0,0,1,1,0,0,0,0,1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,1,0,0,0,0,0,1,1,1,1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,1,0,0,1,0,0,0,1,0,0,0,0,1,1,1,1,0,0,0,1,1,1,1,1,0,1,1,1,1,0,1,0,0,0,0,0,0,0,1,0,1,0,1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,1,0,0,0,0,0,0,0,1,1,1,0,0,0,1,1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,1,0,1,0,0,0,0,0,0,1,0,0,0,0,0,0,0,0,1,1,1,0,0,0,1,1,0,1,1,0,0,1,1,1,1,0,0,1,1,1,1,1,1,1,1,1,1,1,1,1,0,1,0,0,1,1,0,1,1,1,0,0,1,1,0,0,0,0,0,0,0,0,0,0,1,1,1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,1,0,0,0,0,0,0,0,1,0,0,1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,1,1,0,0,0,0,0,1,0,0,0,0,0,0,0,0,0,1,0,0,0,1,1,0,1,0,0,0,0,0,0,0,1,0,0,1,0,0,0,0,1,1,1,1,0,1,0,0,0,0,1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,1,0,0,0,0,0,0,0,0,0,0,1,1,0,0,0,0,0,0,0,0,0,0,0,0,1,0,0,0,0,0,0,0,1,0,0,0,0,1,0,0,0,1,0,0,1,1,0,0,0,0,0,0,0,0,0,0,0,1,1,1,0,0,0,0,0,0,0,0,0,0,0,1,0,0,0,0,0,1,0,0,0,1,1,0,1,0,0,1,1,0,0,0,0,1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,1,1,0,0,1,1,0,0,0,1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0],[4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,3,4,4,3,4,4,3,4,3,3,4,4,4,4,4,4,4,4,4,4,3,3,4,3,4,4,4,4,4,4,3,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,3,3,3,4,4,4,4,4,3,4,4,4,4,4,4,4,3,4,4,3,4,4,4,4,4,4,4,4,4,4,4,4,4,3,4,4,3,4,4,4,4,3,4,3,3,4,4,4,4,4,4,4,3,4,3,4,3,4,3,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,3,4,4,4,3,3,3,3,3,3,4,4,4,3,3,4,4,4,3,4,4,4,4,3,4,3,4,3,3,3,4,4,4,3,4,3,3,3,3,3,3,3,3,3,3,3,3,3,3,4,4,4,4,4,4,4,4,4,4,4,3,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,2,3,3,3,4,2,1,4,4,4,2,4,3,3,3,3,4,4,4,2,4,3,4,4,4,4,4,3,3,3,3,3,4,4,3,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,3,3,4,4,3,4,3,4,3,3,3,4,3,4,4,4,3,4,4,4,3,3,4,3,4,4,1,4,4,4,4,4,1,4,1,4,1,4,1,3,3,4,3,1,3,1,4,3,4,3,3,3,1,4,1,1,4,1,1,1,3,1,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,2,4,4,4,2,2,4,4,4,2,1,1,1,1,4,2,2,1,2,4,2,2,3,3,4,2,3,2,2,2,2,2,4,2,4,4,4,4,2,2,1,4,4,4,1,3,4,3,3,3,3,3,3,3,4,4,4,2,3,3,4,4,1,3,4,1,4,3,3,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,3,3,3,4,4,4,4,4,4,4,4,4,4,4,4,4,4,3,3,3,3,3,3,4,3,4,4,4,4,3,3,4,4,4,4,4,4,3,4,4,4,4,4,4,4,4,4,4,4,4,4,4,3,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4]],"container":"<table class=\"display\">\n  <thead>\n    <tr>\n      <th> <\/th>\n      <th>daily<\/th>\n      <th>daily1<\/th>\n      <th>monthly_listeners<\/th>\n      <th>artist_streams<\/th>\n      <th>streams<\/th>\n      <th>months_sincepk<\/th>\n      <th>explicit<\/th>\n      <th>popular_artist<\/th>\n    <\/tr>\n  <\/thead>\n<\/table>","options":{"columnDefs":[{"className":"dt-right","targets":[1,2,3,4,5,6,7,8]},{"orderable":false,"targets":0},{"name":" ","targets":0},{"name":"daily","targets":1},{"name":"daily1","targets":2},{"name":"monthly_listeners","targets":3},{"name":"artist_streams","targets":4},{"name":"streams","targets":5},{"name":"months_sincepk","targets":6},{"name":"explicit","targets":7},{"name":"popular_artist","targets":8}],"order":[],"autoWidth":false,"orderClasses":false}},"evals":[],"jsHooks":[]}</script>
```


<br><br><br><br>

#####

#### boxCox transformation

##### {.tabset .tabset-fade .tabset-pills}

###### Original View

<br>


```r
dt$daily1 <- exp(dt$daily1)
dt$monthly_listeners <- exp(dt$monthly_listeners)

mylm1 <- lm(daily1 ~ monthly_listeners, dt)

par(mfrow = c(1, 2))

boxCox(mylm1)
plot(dt$monthly_listeners, dt$daily1, col = "orange", pch = 19, xlab = "Monthly Listeners", ylab = "Daily Streams (Millions)")
```

<img src="Spotify-Regression_files/figure-html/raw-1.png" style="display: block; margin: auto;" />

```r
par(mfrow = c(1, 1))
```

<br>

After trying `daily1 ~ monthly_listeners` I got the above boxCox suggestions (left). To the right you can see the plot in its original format.

This suggestion is closer to a sqrt or log transformation. I tested both, but even though sqrt got me a higher R^2^, I went ahead and tried log. Next tab shows how the plot looked like after Y transformation. As a note log y transformation took me to .78 r^2^ while sqrt got me to .82 R^2^. 

<br><br><br><br>

###### Y transformation

<br>

This is how the data looked like after taking the log of Y.

<br>


```r
dt$daily1 <- log(dt$daily1)


mylm1 <- lm(daily1 ~ monthly_listeners, dt)

par(mfrow = c(1, 1))
plot(dt$monthly_listeners, dt$daily1, col = "orange", pch = 19, xlab = "Monthly Listeners", ylab = "Daily Streams (Log Millions)")
```

<img src="Spotify-Regression_files/figure-html/y transformed-1.png" style="display: block; margin: auto;" />

```r
par(mfrow = c(1, 1))
```

<br>

This is how the data looked like after taking the log of Y.

For the first Y transformation discussed in the previous tab, I chose log even though sqrt had a higher R^2^ because, when I looked at the data plotted this way, it looked like it could support an x transformation. If I followed the path of sqrt I would have lost interpretability and the chance to support an x transformation. I tried a log x transformation. See next.

<br><br><br><br>

###### X transformation

<br>

This is how the plot looks after a double log.

<br>


```r
dt$monthly_listeners <- log(dt$monthly_listeners)

mylm1 <- lm(daily1 ~ monthly_listeners, dt)

par(mfrow = c(1, 1))

plot(dt$monthly_listeners, dt$daily1, col = "orange", pch = 19, xlab = "Monthly Listeners (Log)", ylab = "Monthly Streams (Log Millions)")
```

<img src="Spotify-Regression_files/figure-html/final transformed-1.png" style="display: block; margin: auto;" />

<br>


```r
pander::pander(summary(mylm1))
```


--------------------------------------------------------------------
        &nbsp;           Estimate   Std. Error   t value   Pr(>|t|) 
----------------------- ---------- ------------ --------- ----------
    **(Intercept)**       -23.29      0.2893     -80.51       0     

 **monthly_listeners**    1.455      0.01694      85.89       0     
--------------------------------------------------------------------


--------------------------------------------------------------
 Observations   Residual Std. Error   $R^2$    Adjusted $R^2$ 
-------------- --------------------- -------- ----------------
     857              0.3823          0.8961       0.896      
--------------------------------------------------------------

Table: Fitting linear model: daily1 ~ monthly_listeners

<br>

So, in the **log world**, there is an awesome correlation between these two variables. I didn't need to take out outliers, nor filter many things to get to this point. And even though this is not the final model, this how it began to be formed. The numbers on the x Axis were later fixed to represent natural world quantities.

Continue to model summary to keep on reading about this model, or read about an alternative model where I tried `daily ~ streams`. where the daily streams by song (instead of by artist) was being explained by the total streams the song has had.  

<br><br><br><br>

#####
 
#### Alternative Model
 
##### {.tabset .tabset-fade .tabset-pills}

###### Introduction

<br>


```r
dtas <- read_csv("https://github.com/Tonadeleon/Regression-Projects/raw/main/Datasets/dataset.csv")|>
  filter(popularity > 59) |>
  rename(title = track_name,
         artist = artists,
         title_id = track_id,
         album = album_name,
         loud = loudness,
         words = speechiness,
         acoustics = acousticness,
         instrumental = instrumentalness,
         live = liveness) |>
  inner_join(df1, by="artist", relationship =
  "many-to-many") |>

  inner_join(df2, by="artist", relationship =
  "many-to-many")


dtss <- inner_join(dtas, df, by = c("title", "artist"), relationship =
  "many-to-many") |>
  mutate(explicit = as.factor(explicit),
         genre = as.factor(track_genre),
         daily = daily,
         #streams = ifelse(streams>3825666400, 942193161, streams),
         #times = as.factor(time_signature),

         #energy = ifelse(energy>.5,1,0),
         #duration = ifelse(between(duration_ms, 200000, 250000), 1, 0),
         #loud = ifelse(loud>-9, 1, 0),
         dance = ifelse(danceability>.5,1,0),
         words = ifelse(between(words, .025, .06), 1, 0),
         instrumental = ifelse(instrumental>0,1,0),
         #popularity = ifelse(popularity>76,1,0),
         live = ifelse(live<.37,1,0),
         key = case_when(key %in% c(0, 1, 6, 8, 11) ~ 1,TRUE ~ 0),
         explicit = case_when(explicit == "TRUE" ~ 1,TRUE ~ 0),
         genre = case_when(genre %in% c("pop", "chill", "indie", "garage", "ambient", "folk", "indie-pop", "power-pop", "reggae", "soul", "synth-pop", "swedish", "country", "metal", "german", "funk", "alternative", "alt-rock") ~ 1,TRUE ~ 0),
         peak_last_year = ifelse(months_sincepk<13,1,0),
         y = ifelse(popularity >79 ,1 ,0),
         popularity0 = round(popularity / 5) * 5,
         popularity0 = case_when( popularity0 > 84 ~ 1, popularity == 80 ~ 2, TRUE ~ 3 ),
         popularity1 = round(popularity / 10) * 10,
         popularity1 = case_when( popularity1 == 0 ~ 1, popularity1 == 70 ~ 2, popularity1 == 80 ~ 3,  popularity1 == 90 ~ 4, TRUE ~ 5 ),
         times = case_when(time_signature %in% c(4) ~ 1,TRUE ~ 0),
         energy = round(energy*10),
         popular_artist = case_when(artist_streams > 75000 ~ 1, between(artist_streams, 50000,75000) ~ 2, between(artist_streams, 25000,50000) ~ 3, artist_streams < 25000 ~ 4, TRUE ~ 0 ),
         diff = peak_monthly_listeners - monthly_listeners,
         current_hit = ifelse(diff>20000000,1,0),
         current_trend = ifelse(daily_average<0,0,1)) |>
         #popularity = ifelse(popularity>76,1,0)) |>
  #distinct(artist, .keep_all = TRUE) |>
  distinct(title, .keep_all = TRUE) |>
  filter(months_sincepk < 120) |>
  filter(!(row_number() %in% c(159))) |>
  filter(!(row_number() %in% c(108,297,324))) |>
  filter(!(row_number() %in% c(278))) |>
  filter(!(row_number() %in% c(245,215))) |>
  filter(!(row_number() %in% c(200))) |>
  filter(!(row_number() %in% c(208))) |>
  dplyr::select(daily1, streams, daily, popularity0, popularity1,  explicit, dance, energy, key, loud, mode, words, acoustics, instrumental, live, valence, tempo, times, genre, duration_ms, popular_artist, monthly_listeners, peak_monthly_listeners, months_sincepk, peak_last_year, daily_average, diff,current_hit,current_trend, artist_streams)


#dt$daily <- sqrt(dt$daily)
# dt1$daily <- sqrt(dt1$daily)
# dt2$daily <- sqrt(dt2$daily)
# dt3$daily <- sqrt(dt3$daily)

dtss1 <- dtss |>
  filter(popularity0 == 1)

dtss2 <- dtss |>
  filter(popularity0 == 2)

dtss3 <- dtss |>
  filter(popularity0 == 3)

#consider sqrt transf

mylms <- lm(((daily)) ~

            #streams+ #x

             popularity0 +

             popular_artist:current_trend +

             popularity0:genre +

             monthly_listeners +

             months_sincepk +

             streams:words +

             streams:popular_artist +

             streams:current_trend:peak_last_year

, dtss)


c <-coef(mylms)




g1 <- ggplot(dtss1, aes((streams),(daily)))+

  geom_point(show.legend = T, col="orange") +

  #geom_smooth(method = "lm")+

    stat_function(fun=function(
    x, popularity=1, monthly_listeners= 55517581, months_sincepk= 36.50862, popular_artist= 3.232759, current_trend= 0.2327586, genre=0.4482759, words=0.5689655, peak_last_year= 0.3706897)
    
    (b[1] + c[2]*popularity + c[3]*monthly_listeners + c[4]*months_sincepk + c[5]*popular_artist*current_trend + c[6]*popularity*genre) +
      
    (c[7]*words + c[8]*popular_artist + c[9]*current_trend*peak_last_year)*(x), col="orange", linewidth =1)+

  scale_y_continuous(breaks = seq(0, 2000000, by=500000), labels=c("0","500,000","1,000,000","1,500,000","2,000,000"))+

  scale_x_continuous(breaks = c(1000000000, 2000000000, 3000000000, 4000000000), labels = c("1 B", "2 B", "3 B", "4 B"))+

  theme_classic() +

  labs(
    title = "Popularity group: 1",
    subtitle = "Monthly listeners: 55mill, Months since artist peak: 36, \nArtist peaked last year: .4, artist popularity: 3, \nCurrent Trend: .25, Genre: .45, words: .6",
    x= "",
    y = "") +

  theme(
    plot.title = element_text(size = 13, color = "grey15"),
    axis.text.x = element_text(color = "grey35"),
    axis.text.y = element_text(color = "grey35"),
    plot.subtitle = element_text(color = "grey25", size = 10),
    axis.title = element_text(color = "grey15", size = 10))

g2 <- ggplot(dtss2, aes((streams),(daily)))+
   geom_point(show.legend = T, col="#9EA1F9") +
   #geom_smooth(method = "lm")+

   stat_function(fun=function(x, popularity=2, monthly_listeners= 48407097, months_sincepk= 49.54783, popular_artist= 3.565217, current_trend= 0.2608696, genre=0.2782609, words=0.6, peak_last_year= 0.2956522)

     (c[1] + c[2]*popularity + c[3]*monthly_listeners + c[4]*months_sincepk + c[5]*popular_artist*current_trend +c[6]*popularity*genre) +

     (c[7]*words + c[8]*popular_artist + c[9]*current_trend*peak_last_year)*(x), col="#9EA1F9", linewidth =1)+

    scale_y_continuous(breaks = seq(300000, 1200000, by=300000), labels=c("300,000", "600,000","900,000", "1,200,000"))+

    scale_x_continuous(breaks = c(500000000, 1000000000, 1500000000, 2000000000, 2500000000), labels = c(".5 B", "1 B", "1.5 B", "2 B", "2.5 B"))+

   theme_classic() +

   labs(
     title = "Popularity group: 2",
     subtitle = "Monthly listeners: 48 mill, Months since artist peak: 50, \nArtist peaked last year: .3, artist popularity: 3.5, \nCurrent Trend: .26, Genre: .28, words: .6",
     x= "",
     y = "") +

  theme(
    plot.title = element_text(size = 13, color = "grey15"),
    axis.text.x = element_text(color = "grey35"),
    axis.text.y = element_text(color = "grey35"),
    plot.subtitle = element_text(color = "grey25", size = 10),
    axis.title = element_text(color = "grey15", size = 10))

 ## 3


g3 <- ggplot(dtss3, aes((streams),(daily)))+
   geom_point(show.legend = T, col="steelblue") +
   #geom_smooth(method = "lm")+

   stat_function(fun=function(x, popularity=3, monthly_listeners= 46092154, months_sincepk= 45.65, popular_artist= 3.67, current_trend= 0.44, genre=0.19, words=0.57, peak_last_year= 0.17)

     (c[1] + c[2]*popularity + c[3]*monthly_listeners + c[4]*months_sincepk + c[5]*popular_artist*current_trend +c[6]*popularity*genre) +

     (c[7]*words + c[8]*popular_artist + c[9]*current_trend*peak_last_year)*(x), col="steelblue", linewidth =1)+

    scale_y_continuous(breaks = seq(0, 1500000, by=500000), labels=c("0","500,000","1,000,000","1,500,000"))+

     scale_x_continuous(breaks = c(500000000, 1000000000, 1500000000, 2000000000, 2500000000), labels = c(".5 B", "1 B", "1.5 B", "2 B", "2.5 B"))+

   theme_classic() +

   labs(
     title = "Popularity group: 3",
     subtitle = "Monthly listeners: 46 mill, Months since artist peak: 45, \nArtist peaked last year: .2, artist popularity: 3.6, \nCurrent Trend: .44, Genre: .2, words: .6",
     x= "",
     y = "") +

  theme(
    plot.title = element_text(size = 13, color = "grey15"),
    axis.text.x = element_text(color = "grey35"),
    axis.text.y = element_text(color = "grey35"),
    plot.subtitle = element_text(color = "grey25", size = 10),
    axis.title = element_text(color = "grey15", size = 10))
 
 grid.arrange(g1, g2, g3,
              ncol = 3,
              top = textGrob("Spotify's Streaming Correlation", gp = gpar(fontsize = 12, col = "black")),
              left = textGrob("Daily Streams", rot = 90, gp = gpar(fontsize = 12)),
              bottom = textGrob("All Time Streams", gp = gpar(fontsize = 12))
              )
```

<img src="Spotify-Regression_files/figure-html/alternative model-1.png" style="display: block; margin: auto;" />

<br>

$$\text{Math equation:} \\\ \\ \text{Song daily streams} = \text{Popularity} + \text{monthly listeners} \\ + \text{months since peak} \\ + \text{Popular artist influence} \times \text{Current trend} \\ + \text{popularity} \times \text{genre} \\ + \text{All time streams} \times \text{words} \\ + \text{All time streams} \times \text{popular artist} \\ + \text{All time streams} \times \text{Current trend} \times \text{Peak last year} $$

<br>

This was my initial model. Before I found any data from Kworb's project, I found some data in Kaggle that was interesting to check. This dataset contained many variables that were available in SpotifyR (Tempo, Keys, Loudness, etc..). but since SpotifyR is not being maintained anymore; the get_all_artists function will only work for 50 artists, which is not much. this is why I looked up for more information. But before that I did some work and got a decent analysis on their data.

The model was trying to explain, song by song, what made a song more or less popular, not only in their streamings, but also because the model uses multiple variables. For example, song popularity, artist popularity, current artist trend, use or swear words or not, all of these, and others were useful for this model

Popularity had a big role in this model, Spotify offers an expert rating on song popularity 0-100. I worked that rating up so that 3 popularity groups were considered. I wont test hypothesis on this model, but feel free to check how it went overall in terms of significance.

<br><br><br><br>
 
###### Model Selection

<br>


```r
dtag <- read_csv("https://github.com/Tonadeleon/Regression-Projects/raw/main/Datasets/dataset.csv")|>
  #filter(popularity > 59) |>
  rename(title = track_name,
         artist = artists,
         title_id = track_id,
         album = album_name,
         loud = loudness,
         words = speechiness,
         acoustics = acousticness,
         instrumental = instrumentalness,
         live = liveness) |>
  inner_join(df1, by="artist", relationship =
  "many-to-many") |>

  inner_join(df2, by="artist", relationship =
  "many-to-many")


dtssg <- inner_join(dtag, df, by = c("title", "artist"), relationship =
  "many-to-many") |>
  mutate(explicit = as.factor(explicit),
         genre = as.factor(track_genre),
         daily = daily,
         #streams = ifelse(streams>3825666400, 942193161, streams),
         #times = as.factor(time_signature),

         #energy = ifelse(energy>.5,1,0),
         #duration = ifelse(between(duration_ms, 200000, 250000), 1, 0),
         #loud = ifelse(loud>-9, 1, 0),
         dance = ifelse(danceability>.5,1,0),
         words = ifelse(between(words, .025, .06), 1, 0),
         instrumental = ifelse(instrumental>0,1,0),
         #popularity = ifelse(popularity>76,1,0),
         live = ifelse(live<.37,1,0),
         key = case_when(key %in% c(0, 1, 6, 8, 11) ~ 1,TRUE ~ 0),
         explicit = case_when(explicit == "TRUE" ~ 1,TRUE ~ 0),
         genre = case_when(genre %in% c("pop", "chill", "indie", "garage", "ambient", "folk", "indie-pop", "power-pop", "reggae", "soul", "synth-pop", "swedish", "country", "metal", "german", "funk", "alternative", "alt-rock") ~ 1,TRUE ~ 0),
         peak_last_year = ifelse(months_sincepk<13,1,0),
         y = ifelse(popularity >79 ,1 ,0),
         popularity0 = round(popularity / 5) * 5,
         popularity0 = case_when( popularity0 > 84 ~ 1, popularity == 80 ~ 2, TRUE ~ 3 ),
         popularity1 = round(popularity / 10) * 10,
         popularity1 = case_when( popularity1 == 0 ~ 1, popularity1 == 70 ~ 2, popularity1 == 80 ~ 3,  popularity1 == 90 ~ 4, TRUE ~ 5 ),
         times = case_when(time_signature %in% c(4) ~ 1,TRUE ~ 0),
         energy = round(energy*10),
         popular_artist = case_when(artist_streams > 75000 ~ 1, between(artist_streams, 50000,75000) ~ 2, between(artist_streams, 25000,50000) ~ 3, artist_streams < 25000 ~ 4, TRUE ~ 0 ),
         diff = peak_monthly_listeners - monthly_listeners,
         current_hit = ifelse(diff>20000000,1,0),
         current_trend = ifelse(daily_average<0,0,1)) |>
         #popularity = ifelse(popularity>76,1,0)) |>
  #distinct(artist, .keep_all = TRUE) |>
  distinct(title, .keep_all = TRUE) |>
  #filter(months_sincepk < 120) |>
  filter(!(row_number() %in% c(159))) |>
  filter(!(row_number() %in% c(108,297,324))) |>
  filter(!(row_number() %in% c(278))) |>
  filter(!(row_number() %in% c(245,215))) |>
  filter(!(row_number() %in% c(200))) |>
  filter(!(row_number() %in% c(208))) |>
  dplyr::select(daily1, streams, daily, popularity0, popularity1,  explicit, dance, energy, key, loud, mode, words, acoustics, instrumental, live, valence, tempo, times, genre, duration_ms, popular_artist, monthly_listeners, peak_monthly_listeners, months_sincepk, peak_last_year, daily_average, diff,current_hit,current_trend, artist_streams, popularity)




ggplot(dtssg, aes((popularity),(daily)))+

  geom_point(show.legend = T, col="firebrick") +

  #geom_smooth(method = "lm")+

    # stat_function(fun=function(
    # x, popularity=1, monthly_listeners= 55517581, months_sincepk= 36.50862, popular_artist= 3.232759, current_trend= 0.2327586, genre=0.4482759, words=0.5689655, peak_last_year= 0.3706897)
    
    # (b[1] + c[2]*popularity + c[3]*monthly_listeners + c[4]*months_sincepk + c[5]*popular_artist*current_trend + c[6]*popularity*genre) +
    #   
    # (c[7]*words + c[8]*popular_artist + c[9]*current_trend*peak_last_year)*(x), col="orange", linewidth =1)+

  scale_y_continuous(breaks = seq(0, 2500000, by=500000), labels=c("0","500,000","1,000,000","1,500,000","2,000,000","2,500,000"))+
  # 
  # scale_x_continuous(breaks = c(1000000000, 2000000000, 3000000000, 4000000000), labels = c("1 B", "2 B", "3 B", "4 B"))+

  theme_classic() +

  labs(
    title = "Popularity vs Streams",
    subtitle = "Interesting segregation in popularity's expert rating",
    x= "Popularity ranking",
    y = "Daily streams per song") +

  theme(
    plot.title = element_text(size = 13, color = "grey30"),
    axis.text.x = element_text(color = "grey45"),
    axis.text.y = element_text(color = "grey45"),
    plot.subtitle = element_text(color = "grey35", size = 10),
    axis.title = element_text(color = "grey35", size = 10))
```

<img src="Spotify-Regression_files/figure-html/popularity segregation-1.png" style="display: block; margin: auto;" />

<br>

Above is what the pair of daily ~ popularity looks like when plotted. I spent a while considering why there was a big leap on popularity from 20 to around 60. I found that many songs were duplicated. The way they were duplicated is that they had two versions (example, November Rain (original), November Rain (Remastered)). It was the case that the secondary versions of those duplicated songs where in the lower popularity groups. And since the information I have is considering only the most popular songs and artists for the last 10 years, I shouldn't be considering duplicates nor unpopular music. 

So I filtered songs that are popular above 60, which is were a normal trend is on sight. I focused on the popular songs because it's I found all of their information without NAs, However in my main model, I consider all of them without filters.

After filtering the daily~streams plot looked like this. This is colored by popularity group, where popularity 1 are the most popular songs and 3 the lowest. This is considering that they are all popular songs overall compared to the whole industry. And on the previous tab you can see how it looks like with a couple representations of my model with the popularity groups segregated.


```r
asd <- read_csv("https://github.com/Tonadeleon/Regression-Projects/raw/main/Datasets/dataset.csv")|>
  filter(popularity > 59) |>
  rename(title = track_name,
         artist = artists,
         title_id = track_id,
         album = album_name,
         loud = loudness,
         words = speechiness,
         acoustics = acousticness,
         instrumental = instrumentalness,
         live = liveness) |>
  inner_join(df1, by="artist", relationship =
  "many-to-many") |>

  inner_join(df2, by="artist", relationship =
  "many-to-many")


asdf <- inner_join(asd, df, by = c("title", "artist"), relationship =
  "many-to-many") |>
  mutate(explicit = as.factor(explicit),
         genre = as.factor(track_genre),
         daily = daily,
         #streams = ifelse(streams>3825666400, 942193161, streams),
         #times = as.factor(time_signature),

         #energy = ifelse(energy>.5,1,0),
         #duration = ifelse(between(duration_ms, 200000, 250000), 1, 0),
         #loud = ifelse(loud>-9, 1, 0),
         dance = ifelse(danceability>.5,1,0),
         words = ifelse(between(words, .025, .06), 1, 0),
         instrumental = ifelse(instrumental>0,1,0),
         #popularity = ifelse(popularity>76,1,0),
         live = ifelse(live<.37,1,0),
         key = case_when(key %in% c(0, 1, 6, 8, 11) ~ 1,TRUE ~ 0),
         explicit = case_when(explicit == "TRUE" ~ 1,TRUE ~ 0),
         genre = case_when(genre %in% c("pop", "chill", "indie", "garage", "ambient", "folk", "indie-pop", "power-pop", "reggae", "soul", "synth-pop", "swedish", "country", "metal", "german", "funk", "alternative", "alt-rock") ~ 1,TRUE ~ 0),
         peak_last_year = ifelse(months_sincepk<13,1,0),
         y = ifelse(popularity >79 ,1 ,0),
         popularity0 = round(popularity / 5) * 5,
         popularity0 = case_when( popularity0 > 84 ~ 1, popularity == 80 ~ 2, TRUE ~ 3 ),
         popularity1 = round(popularity / 10) * 10,
         popularity1 = case_when( popularity1 == 0 ~ 1, popularity1 == 70 ~ 2, popularity1 == 80 ~ 3,  popularity1 == 90 ~ 4, TRUE ~ 5 ),
         times = case_when(time_signature %in% c(4) ~ 1,TRUE ~ 0),
         energy = round(energy*10),
         popular_artist = case_when(artist_streams > 75000 ~ 1, between(artist_streams, 50000,75000) ~ 2, between(artist_streams, 25000,50000) ~ 3, artist_streams < 25000 ~ 4, TRUE ~ 0 ),
         diff = peak_monthly_listeners - monthly_listeners,
         current_hit = ifelse(diff>20000000,1,0),
         current_trend = ifelse(daily_average<0,0,1)) |>
         #popularity = ifelse(popularity>76,1,0)) |>
  #distinct(artist, .keep_all = TRUE) |>
  distinct(title, .keep_all = TRUE) |>
  #filter(months_sincepk < 120) |>
  filter(!(row_number() %in% c(159))) |>
  filter(!(row_number() %in% c(108,297,324))) |>
  filter(!(row_number() %in% c(278))) |>
  filter(!(row_number() %in% c(245,215))) |>
  filter(!(row_number() %in% c(200))) |>
  filter(!(row_number() %in% c(208))) |>
  dplyr::select(daily1, streams, daily, popularity0, popularity1,  explicit, dance, energy, key, loud, mode, words, acoustics, instrumental, live, valence, tempo, times, genre, duration_ms, popular_artist, monthly_listeners, peak_monthly_listeners, months_sincepk, peak_last_year, daily_average, diff,current_hit,current_trend, artist_streams, popularity)

ggplot(asdf, aes((streams),(daily), col=as.factor(popularity0)))+

  geom_point(show.legend = T) +

  scale_y_continuous(breaks = seq(0, 2500000, by=500000), labels=c("0","500,000","1,000,000","1,500,000","2,000,000","2,500,000"))+
  # 
  scale_x_continuous(breaks = c(1000000000, 2000000000, 3000000000, 4000000000), labels = c("1 B", "2 B", "3 B", "4 B"))+

  theme_classic() +

  labs(
    title = "All time streams vs Daily streams",
    subtitle = "Popularity as grouping factor",
    x= "All time streams per song",
    y = "Daily streams",
    col = "Popularity groups") +

  theme(
    plot.title = element_text(size = 13, color = "grey15"),
    axis.text.x = element_text(color = "grey35"),
    axis.text.y = element_text(color = "grey35"),
    plot.subtitle = element_text(color = "grey25", size = 10),
    axis.title = element_text(color = "grey15", size = 10))
```

<img src="Spotify-Regression_files/figure-html/unnamed-chunk-4-1.png" style="display: block; margin: auto;" />


<br><br><br><br>
 
###### Variables Used
 
<br>

This analysis, even when having a lower R^2^ was a little more involved when considering the final variables that were going to play a role in it. In a summary these were the variables.

<br>

**Response:**

<br>

1. <span style="color: #0047AB;">**daily** - Max count of streams per song</span>

<br>

**Explanatory:**

<br>

2. <span style="color: #0047AB;">**streams** - Total artist streams in Spotify for March 2023. (Main quantitative variable)</span>

3. <span style="color: #0047AB;">**popularity0** - 1-3 categorical variable. Originally a categorical variable 0-100, but then filtered to avoid duplicates, and then rounded in three groups, thus finding more significance.</span>

4. <span style="color: #0047AB;">**words** - 1-0 categorical variable. Expert rating where 1 means the song has too many words, 0 means it has less words on average.</span>

5. <span style="color: #0047AB;">**months_sincepk** - quantitative variable that counts the number of months since the max daily streams per song</span>

6. <span style="color: #0047AB;">**peak_last_year** - 0-1 categorical variable derived from months_sincepk, where the peak happened in at least 12 months before March 2023. One means the artist is on at least one year of popularity, 0 means the artist's popularity is fading (1 year span)</span>

7. <span style="color: #0047AB;">**current_trend**- 0-1 categorical variable. Mutation from average streams, where the numbers where positive or negative, meaning that the artist is losing or getting more streams daily as compared to their career daily average. 1 = postive trend, 0 means negative trend.</span>

8. <span style="color: #0047AB;">**popular_artist** - 0-1 categorical variable. Mutation Where the artist is above the 75^th^ percentile of total streams in Spotify.</span>

9. <span style="color: #0047AB;">**popular_artist** - 0-1 categorical variable. Mutation Where the artist is above the 75^th^ percentile of total streams in Spotify.</span>

10. <span style="color: #0047AB;">**monthly_listeners** - Numerical variable with the amount of followers the artist has on spotify as of March 2023</span>

11. <span style="color: #0047AB;">**genre** - 0-1 categorical variable. 1 means the song's genre is within this group, and 0 is where it's a different less current popular genre by distributions. 1 = "pop", "chill", "indie", "garage", "ambient", "folk", "indie-pop", "power-pop", "reggae", "soul", "synth-pop", "swedish", "country", "metal", "german", "funk", "alternative", "alt-rock". </span>

<br>

When considering all this variables in the model, its significance got better little by little. This model wasn't making much sense to me, but I worked on it so I figured that I could share it. And disregarding that, good insights can be made from this, such as what genres are the most popular ones today, as well as the importance of the amounts of words in a song, "less words being prefered by the current consumers". Also, songs that peaked one year ago or more start to lose popularity rather fast, most songs now are short lived instead of being classic "legend songs", this can lead to consumers wanting more songs, and too many being created that they are lesser in quality throughout generations.
 
<br><br><br><br>
 
###### Summary

<br>


```r
pander::pander(summary(mylms))
```


-----------------------------------------------------------------------------
                  &nbsp;                    Estimate    Std. Error   t value 
------------------------------------------ ----------- ------------ ---------
             **(Intercept)**                  -2496       99828      -0.025  

             **popularity0**                 -121706      17347      -7.016  

          **monthly_listeners**             0.006491     0.001127     5.758  

            **months_sincepk**                2426        677.5       3.581  

     **popular_artist:current_trend**         12867        8617       1.493  

          **popularity0:genre**               46642       15876       2.938  

            **streams:words**               0.0001069   2.173e-05     4.92   

        **popular_artist:streams**          8.435e-05   8.219e-06     10.26  

 **current_trend:streams:peak_last_year**   -7.24e-05   3.678e-05    -1.969  
-----------------------------------------------------------------------------

Table: Table continues below

 
------------------------------------------------------
                  &nbsp;                    Pr(>|t|)  
------------------------------------------ -----------
             **(Intercept)**                 0.9801   

             **popularity0**                1.315e-11 

          **monthly_listeners**             1.96e-08  

            **months_sincepk**              0.0003948 

     **popular_artist:current_trend**        0.1364   

          **popularity0:genre**              0.00354  

            **streams:words**               1.373e-06 

        **popular_artist:streams**          1.294e-21 

 **current_trend:streams:peak_last_year**    0.04985  
------------------------------------------------------


--------------------------------------------------------------
 Observations   Residual Std. Error   $R^2$    Adjusted $R^2$ 
-------------- --------------------- -------- ----------------
     337              264686          0.6156       0.6062     
--------------------------------------------------------------

Table: Fitting linear model: ((daily)) ~ popularity0 + popular_artist:current_trend + popularity0:genre + monthly_listeners + months_sincepk + streams:words + streams:popular_artist + streams:current_trend:peak_last_year

<br>

This model's summary looks decent for not 100% correlated variables. R^2^ of .62, and all significant values can help explain 62% of the variance in this relationship between all time streams and daily streams.
 
<br><br><br><br>
 
###### Diagnostics
 
<br>
 

```r
par(mfrow=c(1,3))
plot(mylms,which=1:2)
plot(mylms$residuals)
```

<img src="Spotify-Regression_files/figure-html/alternative diagnostic-1.png" style="display: block; margin: auto;" />
 
<br>
 
Diagnostics look good, only problem to appear to be presence is a little of right skewness.
 
<br><br><br><br>
 
###### Validation
 
<br>


```r
set.seed(4)
 
num_rows <- 250 #330 total
keep <- sample(1:nrow(dtss), num_rows)
 
mytrain <- dtss[keep, ]
 
mylms <- lm(((daily)) ~

            #streams+ #x

             popularity0 +

             popular_artist:current_trend +

             popularity0:genre +

             monthly_listeners +

             months_sincepk +

             streams:words +

             streams:popular_artist +

             streams:current_trend:peak_last_year
, mytrain)
 
mytest <- dtss[-keep, ] #Use this in the predict(..., newdata=mytest)
 
  yhcc <- predict(mylms, newdata=mytest)
  ybarcc <- mean(mytest$daily) 
  SSTOcc <- sum( (mytest$daily - ybarcc)^2 )
  SSEcc <- sum( (mytest$daily - yhcc)^2 )
  rscc <- 1 - SSEcc/SSTOcc
  n <- length(mytest$daily) #sample size
  pcc  <- length(coef(mylms)) #num. parameters in model
  rscca <- 1 - (n-1)/(n-pcc)*SSEcc/SSTOcc
my_output_table2 <- data.frame(
  Model = c(
    "Monthly Followers Model"),
  `Original R2` = c(
    summary(mylms)$r.squared),
    #summary(bsalm)$r.squared), 
  `Orig. Adj. R-squared` = c(
    summary(mylms)$adj.r.squared),
    #summary(bsalm)$adj.r.squared,
 
  
  `Validation R-squared` = c(
    rscc),
  `Validation Adj. R^2` = c(
    rscca))
 
 
colnames(my_output_table2) <- c(
  "Model", "Original $R^2$", 
  "Original Adj. $R^2$", 
  "Validation $R^2$", 
  "Validation Adj. $R^2$")
 
knitr::kable(my_output_table2, escape=TRUE, digits=5)
```



|Model                   | Original $R^2$| Original Adj. $R^2$| Validation $R^2$| Validation Adj. $R^2$|
|:-----------------------|--------------:|-------------------:|----------------:|---------------------:|
|Monthly Followers Model |        0.59437|              0.5809|           0.6438|               0.60727|

<br>

Validations look decent in this model. Only problem I see is that the validation results have a lot of variance. Whereas in my actual model they're always close to .93 R^2^
 
<br><br><br><br>
 
###
 
####
 
## Model Summary

<br>
 

```r
pander::pander(summary(mylm))
```


----------------------------------------------------------------------
        &nbsp;           Estimate   Std. Error   t value    Pr(>|t|)  
----------------------- ---------- ------------ --------- ------------
    **(Intercept)**       -14.64      0.5426     -26.98    2.214e-116 

     **explicit**        0.09548     0.02827      3.377    0.0007659  

 **monthly_listeners**    0.9529      0.0321      29.69    1.511e-133 

  **artist_streams**      0.4228     0.02244      18.84    1.879e-66  

      **streams**        -0.1919     0.02495     -7.692    3.988e-14  
----------------------------------------------------------------------


--------------------------------------------------------------
 Observations   Residual Std. Error   $R^2$    Adjusted $R^2$ 
-------------- --------------------- -------- ----------------
     857               0.305          0.9341       0.9338     
--------------------------------------------------------------

Table: Fitting linear model: (daily1) ~ explicit + (monthly_listeners) + (artist_streams) + (streams)

<br>

After carefully considering the slope and intercept coefficients, and many tries of adding correlated variables to get a better fit, I ended up using the model above to try to predict daily streams per artist.

Multiple variables were used, and all of them are significant to the model. All the variables supported a log transformation, and these transformations brought the model pretty high up in significance.

This model abilitates the user to test the assumption of streams increase per follower gained, as well as test different scenarios and aim for goal setting, and insight gathering. (See interpretation)

<br><br><br><br>
 
## Diagnostics
 
<br>
 

```r
par(mfrow=c(1,3))
plot(mylm,which=1:2)
plot(mylm$residuals)
```

<img src="Spotify-Regression_files/figure-html/original diagnostic-1.png" style="display: block; margin: auto;" />

<br>
 
All diagnostics look good. Normality may appear to be right skewed on the residuals; another curious thing is that the residual vs fitted plot shows some linear patterns, it is uncertain if this is is a concern for the model, the reason behind it maybe that some artists have a couple different songs in this set; and being that there's a large sample size of almost a thousand unique songs an artists, we will proceed with the analysis. 
 
<br><br><br><br>
 
## Interpretation

### {.tabset .tabset-fade .tabset-panel}

<br>
 
Even when this is a High Dimensional model, it can still be interpreted in a scenario by scenario basis. Slopes may vary according to each scenario. They help predict with a high accuracy the amount of streams an artist can have for a given scenario. 

This tool can be useful to set goals (if you were the artist, how many followers would you need to meet certain goal on streams); Or for curious people that want to estimate streams in the industry, as well as competitors wanting to estimate others' streams and wit this, estimate their monthly earnings too.

Here are a couple examples on how to interpret this model, and how to think about it.

<br>

#### Predictions

<br>


```r
average_monthly_followers <- mean(dt$monthly_listeners)
mean_artist_streams <- mean(dt$artist_streams)
mean_streams <- mean(dt$streams)
mean_daily <- mean(dt$daily)

new_data <- data.frame(monthly_listeners = average_monthly_followers,
                       artist_streams = mean_artist_streams,
                       streams = mean_streams,
                       daily = mean_daily,
                       explicit = 0,
                       popular_artist = 4)

prediction <- predict(mylm, newdata = new_data, interval = "prediction")


UCL <- (prediction[, 3])
LCL <- (prediction[, 2])
Prediction <- (prediction[, 1])


x <- log(50000000)
mean_artist_streams <- mean(dt$artist_streams)
mean_streams <- mean(dt$streams)
mean_daily <- mean(dt$daily)

new_data1 <- data.frame(monthly_listeners = x,
                       artist_streams = mean_artist_streams,
                       streams = mean_streams,
                       daily = mean_daily,
                       explicit = 0,
                       popular_artist = 4)
Prediction2 <- predict(mylm, newdata = new_data1, interval = "prediction")

UCL2 <- (Prediction2[, 3])
LCL2 <- (Prediction2[, 2])
Prediction2 <- (Prediction2[, 1])


ggplot(dt, aes(y = daily1, x = monthly_listeners)) +
  
  geom_point(show.legend = FALSE, col = "orange") +
  
  geom_vline(xintercept = x, linetype = "dotted") +
  
  geom_vline(xintercept = average_monthly_followers, linetype = "dotted") +
  
  stat_function(fun=function(x)

      (b[1] + b[2]*0 + b[4]*9.030 + b[5]*20.55 + b[6]*12.94) +

      (b[3])*x, linewidth =1, col="black")+
  
  geom_segment(aes(x = x, y = LCL2, xend = x, yend = UCL2), linetype = "solid", color = "steelblue", linewidth=1) +
  
  geom_point(aes(x = x, y = Prediction2), color = "red", size = 3) +
  
  geom_segment(aes(x = average_monthly_followers, y = LCL, xend = average_monthly_followers, yend = UCL), linetype = "solid", color = "skyblue", linewidth=1) +
  
  geom_point(aes(x = average_monthly_followers, y = Prediction), color = "red", size = 3) +
  
  geom_label(aes(x = x, y = Prediction2, label = "Prediction 2"), 
             color = "grey15", fill = "white", hjust = -0.1, vjust = -0.5) +
  
  geom_label(aes(x = average_monthly_followers, y = Prediction, label = "Prediction 1"), 
             color = "grey15", fill = "white", hjust = -0.1, vjust = -0.5) +
  
  scale_x_continuous(breaks = log(c(4500000, 10000000, 25000000, 50000000, 100000000)), labels = c("4.5 M", "10 M", "25 M", "50 M", "100 M")) +
  scale_y_continuous(breaks = c(log(.5), log(1.5), log(6), log(20), log(70)), labels=c(".5 M","1.5 M","6 M","20 M" ,"70 M"))+
  theme_classic() +
  labs(title = "Artist Monthly Followers vs. Artist Daily Streams",
       subtitle = "Pred1: All variable mean values in play \nPred 2: All variable mean values in play, but 50 M followers instead of the average 24 M",
       x = "Artist Monthly Followers",
       y = "Artist Daily Streams",
       caption = "Log Scale") +
  theme(
    plot.title = element_text(size = 14, color = "grey15"),
    axis.text.x = element_text(color = "grey35", size=10),
    axis.text.y = element_text(color = "grey35", size=10),
    plot.subtitle = element_text(color = "grey25", size = 11),
    axis.title = element_text(color = "grey15", size = 12))
```

<img src="Spotify-Regression_files/figure-html/pred 1 graph-1.png" style="display: block; margin: auto;" />

<br>


```r
prediction_df <- data.frame(
  "Name" = c("Prediction 1", "Prediction 2"),
  Predicted = round((exp(c(Prediction, Prediction2))),1), 
  LCL1 = round((exp(c(LCL, LCL2))),1), 
  UCL1 = round((exp(c(UCL, UCL2))),1)
)

# Print the table with kable and kable_styling
prediction_df %>%
  kable(format = "markdown", align = "c") %>%
  kable_styling(full_width = FALSE)
```

<table class="table" style="color: black; width: auto !important; margin-left: auto; margin-right: auto;">
 <thead>
  <tr>
   <th style="text-align:center;"> Name </th>
   <th style="text-align:center;"> Predicted </th>
   <th style="text-align:center;"> LCL1 </th>
   <th style="text-align:center;"> UCL1 </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:center;"> Prediction 1 </td>
   <td style="text-align:center;"> 4.6 </td>
   <td style="text-align:center;"> 2.5 </td>
   <td style="text-align:center;"> 8.3 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> Prediction 2 </td>
   <td style="text-align:center;"> 8.6 </td>
   <td style="text-align:center;"> 4.7 </td>
   <td style="text-align:center;"> 15.7 </td>
  </tr>
</tbody>
</table>

<br>

Recall to the first plot in this analysis, where the mean scenario (middle line in Introduction tab), as well as the 1st and 3rd quartiles scenarios where shown as examples. When the all average scenario is put in play all, predictions made based on that assumption will follow the same slope. Remember many scenarios can be predicted depending on variables like, the artist has curse words in their music or not, what is the max amount of streams the artist has had so far in a single month, what is the artist most popular song and its max streams count, and of course we need to predict for a certain a amount of followers to get an accurate daily streams prediction.

In the plot above, the bigger dots represent two stream predictions, and the colored lines represent their respective 95% prediction intervals.

For the light blue prediction, all the average values for each variable were used to predict the daily streams of an artist; such artist has Spotify's industry mean amount of followers as well as the mean in all the other mentioned variables. Keep in mind that this is not all Spotify's mean, rather recall that we are analyzing top 1000 songs and their artists.

The results are in the table above where the numbers are rounded to 1 decimal for simplicity, and they mean million streams.

If I predict for rather 50 Million followers (dark blue Prediction) instead of the average. The prediction follows the same trend. Be assured that when choosing your specifics for each variable, the predictions will follow the same line even when the x values are changed.

We can say that the slope of this line is of **2.5931304**, this means that for every follower gained in a month, we can expect around 2 additional streams. And, to be 95% certain that this prediction is accurate, the prediction interval is added. This means that we are 95% sure that for every new follower, the artist will have between 5 and 8 extra streams.
 
<br><br><br><br>

#### Confidence Interval

<br>


```r
get_prediction_intervals <- function(model, data) {

  prediction_intervals <- data.frame(monthly_listeners = numeric(),
                                     lower_bound = numeric(),
                                     upper_bound = numeric(),
                                     stringsAsFactors = FALSE)

  for (x_val in data$monthly_listeners) {

    new_data <- data.frame(monthly_listeners = (x_val),
                           artist_streams = (mean(data$artist_streams)),
                           streams = (mean(data$streams)),
                           daily = (mean(data$daily)),
                           explicit = 0,
                           popular_artist = 4)
    

    predictions <- predict(model, newdata = new_data, interval = "confidence")
    
    prediction_intervals <- rbind(prediction_intervals, 
                                  data.frame(monthly_listeners = (x_val),
                                             lower_bound = predictions[, "lwr"],
                                             upper_bound = predictions[, "upr"],
                                             stringsAsFactors = FALSE))
  }
  
  return(prediction_intervals)
}



prediction_intervals <- get_prediction_intervals(mylm, dt) |> 
  mutate(monthly_listeners = (monthly_listeners),
         up = (upper_bound),
         low = (lower_bound))

dt123 <- inner_join(dt,prediction_intervals,by="monthly_listeners")

ggplot(dt123, aes(x = monthly_listeners, y = daily1)) +
  geom_point(color = "orange") +
  stat_function(fun=function(x)

      (b[1] + b[2]*0 + b[4]*9.030 + b[5]*20.55 + b[6]*12.94) +

      (b[3])*x, linewidth =1, col="steelblue4")+
  
  geom_ribbon(aes(ymin = low, ymax = up), fill = "skyblue1", alpha = 0.3) +
  
  scale_x_continuous(breaks = c(log(4500000), log(10000000), log(25000000), log(50000000), log(100000000)),
                     labels = c("4.5 M", "10 M", "25 M", "50 M", "100 M")) +
  
  scale_y_continuous(breaks = c(log(0.5), log(1.5), log(6), log(20), log(70)),
                     labels = c("0.5 M", "1.5 M", "6 M", "20 M", "70 M")) +
  
  theme_classic() +
  labs(
    title = "Artist Monthly Followers vs. Artist Daily Streams",
    subtitle = "The light blue ribbon represents the confidence interval for this specific scenario \n Scenario when all the mean values for all variables are present",
    x = "Artist Monthly Followers",
    y = "Artist Daily Streams",
    caption = "Log scale") +
  theme(
    plot.title = element_text(size = 14, color = "grey15"),
    axis.text.x = element_text(color = "grey35", size=10),
    axis.text.y = element_text(color = "grey35", size=10),
    plot.subtitle = element_text(color = "grey25", size = 11),
    axis.title = element_text(color = "grey15", size = 12))
```

<img src="Spotify-Regression_files/figure-html/unnamed-chunk-6-1.png" style="display: block; margin: auto;" />

<br>

Consider this graph to be a 2D representation of a higher dimensional model's confidence interval. You can see a ribbon representing a confidence interval for this situation. This confidence interval takes place when the mean of all other variables is used to get predictions; In reality, there is not a "one single confidence interval fits all lines" situation, but a confidence interval for each line; The lines then would go in the middle of each of those confidence intervals. However since the amount of lines is near to infinite, it suffices to show one for the purpose of this analysis, still we should remember this is not the only confidence interval in existence.

This is an understandable representation because if this graphing tool were to graph every ribbon of every line (If it had the capacity to do so all at once), the plot would then be filled with infinite ribbons for all the lines that this model could portray; and almost all the area where the dots are (and even possibly the whole plot) would be filled with confidence interval ribbons. 

This would also be a good thing, it means that most data points can be explained by this model.
 
<br><br><br><br>

#### Total Artist Streams

<br>


```r
ggplot(dt, aes(y=(daily1), x=(artist_streams)))+
  geom_point(show.legend = F, col="firebrick")+
  
  scale_x_continuous(breaks = c(log(1000), log(3000), log(8000), log(22000), log(60000)), labels=c("1 B", "3 B", "8 B","22 B","60 B")) +
   
  scale_y_continuous(breaks = c(log(.5), log(1.5), log(6), log(20), log(70)), labels=c(".5 M","1.5 M","6 M","20 M" ,"70 M"))+
 
  theme_classic() +
  labs( 
    title = "Artist Monthly Followers vs. Artist Daily Streams", 
    subtitle = "Updated as of March 2023 | Spotify Measurements",
    x= "Total artist streams (Spotify)",
    y = "Artist Daily Streams") +
  theme(
    plot.title = element_text(size = 14, color = "grey15"),
    axis.text.x = element_text(color = "grey35", size=10),
    axis.text.y = element_text(color = "grey35", size=10),
    plot.subtitle = element_text(color = "grey25", size = 11),
    axis.title = element_text(color = "grey15", size = 12))
```

<img src="Spotify-Regression_files/figure-html/unnamed-chunk-7-1.png" style="display: block; margin: auto;" />

<br>

As seen here, total artist streams (All time) is also a good predictor of daily streams; however, since these are the all time streams of an artist it's not as good as using the artist followers to predict streams. Interestingly enough, this variable also supported a double log transformation, and is included in the model as an explanatory variable.

<br><br><br><br>

###
 
## Validation

<br>
 

```r
set.seed(17)
 
num_rows <- 440 #882 total
keep <- sample(1:nrow(dt), num_rows)
 
mytrain <- dt[keep, ]
 
mylm <- lm((daily1) ~ 
             
             explicit +
             
             ###
             
             (monthly_listeners) + 
             
             (artist_streams) + 
             
             (streams)
, mytrain)
 
mytest <- dt[-keep, ] #Use this in the predict(..., newdata=mytest)
 
  yhcc <- predict(mylm, newdata=mytest)
  ybarcc <- mean(mytest$daily1) 
  SSTOcc <- sum( (mytest$daily1 - ybarcc)^2 )
  SSEcc <- sum( (mytest$daily1 - yhcc)^2 )
  rscc <- 1 - SSEcc/SSTOcc
  n <- length(mytest$daily1) #sample size
  pcc  <- length(coef(mylm)) #num. parameters in model
  rscca <- 1 - (n-1)/(n-pcc)*SSEcc/SSTOcc
my_output_table2 <- data.frame(
  Model = c(
    "Alternative Model"),
  `Original R2` = c(
    summary(mylm)$r.squared),
    #summary(bsalm)$r.squared), 
  `Orig. Adj. R-squared` = c(
    summary(mylm)$adj.r.squared),
    #summary(bsalm)$adj.r.squared,
 
  
  `Validation R-squared` = c(
    rscc),
  `Validation Adj. R^2` = c(
    rscca))
 
 
colnames(my_output_table2) <- c(
  "Model", "Original $R^2$", 
  "Original Adj. $R^2$", 
  "Validation $R^2$", 
  "Validation Adj. $R^2$")
 
knitr::kable(my_output_table2, escape=TRUE, digits=5)
```



|Model             | Original $R^2$| Original Adj. $R^2$| Validation $R^2$| Validation Adj. $R^2$|
|:-----------------|--------------:|-------------------:|----------------:|---------------------:|
|Alternative Model |        0.93058|             0.92994|          0.93749|               0.93688|

<br>

Validations are satisfying. Even when using a sample size less than half of the original, the R^2^ coefficient remained true to the model. This assures that the model is good for describing the estimation of daily artists' streams in Spotify. We could use the streaming results of this model to estimate how much an artist makes if we wanted to. The use cases of this model are many, and can be proven useful for artists when planning ahead and focusing efforts on some specific marketing task.

<br><br><br><br>

<!-- add datasets -->
