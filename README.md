# data-analysis-with-python


## This is what we are going to create today, which is 100% python using the Dash framework:

![dataanalysis](https://user-images.githubusercontent.com/90658763/185744892-02539409-90fc-4327-b021-e7589ef79bd4.gif)


## Expectations

### If you’re reading through this top to bottom, here is what you can expect to get out of this article:

#### 1. Learn to build a professional quality data analysis dashboard, completely in python using the Dash / Plotly frameworks using a non-standard data set.
#### 2. Obtaining the data.
#### 3. Understanding the data and the challenges it will pose.
#### 4. Methods used to manipulate the data in various forms.
#### 5. Construction of the dashboard framework to view and analyze the data.
#### 6. Developing several distinct chart types using plotly libraries to visualize the data in different ways.


## Requirements
### To work through this article, you will need several things that are all easily avialable:

### Computer that can run Python (at least version 3.7 though I developed this using python 3.9)
### The modules listed in the “requirements.txt” file
#### Some nice-to-haves, but not specifically required is a Fed API Key for updating the data using the pull_fed_data.py script.

## Data Discussion
### The first step in any analysis project is to know and understand the data you will work with. This is no different. In fact, this data is specifically a little more challenging, so it does require some understanding of what’s going on.

A sample data file is located in the repository directory named data/fed_data.csv:

![image](https://user-images.githubusercontent.com/90658763/185745357-83ad3003-9a89-4b39-9b96-d1c2e0a51b3c.png)

Sample of fed_data.csv

First Steps
We know that we need to read in our data and probably do some formatting. We also know that we are going to output this to a dashboard for use. Let’s set that up.

Much like in my commodities dashboard (linked in the intro) I started with a basic framework all included in the repository:

main.py — this is the layout file and driver for the dash dashboard. More on that later.
business_logic.py — this is the file that is called by main.py and sets up some of our global dataframes and constructs.
support_functions.py — this is the workhorse for processing and functional logic.
layout_configs.py — this is a container file to organize various layouts and visual elements for charts.
For these initial steps, we will concern ourselves with the support_functions.py file.

We have a basic understanding of the format of our data, so our first task is to get it into the system in a manner that we can readily work with. For this, we read the whole data file into a Pandas dataframe.
```python3```
# Base retrieval function - reads from CSV

```python3
def get_fed_data():
    file_path = base_path + "fed_data.csv"
    # file_path = base_path + "fed_dump.csv"
    df = pd.read_csv(file_path, na_values="x")
    df.rename(
        {"data": "report_data", "hash": "report_hash"},
        axis=1,
        inplace=True,
    )
    df["report_date"] = df["report_date"].values.astype("datetime64[D]")
    df["release_date"] = df["release_date"].values.astype("datetime64[D]")

    return df
```

This function is pretty straightforward. We pull the configurable base_path variable (not shown) into a format for file_path, which is fed into the read_csv pandas function to create dataframe “df”. From there, we simply rename some columns for consistency and ensure data types are correct for the dates. Then we return the processed data frame.

All simple so far.

However, we can’t just plunk this raw dataframe into a chart and let the magic algorithms do their thing. We need to think about what we need to do with that data individually.

Here lies a challenge with a project like this. No one gave me a spec sheet or instructions for what I needed to get out of it. I had to figure it out myself. The horror…

## Processing

### When faced with ambiguous requirements, you often have to think in generic terms. That’s where I started this stage.

I knew I would need to extract the data based on various criteria and formats, so I began creating some helper functions that could be called individually or as part of a larger process to filter and refine.

I won’t go through all of them, but this is the general template:

### function to pull specific report

```python3
def get_report_from_fed_data(df1, report_name):
    df = df1.copy()
    df = df[df["report_name"] == report_name]
    df.sort_values(by=["report_date"], inplace=True)
    df.reset_index(drop=True, inplace=True)
    return df
 ```
    
This specific function is defined to pull all rows of an input dataframe based on a report name that is passed in.

I first begin by making a copy of the dataframe. I do this because if you modify data within a dataframe slice, you will get warnings from pandas. I HATE warnings like that.

The next line simply does the filter on the copy of the passed-in dataframe. This line is mostly what changes from helper function to helper function through this section of code.

I finally sort the resulting data frame appropriately and logically based on what the function is designed to do and then reset the index. The last step is so that the returned dataframe has a consistent index that only encompasses that rows returned.

I have helper functions for filtering to an exact report_date or a report_date greater than or equal to a passed in date as well as similar functions for release_date. All in all, these cover most of the conditions I could see being used to create charts.

Taking the time now to plan out what is possibly needed is ultimately much more efficient than trying to do it on the fly. Thinking about the underlying data and anticipating use cases provides a better contextual understanding and can often save time down the road.

## Data Labelling

### As I began to think more about my end results, I realized I had an annoying gap that I should fix. It goes back to that pesky report_name value and the cryptic nature of it.

To be more user friendly, I should provide a long name that can be referenced. So, I created another function. This is abbreviated here just for sake of brevity, but the core is intact:

```python3
def add_report_long_names(df1):
    df = df1.copy()
        if df.loc[index, "report_name"] == "AMDMUO":
            df.loc[
                index, "report_long_name"
            ] = "Manufacturers Unfilled Orders: Durable Goods"
            df.loc[index, "category"] = "Production"
            return df
  ```
  
The snippet code above relates to the header image I provided on the data. In this, I simply examine the report_name filed and if it is equal to “AMDMUO” I set a report_long_name value equal to “Manufacturers Unfilled Orders: Durable Goods”. I also take the opportunity to put in a category of “Production”.

## Normalization
### Something that is not usually obvious to new analysts working with this type of data is figuring out a method for direct comparison between two wildly different values. How do we normalize that so we can plot a comparison directly on a chart and have it make sense?

A practical way to describe it is how do you plot comparative performance of something like Amazon and Walmart on a chart using stock data? The easiest method is to define the comparison based on rates of change.

The same process works here. And, I do it quite often with type of data.

Luckily, pandas makes short work of it:

```python3
def period_change(df):
    df["period_change"] = df.report_data.pct_change()
    df["relative_change"] = 1 - df.iloc[0].report_data / df.report_data
    return df
```
This function is designed to sit at the end of a filtering chain. It expects that the dataframe fed in will be a distinct report, filtered to a single date and ordered by that date.

The function adds two fields, one for the period change, which is the rate of change from the prior row. The other is the baseline change, which is the percent change from the first row of the dataframe.

But, for this to work, we need that absolute unique, last value:

```python3
def get_latest_data(df1):
    df = df1.copy()
    df = df.sort_values("release_date").groupby("report_date").tail(1)
    df.sort_values(by=["report_date"], inplace=True)
    df.reset_index(drop=True, inplace=True)
    return df
   ```
Again, this is the same template as the other helpers, but in this case, the filter takes the raw dataframe, sorts by release_date, groups by report_date, and pulls the newest entry in the grouped list. As new releases are added, this function makes sure the newest data is used. Simple and effective and allows us to easily feed a clean dataframe into the period_change function.

## Categories
### As soon as I added the category field, I knew I wanted to use it. I still wasn’t sure how specifically, but I did need a means of getting all relevant data from a category and not a report.

Unlike the other helper functions that build upon each other, this function would have to work on the main data frame. That didn’t thrill me, but it’s the way it has to be.

```python3
def get_category_data_from_fed_data(df1, report_name, report_date):
    df = df1.copy()
    master_list = df["report_name"].unique()
    master_list = pd.DataFrame(master_list, columns=["report_name"])
    master_list = add_report_long_names(master_list)
    filtered_list = master_list[master_list["report_name"] == report_name]
    filtered_list = master_list[
        master_list["category"] == filtered_list.category.iloc[0]
    ]

    df_out = pd.DataFrame()
    for index, row in filtered_list.iterrows():
        temp_df = get_report_from_fed_data(df, row.report_name)
        temp_df = get_report_after_date_fed_data(temp_df, report_date)
        temp_df = get_latest_data(temp_df)
        temp_df = period_change(temp_df)
        temp_df = add_report_long_names(temp_df)
        df_out = df_out.append(
            temp_df,
            ignore_index=True,
        )

    df_out["period_change"] = df_out["period_change"].fillna(0)

    return df_out
 ```
This took a little bit of processing gymnastics, so I’ll step through it.

For inputs, we take the master dataframe along with a specific target report to use as a base to determine category. We also pass in the report_date as a starting point to limit our scope.

Like the other helpers, we create a copy to work on. Again, let’s avoid those warnings.

Our first step is to create a unique list of the reports, which is then converted from an array back to a dataframe. This allows us to feed that list of reports into the add_report_long_names function to give us the long names and categories for each.

We then create a filtered list of just the report name we’re interested in. Ideally, this should only be one row, but I was a bit paranoid so I assumed there might be a condition where more than one row existed. That explains the second level filter to just get the first row’s category.

Finally, we create an empty data frame to hold our results and loop through the rows of the filtered_list against the main dataframe. This loop shows the waterfall filtering in action. For each row, we get the matching report, we then filter the result to get just the dates after the start date, we then take that result and pull just the latest data, we take that result and add the period change fields, then we run that result through the add_report_long_names function to add additional descriptors. At the bottom of the waterfall, we take the result and add it to the output dataframe.

This leaves us with a clean set of report data matching on the category descriptor.

And that’s the data processing portion of our program. We are now ready to put this data to use.

## Dashboard
### Like my commodities report dashboard, I started with a clean slate. I won’t belabor the discussion on that here, but if interested check out the article linked in the introduction.

After much debate and determining what I really wanted to see displayed on a regular basis, I settled on a pretty simple arrangement and design.

The layout is a basic grid as provided by the dash-bootstrap-components module with the CYBORG bootstrap theme applied. I like dark colors for my dashboards.

Note that in order to get my charts to match, I used the plotly template.default setting of “plotly_dark”.

Here is how I set up my layout:

### Row 1: Header and title.
### Row 2: Report Selector with Start Date selector.
### Row 3: Information bar showing latest report date, latest release date, and latest data value.

### Row 4: Raw Data scatter chart.

![image](https://user-images.githubusercontent.com/90658763/185746673-127c02ff-2d56-4e49-90b5-429a4c5bb820.png)

### Row 5: Line charts on periodic changes from last value and from the baseline start date.

![image](https://user-images.githubusercontent.com/90658763/185746695-9937e6ed-d808-499b-9f29-9404d14bf128.png)




