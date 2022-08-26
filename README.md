
# RStudio Connect Usage Data

This repository illustrates several examples for getting started with
the RStudio Connect usage data. The examples:

-   [./examples/last_30_days](./examples/last_30_days)
-   [./examples/interactive_app](./examples/interactive_app)
-   [./examples/realtime](./examples/realtime)
-   [./examples/connectAnalytics](./examples/connectAnalytics)
-   [./examples/connectViz](./examples/connectViz)

The examples are generated using the [RStudio Connect Server
API](https://docs.rstudio.com/connect/api). The API and data collection
are both available as of RStudio Connect 1.7.0. The API contains data to
help answer questions like:

-   What content is most visited?
-   Who is visiting my content?
-   What reports are most common?
-   Has viewership increased over time?
-   Did my CEO actually visit this app?

**A data science team’s time is precious, this data will help you focus
and justify your efforts.**

## Basic example

The following code should work as-is if copied into your R session. NOTE
that it uses the [`connectapi`](https://github.com/rstudio/connectapi)
package, which can be installed from GitHub with
`remotes::install_github("rstudio/connectapi")`. You just need to
replace the server name, and have your API key ready.

``` r
## ACTION REQUIRED: Change the server URL below to your server's URL
Sys.setenv("CONNECT_SERVER"  = "https://connect.example.com/rsc") 
## ACTION REQUIRED: Make sure to have your API key ready
Sys.setenv("CONNECT_API_KEY" = rstudioapi::askForPassword("Enter Connect Token:")) 
```

This will use the `get_usage_shiny()` function to pull the latest
activity of the Shiny apps you are allowed to see within your server.

``` r
library(ggplot2)
library(dplyr)
library(connectapi)

client <- connect()

# Get and clean the Shiny usage data
shiny_rsc <- get_usage_shiny(
  client,
  from = lubridate::today() - lubridate::ddays(7), 
  limit = Inf
  ) %>%
  filter(!is.na(ended)) %>%
  mutate(session_duration = ended - started)

glimpse(shiny_rsc)
```

    ## Rows: 336
    ## Columns: 6
    ## $ content_guid     <chr> "dad2ac84-2450-4275-b26e-8e0d9d741100", "6f69884b-562…
    ## $ user_guid        <chr> NA, NA, NA, NA, NA, NA, NA, "7defcf1b-69f6-4bc5-8b52-…
    ## $ started          <dttm> 2022-02-23 01:54:12, 2022-02-23 03:17:46, 2022-02-23…
    ## $ ended            <dttm> 2022-02-23 01:54:45, 2022-02-23 03:19:25, 2022-02-23…
    ## $ data_version     <int> 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1,…
    ## $ session_duration <drtn> 33 secs, 99 secs, 3685 secs, 61 secs, 59 secs, 51 se…

The identifiers used for the content in RStudio Connect are GUIDs. We
can retrieve content names using the API. The API handles only one GUID
at a time, so `purrr`’s `map_dfr()` is used to iterate through all of
the unique GUIDs in order to get every Shiny app’s title.

``` r
# Get the title of each Shiny app
shiny_rsc_titles <- shiny_rsc %>%
  count(content_guid) %>% 
  pull(content_guid) %>%
  purrr::map_dfr(
    ~tibble(content_guid = .x, content_name = content_title(client, .x))
    )

glimpse(shiny_rsc_titles)
```

    ## Rows: 40
    ## Columns: 2
    ## $ content_guid <chr> "0287f7d9-4d55-4813-8852-680f54beaad1", "06484fbb-f686-42…
    ## $ content_name <chr> "Example Palmer Penguins Shiny Dashboard", "Classroom Stu…

The new `shiny_rsc_titles` table, and the `shiny_rsc` can be joined to
return the “user readable” title. Using standard `dplyr` and `ggplot2`
functions, we can now determine things such as the top 10 apps based on
how long their average sessions are.

``` r
# Calculate the average session duration and sort
app_sessions <- shiny_rsc %>%
  inner_join(shiny_rsc_titles, by = "content_guid") %>%
  group_by(content_name) %>%
  summarise(avg_session = mean(session_duration)) %>%
  ungroup() %>%
  arrange(desc(avg_session)) %>%
  head(10)
  
# Plot the top 10 used content
app_sessions %>%
  ggplot(aes(content_name, avg_session)) +
  geom_col() +
  scale_y_continuous() +
  geom_text(aes(y = (avg_session + 200), label = round(avg_session)), size = 3) +
  coord_flip() +
  theme_bw() +
  labs(
    title = "RStudio Connect - Top 10", 
    subtitle = "Shiny Apps", 
    x = "", 
    y = "Average session time (seconds)"
    )
```

![](README_files/figure-gfm/analyze_data-1.png)<!-- -->

Learn more about programmatic deployments, calling the server API, and
custom emails [here](https://docs.rstudio.com/user).

# Retrieve user access permissions

``` r
library(connectapi)
client <- connect( # connection to the RSC server
  auth = "manual",
  server = "https://rstudio-connect.heor.co.uk",
  api_key = "your-api-key-or-sysenv-call-goes-herre"
)
dat <- client %>% # tbl detailing apps on the server
  get_content() %>%
  filter(str_detect(content_category, "^$")) # empty string indicates 'app'
guids <- dat %>% # ids for each app on the server
  pull("guid")
contents <- client %>% # lookup to go from content id to content name/title
  get_content(limit = Inf) %>%
  filter(content_category == "") %>%
  select(all_of(c("content_guid"="guid", "content_name"="name", "content_title"="title")))
groups <- client %>% # all groups of users defined on the server
  get_groups(limit = Inf)
group_guids <- groups %>% # the ids for each group
  pull(guid)
group_members <- group_guids %>% # list of tbls detailing which user is in which group
  lapply(get_group_members, src = client)
group_user_lookup <- tibble(group_guid = group_guids, tbl = group_members) %>%
  # lookup tbl to go from the id of a group to details about the users in the group
  unnest(cols = all_of("tbl")) %>%
  filter(
    not(locked),
    not(str_detect(first_name, "^$")),
    not(str_detect(last_name, "^$")),
    not(str_detect(email, ".*heor.*"))
  ) %>%
  mutate(name = as.character(glue("{first_name} {last_name}"))) %>%
  select(all_of(c("group_guid", "principal_guid"="guid", "username", "name", "email")))
users <- client %>% # details on all users
  get_users(limit = Inf) %>%
  filter(
    not(locked),
    not(str_detect(first_name, "^$")),
    not(str_detect(last_name, "^$")),
    not(str_detect(email, ".*heor.*"))
  ) %>%
  mutate(name = as.character(glue("{first_name} {last_name}"))) %>%
  select(all_of(c("principal_guid"="guid", "username", "name", "email")))
permissions <- guids %>%
  lapply(content_item, connect = client) %>% # R6 object representing each app
  lapply(get_content_permissions) %>% # the user permissions for each app
  bind_rows() # bundled into a single tbl
permissions_users <- permissions %>% # output for users assigned individually to apps
  filter(principal_type == "user") %>%
  select(all_of(c("principal_guid", "content_guid"))) %>%
  inner_join(users, by = c("principal_guid")) %>%
  inner_join(contents, by = c("content_guid"))
permissions_groups <- permissions %>% # output for users assigned to apps through a group
  filter(principal_type == "group") %>%
  select(all_of(c("group_guid"="principal_guid", "content_guid"))) %>%
  inner_join(group_user_lookup, by = c("group_guid")) %>%
  inner_join(contents, by = c("content_guid"))
permissions_out <- bind_rows(permissions_users, permissions_groups) %>% # output
  select(all_of(c("name", "username", "email", "content_name", "content_title")))
```
