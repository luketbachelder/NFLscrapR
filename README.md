# NFLscrapR
NFL data analysis, with a focus on Dallas Cowboys production and efficiency
NFL Data Analyst Ben Baldwin at the Athletic created a very useful startup guide into nflscrapR, which is a dataset that contains scraped play by play data from ESPN. He provided the code to get the correct libraries and packages from R and the .csv data. The code to my projects are below:
install.packages("tidyverse") 
install.packages("dplyr")
install.packages("na.tools")
install.packages("ggimage") the initial packages that need to be loaded

library(tidyverse)
library(dplyr)
library(na.tools)
library(ggimage) now we need to load the libraries into the R session. These are for the graphs and charts, not the actual data itself

pbp_2019 <- read_csv(url("https://github.com/ryurko/nflscrapR-data/raw/master/play_by_play_data/regular_season/reg_pbp_2019.csv"))
this is where the data is.

pbp_2019 %>% select(posteam, defteam, desc, play_type) %>% head
(if this is ran, we can see the first 5 rows of data (head = top 5) from the first NFL game. This was CHI vs. GB in 2019. It was a very boring game)

pbp_rp_2019 <- pbp_2019 %>% 
	filter(!is_na(epa), play_type=="no_play" | play_type=="pass" | play_type=="run")
(This is taking into account no-plays, which are penalties).

pbp_rp_2019 <- pbp_rp_2019 %>%
	mutate(
	pass = if_else(str_detect(desc, "( pass)|(sacked)|(scramble)"), 1, 0),     (creating a variable for pass plays and run plays)
	rush = if_else(str_detect(desc, "(left end)|(left tackle)|(left guard)|(up the middle)|(right guard)|(right tackle)|(right end)") & pass == 0, 1, 0),
	success = ifelse(epa>0, 1 , 0)
	) 
pbp_rp_2019 <- pbp_rp_2019 %>% filter(pass==1 | rush==1)
(committing the filter on the run/pass play variable)

(This next piece of code was intended to scrape the play-by-play data from 2009-2019, which is all the data that exists in nflscrapR. The difficulty I had was when I tried to bind these datasets together to create a datalist called "pbp_all" This would be used to create a bigger and loger story in relation to Dallas Cowboys success. 

first <- 2009 #first season to grab. min available=2009
last <- 2018 # most recent season
datalist = list()
for (yr in first:last) {
pbp <- read_csv(url(paste0("https://github.com/ryurko/nflscrapR-data/raw/master/play_by_play_data/regular_season/reg_pbp_", yr, ".csv")))
games <- read_csv(url(paste0("https://raw.githubusercontent.com/ryurko/nflscrapR-data/master/games_data/regular_season/reg_games_", yr, ".csv")))
pbp <- pbp %>% inner_join(games %>% distinct(game_id, week, season)) %>%
datalist[[yr]] <- pbp # add it to your list
}
pbp_all <- dplyr::bind_rows(datalist)

(It didn't work, becuase the column blocked player ID had non-numerical values.)
(I tried a number of things, ranging from converting that column to be all null, so it would pass on the bind to trying to stop the validation itself from being thrown due to logical/character misatches. Eventually, I was able to just drop that column from the individual datasets and then bind, but this was last resort, since it's just dumping data that could be useful. Fortunately, this column was not important for my scope.)

first <- 2009 #first season to grab. min available=2009
last <- 2019 # most recent season
datalist = list()
for (yr in first:last) {
pbp <- read_csv(url(paste0("https://github.com/ryurko/nflscrapR-data/raw/master/play_by_play_data/regular_season/reg_pbp_", yr, ".csv")))
games <- read_csv(url(paste0("https://raw.githubusercontent.com/ryurko/nflscrapR-data/master/games_data/regular_season/reg_games_", yr, ".csv")))
pbp <- pbp %>% inner_join(games %>% distinct(game_id, week, season)) %>% select (-fumble_recovery_2_yards, -blocked_player_id, -fumble_recovery_2_player_id)
datalist[[yr]] <- pbp # add it to your list
}
pbp_all <- dplyr::bind_rows(datalist)

(I ended up having to deselect 3 columns from being checked prior to the join. The non-logical values were a pain.)
select (-fumble_recovery_2_yards, -blocked_player_id, -fumble_recovery_2_player_id)

(Next, I created some cool graphs. Here is one of them that shows success rate and EPA per play for all of the 2019 season.)

chart_data <- pbp_rp %>% (creating a new dataframe called chart_data from existing df pbp_rp)
	filter(pass==1, ) %>% (We only care about pass plays)
	group_by(posteam) %>% (by team, the column is posteam in the datasets)
	summarise(
	num_db = n(), (number of dropbacks = n)
	epa_per_db = sum(epa) / num_db, (EPA/dropback = total EPA/n)
	success_rate = sum(epa > 0) / num_db (if the play is successfull, it must be greater than 0)
	)

nfl_logos_df <- read_csv("https://raw.githubusercontent.com/statsbylopez/BlogPosts/master/nfl_teamlogos.csv") (cool little data scrape)
chart <- chart_data %>% left_join(nfl_logos_df, by = c("posteam" = "team_code")) 

chart %>% 
ggplot(aes(x = success_rate, y = epa_per_db)) + (this is setting the graph)
	geom_image(aes(image = url), size = 0.05) + (size params)
	labs(x = "Success rate",
	y = "EPA per play",
	caption = "Data from nflscrapR",
	title = "Dropback success rate & EPA/play",
	subtitle = "2019") +
	theme_bw() +
	theme(axis.title = element_text(size = 12),
	axis.text = element_text(size = 10),
	plot.title = element_text(size = 16),
	plot.subtitle = element_text(size = 14),
        plot.caption = element_text(size = 12))

ggsave('Success Rate & EPA/play_2019.png', dpi=1000) (ggplot can save a .png file through Rstudio.)
