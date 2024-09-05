### [Back to Main-Projects tab](https://github.com/B-White-M/Projects/tree/main)


# Global_Assets_Dashboard:

Power BI project that analyzes and visualizes asset management data, identifies recurring errors and potential financial risks, and includes a real-time location visualization map connected to an SQL environment to support decision-making.
-This Dashbard saved >1250 hours annually and represents a saving of >$850K USD.


# Brief explanation of the logic used:

In response to the need to identify and mitigate potential legal and financial impacts on capital products, I developed a system in Power BI that performs a real-time analysis of these assets. This system constantly monitors the values ​​of these products and displays potential financial impacts, broken down by their geographic location.

The globally published dashboard has become a key tool for preventing adverse financial impacts on the company, allowing executives and stakeholders to make decisions based on up-to-date and accurate data.


## Technologies Used: 

### - Power BI
### - Real-Time Data Integration
### - DAX for Data Transformation
### - M (query) data gathering
### - SQL for Data Collection and Connection 


## Key Features

### Real-Time Visualization: 
The dashboard collects real-time data from various sources to provide a clear picture of potential legal and financial impacts.
### Geographic Analysis: 
Break down impacts by geographic location, making it easier to identify localized risks.
### Proactive Alerts: 
Identify potential legal impacts, giving teams the ability to take preventative action.
### Global Publishing: 
The dashboard is globally accessible by different stakeholders within the organization for constant monitoring.


## Results:

- Reduction of potential financial impacts due to proactive problem detection.
- Global implementation of the visualization system, currently used by different levels of the organization.
- This Dashbard saved >1250 hours annually and represents a saving of >$850K USD.


# M (query) Code used:
Here is an example of the Code M (Power Query) code used in the project

## Data Source details:

```m
let
    Source = SapHana.Database("EXAMPLE", [EXAMPLE2="2.0"]),

// Extract content from the specified source
Contents = Source{...}[...],

    // Retrieve the data from the company database
    AssetListQuery1 = COMPANY.finance.COMPANY{...},

    // Add and apply parameters to filter the data
    #"Added Parameters" = Cube.Transform(AssetListQuery1,
    {
        {Cube.ApplyParameter, "IP_Fiscal_Year", {"XX"}},    // Filter by fiscal year
        {Cube.ApplyParameter, "IP_Period", {"XX"}},         // Filter by period
        {Cube.ApplyParameter, "IP_DeprArea", {"XX"}},       // Filter by depreciation area
        {Cube.ApplyParameter, "ip_AcquisitionDateStart", {#date(1900, 1, 1)}},  // Acquisition start date
        {Cube.ApplyParameter, "ip_AcquisitionDateEnd", {#date(9999, 12, 30)}},   // Acquisition end date
        {Cube.ApplyParameter, "IP_AssetCapitalizationStart", {#date(1900, 1, 1)}}, // Asset capitalization start date
        {Cube.AddAndExpandDimensionColumn, "..."},
        {Cube.AddMeasureColumn, "Accum Depr", "[Measures].[AccumDepreciation]"},   // Accumulated depreciation
        {Cube.AddMeasureColumn, "Net Book Value", "[Measures].[NetBookVal]"},     // Net book value
        {Cube.AddMeasureColumn, "APC Amount", "[Measures].[APCAmount]"}           // Asset Purchase Cost (APC) amount
    }),

in
    #"Added Parameters"
```

## Data Clean-up:

```m code

    // Change data types of selected columns for further processing
    #"Changed Type" = Table.TransformColumnTypes(#"Added Parameters", {...}),

    // Filter rows based on conditions: age, net book value, and APC amount
    #"Filtered Rows" = Table.SelectRows(#"Changed Type", 
    each Date.Year(DateTime.LocalNow()) - Date.Year([Asset Capitalization Date]) >= 10 
    and [Net Book Value] = 0 
    and [APC Amount] >= 10000 
    and List.Contains({"Class1", "Class2"}, [Asset Class]) 
    and List.Contains({"Eval1", "Eval2", "Eval3"}, [Eval Group 1])
    ),

    // Create a unique ID column by combining plant and location
    #"Lat_Long Setup" = Table.AddColumn(#"Filtered Rows", "Plant_UID", each Text.Combine({Text.From([Asset Plant], "en-US"), [Asset Location]}, "_"), type text),

    // Rename columns for clarity
    #"Renamed Columns" = Table.RenameColumns(#"Lat_Long Setup", {{"Asset Company", "Company Code"}, {"OPS1 Super Group Name", "Super Group Full Name"}, {"OPS1 Division Name", "Division Full Name"}, {"OPS1 Group Name", "Group Full Name"}}),

    // Duplicate columns for further transformations
    #"Duplicated Column Div" = Table.DuplicateColumn(#"Renamed Columns", "Division Full Name", "Division"),
    #"Duplicated Column SG" = Table.DuplicateColumn(#"Duplicated Column Div", "Super Group Full Name", "Super Group"),
    #"Duplicated Column Group" = Table.DuplicateColumn(#"Duplicated Column SG", "Group Full Name", "Group"),

    // Split text by delimiter in the duplicated columns to separate values
    #"Split Column by Div" = Table.SplitColumn(#"Duplicated Column Group", "Division", Splitter.SplitTextByDelimiter("|", QuoteStyle.Csv), {"Division"}),
    #"Split Column by SG" = Table.SplitColumn(#"Split Column by Div", "Super Group", Splitter.SplitTextByDelimiter("|", QuoteStyle.Csv), {"Super Group"}),
    #"Split Column by Group" = Table.SplitColumn(#"Split Column by SG", "Group", Splitter.SplitTextByDelimiter("|", QuoteStyle.Csv), {"Group"}),

    // Create a combined unique identifier based on company and division
    #"UID Column" = Table.AddColumn(#"Split Column by Group", "Company&Group UID", each Text.Combine({[Company Code], [Division]}, "_"), type text),

    // Calculate the aging of assets based on capitalization date
    Aging = Table.AddColumn(#"UID Column", "Aging (years)", each Date.Year(DateTime.LocalNow()) - Date.Year([Asset Capitalization Date])),

    // Change the type of the "Aging" column to integer
    #"Change Type Aging" = Table.TransformColumnTypes(#"Aging", {{"Aging (years)", Int64.Type}}),

    // Merge queries to identify assets not found in a secondary dataset
    #"Merged Queries" = Table.NestedJoin(#"Change Type Aging", {"Asset Full Number"}, #"CONCATENATE Aged", {"Asset Number"}, "CONCATENATE Aged (Table)", JoinKind.LeftAnti),

    // Expand the merged table to expose the relevant columns
    #"Expanded CONCATENATE Aged (Table)" = Table.ExpandTableColumn(#"Merged Queries", "CONCATENATE Aged (Table)", {"Asset Number"}, {"CONCATENATE Aged (Table).Asset Number"}),

```
## Data integration for Primary Key generation:

```m

    // Filter out rows where there is a match in the CONCATENATE Aged table
    #"Filtered Data" = Table.SelectRows(#"Expanded CONCATENATE Aged (Table)", each [#"CONCATENATE Aged (Table).Asset Number"] = null),

    // Remove unnecessary columns that were used for filtering
    #"Removed Columns" = Table.RemoveColumns(#"Filtered Data", {"CONCATENATE Aged (Table).Asset Number"}),

    // Filter out null or empty values in the "Division" and "Company Code" columns
    #"Remove nulls" = Table.SelectRows(#"Removed Columns", each [#"Division"] <> null and [#"Division"] <> "" and [#"Company Code"] <> null and [#"Company Code"] <> ""),

    // Create a final unique identifier by combining company code and division
    #"UID creation" = Table.AddColumn(#"Remove nulls", "UID", each Text.Combine({Text.From([Company Code], "en-US"), [Division]}, "_"), type text)

```

## Primary Key integration across multiple repositories (Power BI report server):

```
// Combining multiple tables into a single table for further processing
    Source = Table.Combine({#"CONCATENATE Aged", #"CONCATENATE Misaligned CC",  #"CONCATENATE Missing Main", #"CONCATENATE Missing UAI"}),

    // Selecting only the relevant columns from the combined table
    #"Removed Other Columns" = Table.SelectColumns(Source,{"Super Group", "Division", "Company Code", "Group"}),

    // Changing the data type of certain columns to ensure consistency
    #"Changed Type" = Table.TransformColumnTypes(#"Removed Other Columns",{{"Division", type text}, {"Company Code", type text}}),

    // Creating a new column called "UID" by merging the "Company Code" and "Division" columns with an underscore separator
    #"Inserted Merged Column" = Table.AddColumn(#"Changed Type", "UID", each Text.Combine({[Company Code], [Division]}, "_"), type text),

    // Filtering rows to remove any records where "Division" or "Company Code" are null or empty
    #"Filtered Rows" = Table.SelectRows(#"Inserted Merged Column", each [#"Division"] <> null and [#"Division"] <> "" and [#"Company Code"] <> null and [#"Company Code"] <> ""),

    // Removing duplicate rows based on the unique "UID" column to maintain uniqueness
    #"Removed Duplicates" = Table.Distinct(#"Filtered Rows", {"UID"})

```
## Primary Key integration across multiple repositories

![Intvw 1](https://github.com/user-attachments/assets/af1c91a4-71e4-4093-a7e0-a5ee2055fca2)

## Table of inputs provided generated, based on the Primary Key.

![Intvw 4](https://github.com/user-attachments/assets/cd0a322a-26a4-4b56-b90e-8960b7a009d9)


## Query that extracts data from an Excel, processes location-related data (using web-scrapping data), and prepares it for mapping by sorting and filtering important columns.

```
// Extracting the "Locations_Table" from an Excel workbook
    Source = Excel.Workbook(File.Contents("Path/To/Location.xlsx"), null, true),
    Locations_Table = Source{[Item="Locations_Table",Kind="Table"]}[Data],

    // Changing the data types of columns to their correct formats (text, number, etc.)
    #"Changed Type" = Table.TransformColumnTypes(Locations_Table,{{"Company", type text}, {"Plant", type text}, {"Location", type text}, {"Latitude", type number}, {"Longitude", type number}, {"Location Status", type text}}),

    // Filtering out rows where "Location" or "Plant" data is missing
    #"Remove Nulls" = Table.SelectRows(#"Changed Type", each [#"Location"] <> null and [#"Location"] <> "" and [#"Plant"] <> null and [#"Plant"] <> ""),

    // Selecting relevant columns for mapping and analysis
    #"Map Lat&Long" = Table.SelectColumns(#"Remove Nulls",{"Plant", "Location", "Latitude", "Longitude", "Location Status"}),

    // Creating a new column "Plant_UID" by merging "Plant" and "Location" with an underscore separator for unique identification
    #"Inserted Merged Column Location" = Table.AddColumn(#"Map Lat&Long", "Plant_UID", each Text.Combine({Text.From([Plant], "en-US"), [Location]}, "_"), type text),

    // Sorting the data by the "Plant" column for better readability and analysis
    #"Sorted Rows" = Table.Sort(#"Inserted Merged Column Location",{{"Plant", Order.Ascending}})

```
## Visualization that extracts data from an Excel, processes location-related data 

![Intvw 2](https://github.com/user-attachments/assets/e9ac9b17-af6f-4fb3-8d92-af8c33fb1177)

## Visualization (with filter applied) that extracts data from an Excel, processes location-related data 

![Intvw 3](https://github.com/user-attachments/assets/4ebbad69-dc33-44be-919a-f4b224f3f6ad)


## DAX Code used:
Here is an example of the DAX Code code used in the project

### Measure #1:

```dax
// If  the count of 'Asset_Number' entries from the 'Aged SAMPLE' table = 0, it returns 0; otherwise, it returns the total count.
Aged Full Number =
IF (
    COUNT ( 'Aged SAMPLE'[Asset_Number] ) = 0,
    0,
    COUNT ( 'Aged SAMPLE'[Asset_Number] )
)

```

### Measure #2:

``` dax
// This measure calculates the difference between the total number of assets and the number of assets with input owners in the 'SAMPLE' table.
// If all assets have input owners, it returns 0; otherwise, it returns the difference.
CONCAT SAMPLE Total = 
VAR CurrentYear = YEAR(TODAY())
VAR CurrentQuarter = QUARTER(TODAY())
VAR AssetCount = 

CALCULATE(
    COUNT('CONCATENATE  SAMPLE'[Asset Number]),
    YEAR('CONCATENATE  SAMPLE'[Quarter]) = CurrentYear,
    QUARTER('CONCATENATE  SAMPLE'[Quarter]) = CurrentQuarter
)
RETURN
IF(
    (AssetCount)=0,
    0,
    AssetCount)

```

### Measure #3:

``` dax code
// This measure sums the '$_Price' for all assets in the 'Aged SAMPLE' table where the 'Input Group Owner' is not blank and the asset is classified as "Taxable Asset".
TaxableResponses = 
SUMX(
    FILTER(
        'Aged Assets Knime',
        'Aged Assets Knime'[Input Group Owner] <> "" &&
        'Aged Assets Knime'[Taxable Asset] = "Taxable Asset"
    ),
    'Aged Assets Knime'[$_Price]
)
```

### [Back to Main-Projects tab](https://github.com/B-White-M/Projects/tree/main)
