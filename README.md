### lakehouse_utils:

The purpose of the lakehouse utils package is threefold 

I) Expedite the time and level of effort for migrating pipelines from cloud data warehouses to the Lakehouse  (ie dbt + databricks). This is done by transpiling functions that are not natively available in spark sql to compatible spark sql functions that take in the same input(s) and render the same outputs. This is all done via DBT macros (feel free to reference the macros directory).  

II) Be a centralized source of truth for warehouse function mapping to Databricks function mapping. Also surface instances where certain functions can not be automated and manual intervention is required. You can find the full list of supported functions in the functionlist.csv in the seed directory; you can also find further information in the read.me in the macros directory.  

III) Surface best practices around unit tests to instill confidence that the macros are robust and reliable (feel free to reference the tests directory). 

### Installation instructions:

I) Include this package in your packages.yml — check [here](https://github.com/rlsalcido24/lakehouse_utils/releases/tag/v0.1.1) for the latest version number.

II) Run dbt deps

III) Execute dbt seed -- this project includes a CSV that must be seeded for automated transpilation.

### Database support:

This package has been tested on Snowflake and Databricks.

### Manual 'Hello World' CUJ:  

To get a quick sense of what this module offers, you can reference and run the models in the models directory. Here are the relevant steps:  

I) Insert the relevant values into the profiles.yml based on your warehouse and lakehouse credentials.

II) Build the Snowflake models by executing dbt run --target snow 

III) Build the Databricks models by temporarily updating the models_path key in profiles.yml to be tmp and executing dbt run --target databricks.

IV) Observe that when when the models build on Databricks they transpile the Snowflake functions that invoke macros (wrapped in curly braces). Also observe that while syntax is slightly different in each system the end results are still the same. Also note that manually 'migrating' these two models from scratch should take no longer than 5 mins-- it is just a matter of wrapping the relevant function in braces and wrapping the input parameters in quotes. 

### Automated 'to the moon' CUJ 

I) Create a seperate dev branch of your dbt project. Execute dbt seed (if you haven't already). Run the helper function_to_macro.py file on Databricks. Obvserve that all your Snowflake models have now been automatically refactored to reference relevant macros, therefore making it possible to build these models in Databricks.

II) dbt run.

III) Execute unit tests to ensure that the models built in Databricks match the models built on Snowflake.

IV) Once you have sufficient confidence copy the directory from the models directory of the dev branch to the models directory of the main branch and name the directory something different to differentiate it from the Snowflake models (ie Snow and Databricks) 

V) Build models in both systems until sufficient confidence is instilled to run the models solely on one system.

## Macros:

### zeroifnull ([source](https://github.com/rlsalcido24/lakehouse_utils/blob/main/macros/zeroifnull.sql))

This macro takes in a column input and leverages the nvl function to return 0 if the input is null.
```
    {% macro zeroifnull(column_name) %}
    nvl({{column_name}}, 0)
    {% endmacro %}
```

Arguments:

a) Column name

### to_number ([source](https://github.com/rlsalcido24/lakehouse_utils/blob/main/macros/to_number.sql))

This macro takes in an expression and optional format/precision/scale and returns a decimal with default formatting or the specified formatting.
```
   {% macro to_number(expr, format, precision, scale) %}


   {% if scale %}

   cast({{expr}} as decimal({{precision}}, {{scale}}))


   {% elif precision %}

   cast({{expr}} as decimal({{format}}, {{precision}}))


   {% elif format %}

   to_number({{expr}}, "{{format}}")


   {% else %}

   cast({{expr}} as decimal(38, 0))

   {% endif %}	

   {% endmacro %}
```

Arguments:

a) Expr <br>
b) format (optional) <br>
c) precision (optional) <br>
d) scale (optional) 

### timestampadd ([source](https://github.com/rlsalcido24/lakehouse_utils/blob/main/macros/timestampadd.sql))

This macro takes in a time unit and adds the unit to an existing timestamp. 
```
    {% macro timestampadd(unit, measure, base) %}
    CASE 
    WHEN lower({{unit}}) = 'year'   THEN {{base}} + make_interval({{measure}})
    WHEN lower({{unit}}) = 'month'  THEN {{base}} + make_interval(0, {{measure}})
    WHEN lower({{unit}}) = 'day'    THEN {{base}} + make_interval(0, 0, 0, {{measure}})
    WHEN lower({{unit}}) = 'hour'   THEN {{base}} + make_interval(0, 0, 0, 0, {{measure}})
    WHEN lower({{unit}}) = 'minute' THEN {{base}} + make_interval(0, 0, 0, 0, 0, {{measure}})
    WHEN lower({{unit}}) = 'second' THEN {{base}} + make_interval(0, 0, 0, 0, 0, 0, {{measure}})
    END
  {% endmacro %}
```

Arguments:

a) time unit <br>
b) measure (ie number of time units to add) <br>
c) base timestamp

### timestampdiff ([source](https://github.com/rlsalcido24/lakehouse_utils/blob/main/macros/timestampdiff.sql))

This macro takes in two timestamps and calculates the difference by quantity of units.
```
    {% macro timestampdiff(unit, arg1,arg2) %}
    CASE 
    WHEN lower({{unit}}) = 'year'   THEN EXTRACT(YEAR FROM {{arg2}}) - EXTRACT(YEAR FROM {{arg1}})
    WHEN lower({{unit}}) = 'month'  THEN (EXTRACT(YEAR FROM {{arg2}}) * 12 + EXTRACT(MONTH FROM 
    {{arg2}}))
                          - (EXTRACT(YEAR FROM {{arg1}}) * 12 + EXTRACT(MONTH FROM {{arg1}}))
    WHEN lower({{unit}}) = 'day'    THEN datediff(CAST({{arg2}} AS DATE), CAST({{arg1}} AS DATE))
    WHEN lower({{unit}}) = 'hour'   THEN EXTRACT(HOUR FROM {{arg2}}) - EXTRACT(HOUR FROM {{arg1}})
    WHEN lower({{unit}}) = 'minute' THEN (EXTRACT(HOUR FROM {{arg2}}) * 60 + EXTRACT(MINUTE FROM 
    {{arg2}}))
                          - (EXTRACT(HOUR FROM {{arg1}}) * 60 + EXTRACT(MINUTE FROM {{arg1}}))
    WHEN lower({{unit}}) = 'second' THEN (EXTRACT(HOUR FROM {{arg2}}) * 3600 + EXTRACT(MINUTE FROM 
    {{arg2}}) * 60 + EXTRACT(SECOND FROM {{arg2}}))
                          - (EXTRACT(HOUR FROM {{arg2}}) * 3600 + EXTRACT(MINUTE FROM {{arg2}}) * 
    60 + EXTRACT(SECOND FROM {{arg2}}))
    END
    {% endmacro %}
```

Arguments:

a) unit <br>
b) timestamp being subtracted <br>
c) timesamp being subtracted from

### dayname ([source](https://github.com/rlsalcido24/lakehouse_utils/blob/main/macros/dayname.sql))

This macro takes in a date and returns a string for day of week of that date.
```
    {% macro dayname(arg) %}
     CASE 
         WHEN datediff(CAST({{arg}} AS DATE), DATE'1799-12-29') % 7 = 0 THEN 'Sun'
         WHEN datediff(CAST({{arg}} AS DATE), DATE'1799-12-29') % 7 = 1 THEN 'Mon'
         WHEN datediff(CAST({{arg}} AS DATE), DATE'1799-12-29') % 7 = 2 THEN 'Tue'
         WHEN datediff(CAST({{arg}} AS DATE), DATE'1799-12-29') % 7 = 3 THEN 'Wed'
         WHEN datediff(CAST({{arg}} AS DATE), DATE'1799-12-29') % 7 = 4 THEN 'Thu'
         WHEN datediff(CAST({{arg}} AS DATE), DATE'1799-12-29') % 7 = 5 THEN 'Fri'
         ELSE 'Sat' END
     {% endmacro %}
```

Arguments:

a) date

### Next Steps: 

I) Deploy this logic as a package in the dbt hub so it is simpler to interface with.

Note that we gladly welcome contributions from the partners and from the community-- if intersted please submit a pull request! We can particularly use support with increasing our surface area of supported functions. When submitting a PR please include a unit test for each new macro added-- as long as any new unit tests pass, and existing tests continue to pass, we should be able to GTM (example pr template in .github directory). If you have a request to support a particular function please do log it as an enhancement in issues and happy building!!
