### [Back to Main-Projects tab](https://github.com/B-White-M/Projects/blob/8607106993c92fbc311fdcfcad864c89a5b70c85/README.md)


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

```m
## Data Source details:

// Establish connection to the SAP HANA database (example data)
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

    // Filter out rows where there is a match in the CONCATENATE Aged table
    #"Filtered Data" = Table.SelectRows(#"Expanded CONCATENATE Aged (Table)", each [#"CONCATENATE Aged (Table).Asset Number"] = null),

    // Remove unnecessary columns that were used for filtering
    #"Removed Columns" = Table.RemoveColumns(#"Filtered Data", {"CONCATENATE Aged (Table).Asset Number"}),

    // Filter out null or empty values in the "Division" and "Company Code" columns
    #"Remove nulls" = Table.SelectRows(#"Removed Columns", each [#"Division"] <> null and [#"Division"] <> "" and [#"Company Code"] <> null and [#"Company Code"] <> ""),

    // Create a final unique identifier by combining company code and division
    #"UID creation" = Table.AddColumn(#"Remove nulls", "UID", each Text.Combine({Text.From([Company Code], "en-US"), [Division]}, "_"), type text)

in
    #"UID creation"
```

### DAX Code used:
Here is an example of the DAX Code code used in the project

```DAX
Data Source details:


