---
layout: post
title:  "ArcGIS API for Python Cookbook: Update string field storing boolean values to integer field with domain"
date:   2024-11-29 12:19:20 -0800
categories: jekyll update
---
### Issue
![Screenshot of string field storing boolean attributes -- bad!]({{site.url}}/_assets/YesNoFieldWithoutDomain.png)

Storing boolean attributes (e.g. `Yes`/`No` values) in a string field without any domain is the most common data quality issue I encounter when managing feature layers in ArcGIS Online/Enterprise. Doing so introduces the potential for data inconsistency, increases the complexity of maintenance/analysis, and wastes space. If this issue gets introduced, it tends to persist, since there's currently no easy fix.

### Solution
![Screenshot of integer field with domain storing boolean attributes -- good!]({{site.url}}/_assets/YesNoFieldWithDomain.png)

Here's a quick snippet for updating a feature layer string field storing `Yes`/`No` values to an integer field with a `Yes`=`1`/`No`=`0` domain:
{% highlight python %}
from getpass import getpass

from arcgis.gis import GIS

gis = GIS("https://arcgis.com", "<username>", getpass())

questionable_data_choices_item = gis.content.get("<item_id>")
questionable_data_choices_layer = questionable_data_choices_item.layers[<layer_index>]

# Create a temporary integer field to store the converted string values
temp_new_field = {
    "name": "TempActiveWell",
    "alias": "Temp Active Well",
    "type": "esriFieldTypeInteger",
    "editable": True,
    "nullable": True,
    "defaultValue": None,
    "description": None
}
questionable_data_choices_layer.manager.add_to_definition({"fields": [temp_new_field]})

# Convert string values to integers using SQL CASE expression
sql_expression = "CASE WHEN ActiveWell = 'Yes' THEN 1 WHEN ActiveWell = 'No' THEN 0 ELSE NULL END"
questionable_data_choices_layer.calculate(where="1=1", calc_expression={"field": "TempActiveWell", "sqlExpression": sql_expression})

# Now that values are converted and stored in temp field, delete original string field and recreate as integer field with domain
questionable_data_choices_layer.manager.delete_from_definition({"fields": [{"name": "ActiveWell"}]})
new_field = {
    "name": "ActiveWell",
    "alias": "Active Well",
    "type": "esriFieldTypeInteger",
    "editable": True,
    "nullable": True,
    "defaultValue": None,
    "description": None,
    "domain": {
        "type" : "codedValue",
        "codedValues" : [
            {
                "name" : "Yes",
                "code" : 1
            },
            {
                "name" : "No",
                "code" : 0
            }
        ]
    }
}

# Transfer values from temp field to new integer domain field
questionable_data_choices_layer.calculate(where="1=1", calc_expression={"field": "ActiveWell", "sqlExpression": "TempActiveWell"})

# Delete temp field
questionable_data_choices_layer.manager.delete_from_definition({"fields": [{"name": "TempActiveWell"}]})
{% endhighlight %}