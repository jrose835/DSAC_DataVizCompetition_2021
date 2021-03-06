## DSAC Data Viz Competition 2021


The following is the work I did for the 2021 Emory DSAC data visualization competition. If you're interested in a little holiday cheer with data check it out below!

#### Task: Create a plot using the folowing datasets... 

1. Christmas songs in the Billboard top 100 list during December from 1958 to 2017 (christmas_billboard_data.csv)

2. Weather in Chicago on Christmas day from 1871 to 2018 (ChicagoWeatherChristmas.csv)

3. The gifts and the quantity of gifts acquired each day in "12 days of Christmas" (12_Days_of_Christmas.csv)


#### Package Setup:

`library(tidyverse)`

`library(janitor)`

`library(lubridate)`

`library(forcats)`

`library(gganimate)`


Here are the first two datasets:

`weather <- read_csv(file="datasets/ChicagoWeatherChristmas.csv") %>% clean_names()`

`songs <- read_csv("datasets/christmas_billboard_data.csv") %>% clean_names()`

## First let's take a look at the weather

There's a lot of data here! It should be easier to visualize this data by decade instead of each individual year I think. 

After that I want to do things like: 
 
 * total up the christmas snow per decade
 
 * total up the number of white christmas's per decade
 
 * make an apporpirately themed ggplot!

```
weather_defined <- subset(weather, white_christmas!="Not Defined") %>%
  mutate(decade= year-(year %% 10))
weather_defined$white_christmas <- as.logical(weather_defined$white_christmas)
white_xmas <- weather_defined %>% group_by(decade) %>% 
  summarize(n = n(), 
            white_xmases=sum(white_christmas), 
            total_snow=sum(snow), 
            total_precip = sum(precipitation),
            percent_white =white_xmases/(white_xmases+n),
            ave_snow = mean(snow),
            sd_snow = sd(snow)
                            )
white_xmas
```

Great! Data seems to be tidied up exactly how I want it. 

### Now let's plot it in a festive way!

```
ggplot(white_xmas, aes(x=decade, y=total_snow)) + geom_line() + geom_point(aes(size=n), color="white") + theme_minimal() + scale_x_continuous(breaks=unique(white_xmas$decade)) + labs(x="Decade", y="Total Chirstmas Snowfall (inches)", size="# White Christmases") +
ggtitle("It Still Snows on Christmas in Chicago", subtitle = "...just not as much as it used to") +
  theme(plot.background = element_rect(fill="lightsteelblue3"),
        panel.grid.minor = element_blank(),
        panel.grid.major.x = element_blank(),
        panel.grid.major.y = element_line(linetype="dashed"),
        text = element_text(colour="white"),
        axis.text = element_text(colour="white")
        )
ggsave("StillSnows.jpg", height=5, width=8)
```

![A line plot of total snow (inches) on Christmas in Chicago by decade](https://github.com/jrose835/DSAC_DataVizCompetition_2021/blob/master/StillSnows.jpg?raw=true)


#### Some insights:

* The number of white christmas's in Chichago has not really changed from decade to decade.

* HOWEVER the amount total snow on christmas day seems to be consistently less over the last 4 decades. Something to do with global warming and delayed winter weather perhaps?

## Now on to the holiday songs!

First I need to code the song variable as a factor so I can group and plot effectively with it

`songs$song <- factor(songs$song)`

### How long till Christmas?

One interesting metric is how far away from the actual date of Christmas each of these weeks are. Let's normalize the time component to 12/25 and see what it looks like.

```
songs$week_date <- mdy(songs$weekid)
#Set all years to 2000 or 2001 if the month is January
for (i in 1:length(songs$week_date)){
  if (month(songs$week_date[i])==1){
    year(songs$week_date[i]) <- 2001 
  }
  else {
    year(songs$week_date[i]) <- 2000
  }
}
xmas <- mdy("12-25-2000")
songs <- mutate(songs, xmas_dist = as.numeric(difftime(week_date, xmas, units="days")))
summary(songs$xmas_dist)
```

| Min.    | 1st Qu. | Median | Mean  | 3rd Qu. | Max.   |
|---------|---------|--------|-------|---------|--------|
| -50.000 | -7.500  | 3.000  | 2.364 | 12.000  | 37.000 |

Now summarize/facet by decade again to decluter the plot

```
songs <- mutate(songs, decade= year-(year %% 10))
songs$decade <- factor(songs$decade, ordered = T)
summary(songs$decade)
ggplot(songs, aes(x=xmas_dist, y=week_position, group=song)) + geom_line(aes(color=decade)) + scale_y_reverse() + facet_wrap(vars(decade))
```

| 1950 | 1960 | 1970 | 1980 | 1990 | 200 | 2010 |
|------|------|------|------|------|-----|------|
| 21   | 144  | 44   | 35   | 24   | 50  | 69   |

![Faceted line plot of chart trajectories with time from Christmas day on the x axis and billboard chart position on the y axis](https://github.com/jrose835/DSAC_DataVizCompetition_2021/blob/master/trajectories.png?raw=true)

Interesting in itself that there are so many more in the 1960s than any other decade.

Also, the songs seem to follow a parabolic arc here,possibly because we selected for ONLY holiday songs in this dataset. Most gradually start rising in position on the charts until you get a week or so past Christmas. Then many of them have a rather sharp drop off!

### What are the greatest Holiday hits?

Below I am doing a few things. First, I want to calculate a single score to measure the "hit potential" of each song over these holiday months. I call this the "hit score", and it is calculated by taking (100/peak_position) * weeks on the charts. 

**The higher the hit score, the higher and longer a particular song was on the charts for.** 

```
hit_score <- songs %>% group_by(performer, songid) %>% 
  summarise(peak_position=mean(peak_position), 
            weeks=mean(weeks_on_chart), 
            hit_score=(100/peak_position)*weeks) %>% 
  arrange(desc(hit_score)) %>% 
  ungroup()
songs2 <- left_join(songs, select(hit_score, songid, hit_score), by="songid") 
#Need to fix the one repeat with EXACT same hit score
songs2[songs2$songid=="BelieveBrooks & Dunn",]$hit_score <- 34
```

Now I want to rank each of the songs by hit_score based on the year and ALL previous years. This way we can see which decades generated the most chart-topping songs, and how this changed over the years.

I'm also going to rank the songs by hit score in each year to see what people were caroling to the most over the years.

```
songs_list <- list()
years <- unique(songs2$year)
for (i in 1:length(years)) {
  songs_list[[i]] <- subset(songs2, year <=years[i]) %>%
    ungroup() %>%
    distinct(songid, .keep_all=T) %>%
    #group_by(songid) %>%
    mutate(rank=rank(-hit_score),
           rank_year = years[[i]]) %>%
    subset(rank<25) %>%
    select(songid, performer, song, hit_score, year,decade,rank, rank_year)
}
songs_big_df <- Reduce(
  function(x, y, ...) merge(x, y, all = TRUE, ...),
  songs_list
)
songs_big_df <- mutate(songs_big_df, 
                       label = paste(song, performer, year, sep=" "))
```

And finally now I will animate it all over time and beautify it!

```
color_pal <- c("1950"="#3C4930", "1960"="#8AAEE2", "1970"="#A6001D",
               "1980"="#D00016","1990"="#D7BA5C",
               "2000"="#BF5E73","2010"="#AF8952")
animation <- ggplot(songs_big_df, aes(rank, group = songid,
                          fill = as.factor(decade), color = as.factor(decade))) +
  geom_tile(aes(y = hit_score/2,
                height = hit_score,
                width = 0.9), alpha = 0.8, color = NA) +
  geom_text(aes(y = 0, label = paste(label, " ")), vjust = 0.1, hjust = -0.2, color="black") +
  coord_flip(clip = "off", expand = FALSE) +
  scale_x_reverse() +
  scale_fill_manual(values=color_pal) +
  guides(color = FALSE, fill = FALSE) +
  labs(y="Hit Score (peak position * weeks on chart)") +
  theme(axis.line=element_blank(),
        #text = element_text(family = "Varela Round"),
        #axis.text.x=element_blank(),
        axis.text.y=element_blank(),
        axis.ticks=element_blank(),
        #axis.title.x=element_blank(),
        axis.title.y=element_blank(),
        legend.position="none",
        panel.background=element_blank(),
        panel.border=element_blank(),
        panel.grid.major=element_blank(),
        panel.grid.minor=element_blank(),
        panel.grid.major.x = element_line( size=.1, color="grey" ),
        panel.grid.minor.x = element_line( size=.1, color="grey" ),
        plot.title=element_text(size=25, 
                                #hjust=0.5, 
                                face="bold", colour="firebrick",
                                #vjust=-1
                                ),
        plot.subtitle=element_text(size=18, 
                                   #hjust=0.5, 
                                   face="italic", 
                                   color="darkgreen"),
        plot.caption =element_text(size=8, 
                                   #hjust=0.5, 
                                   face="italic", color="darkgrey"),
        plot.background=element_rect(fill="#FFEDE1"
                                       #"#DDE4F8"
                                     ),
        #plot.margin = margin(2,2, 2, 4, "cm")
        ) +
  transition_states(rank_year, transition_length = 4, state_length = 1) +
  enter_fade() + 
  exit_fade() +
  labs(title = 'Top 25 Holiday Hits Over the Years',
    subtitle = 'Year: {closest_state}',
    caption = "Data: Hot 100 singles chart from Billboard.com")
```

```
animate(animation, duration=30,fps=20, width=450, height=600, 
        renderer=gifski_renderer("holidayhits.gif"))
```

![Animated bar plot showing top holiday song hits and their hit score over the years](https://github.com/jrose835/DSAC_DataVizCompetition_2021/blob/master/holidayhits.gif?raw=true)

## Now let's bring it together

I'll use my "Days to Christmas" variable to make some plots of the absolute greatest Holiday hits of all time (according to hit score) to see how their trajectory went!

```
top_song <- subset(songs, songid=="This One's For The ChildrenNew Kids On The Block"|
                     songid =="MistletoeJustin Bieber" |
                     songid == "AmenThe Impressions" |
                     songid == "All I Want For Christmas Is YouMariah Carey" |
                     songid =="Same Old Lang SyneDan Fogelberg") %>%
            mutate(label = paste(song, performer, sep=" "))
top_song <- top_song %>% group_by(song)
trajectory <- ggplot(top_song, aes(x=xmas_dist, y=week_position, group=song)) + geom_line(aes(color=label)) + scale_y_reverse() + theme_minimal() + 
  labs(x="Days to Christmas", y = "Chart Position") +
  transition_reveal(xmas_dist, )
animate(trajectory, duration=30,fps=20, width=700, height=600, 
        renderer=gifski_renderer("tophitstrajectory.gif"))
```


![An animated line plot showing the top 5 songs by hit score and their trajectory. Time from Christmas day is on the x axis and billboard chart position on the y axis](https://github.com/jrose835/DSAC_DataVizCompetition_2021/blob/master/tophitstrajectory.gif?raw=true)

#### A few things:

* Bieber clearly jumped the shark and released his "MISTLETOE" a month too early, but we all forgave him and started jamming to it right at the holiday anyway

* Alternatively, Mariah knew that the absolute best time to release "ALL I WANT FOR CHRISTMAS" was the week before the big day! This one didn't need to go through the typical slow rise to fame, but it did have to deal with a lot of variablity week to week for some wierd reason. 

* Same Old Lang Syne doesn't seem to have the typical drop off that most other holiday songs have. Is this an elusive New Years jam??

### Summary

This project was fun! I learned a few things along the way...namely:

* Working with lubridate and date intervals

* How to animate ggplots using gganimate

* How to floor dates to decades


Extra bonus: I got to listen to a whole bunch of new holiday songs while working that I had never heard of before!

#### Until next time!




