# csvdedupe

Command line tools for using the [dedupe python library](https://github.com/dedupe.io/dedupe/) for deduplicating CSV files. It is a fork of [csvdedupe][https://github.com/dedupeio/csvdedupe].
## Installation and dependencies

```
pip install csvdedupe
```

## Use Case


By example, we use data preprocessing and ML to, respectively, filter down about 21k bp software listing to about 1k, and then collapse them down to 230 titles. I use csvdedup for the latter.

The titles are listed in a csv filed, called software.csv. There are three columns or attributes in the file:
1. productname e.g. Postgresql Database Server
2. productvendor e.g. Oracle
3. productversion e.g. 11.x

**I define a config file :**

```json
{
  "field_names": ["productname", "productvendor", "productversion"],
  "field_definition" : [{"field" : "productname", "type" : "String"},
                        {"field" : "productversion", "type" : "Exact",
                         "has_missing" : true}],
  "output_file": "group_software.csv",
  "skip_training": false,
  "training_file": "training.json",
  "sample_size": 1500000, 
  "recall_weight": 2
}
```

## Training and application together

```bash
csvdedupe software.csv --config_file=config.json
```

We need to provide a set of labeled examples to train the classifier in the csvdedup system. The system develops rules for classification. For the first time, csvdedup requires training, namely to classify two records as (y) duplications, (n) not-duplicates, or (u) unsure. On subsequent runs, the training is optional and augmentative.

The more labeled examples are provided, the better the deduplication results will be. At minimum, we must  provide __10 positive matches__ and __10 negative matches__.

The results of your training will be saved in a JSON file ( __training.json__, unless specified otherwise with the `--training-file` option) for future runs of csvdedupe.

Here's an example labeling operation:

```bash
productname : sql server 2008 r2 transact-sql language service
productversion : 10.x

productname : microsoft sql server 2008 r2 transact-sql language service
productversion : 10.x

28/10 positive, 2/10 negative
Do these records refer to the same thing?
(y)es / (n)o / (u)nsure / (f)inished / (p)revious
```

The training is optional and augmentative on subsequent runs.

The output file, group_software.csv, is identical to software.csv with the addition of cluster_id. The training is also stored in a file called learned_settings.

Read more about the options and seetings on the root project [csvdedupe][https://github.com/dedupeio/csvdedupe].

I'd like to thank my mentors (Mark and Hemant) to give me the opportunity and seed work to filter and collapse 21k bp software listing to about 1k. You may be hearing it has been a great success so far.

I used data science and ML for grouping software in clusters. Feel awesome to contribute meaningfully to the project. 


## Training


## Output
`csvdedupe` attempts to identify all the rows in the csv that refer to the same thing. Each group of
such records are called a cluster. `csvdedupe` returns your input file with an additional column called `Cluster ID`,
that either is the numeric id (zero-indexed) of a cluster of grouped records or an `x` if csvdedupe believes
the record doesn't belong to any cluster.

`csvlink` operates in much the same way as `csvdedupe`, but will flatten both CSVs in to one
output file similar to a SQL [OUTER JOIN](http://stackoverflow.com/questions/38549/difference-between-inner-and-outer-join) statement. You can use the `--inner_join` flag to exclude rows that don't match across the two input files, much like an INNER JOIN.


## Preprocessing
csvdedupe attempts to convert all strings to ASCII, ignores case, new lines, and padding whitespace. This is all
probably uncontroversial except the conversion to ASCII. Basically, we had to choose between two ways of handling
extended characters.

```
distance("Tomas", "Tomás')  = distance("Tomas", "Tomas")
```

__or__

```
distance("Tomas, "Tomás") = distance("Tomas", "Tomzs")
```

We chose the first option. While it is possible to do something more sophisticated, this option seems to work pretty well
for Latin alphabet languages.

## Testing

Unit tests of core csvdedupe functions
```bash
pip install -r requirements-test.txt
nosetests
```

## Community
* [Dedupe Google group](https://groups.google.com/forum/?fromgroups=#!forum/open-source-deduplication)
* IRC channel, #dedupe on irc.freenode.net

## Recipes

### Combining and deduplicating files from different sources.

Lets say we have a few sources of early childhood programs in Chicago and we'd like to get a canonical list.
Let's do it with `csvdedupe`, `csvkit`, and some other common command line tools.

#### Alignment and stacking
Our first task will be to align the files and have the same data in the same columns for stacking.

First, let's look at the headers of the files.

File 1
```console
> head -1 CPS_Early_Childhood_Portal_Scrape.csv
"Site name","Address","Phone","Program Name","Length of Day"
```

File 2
```console
> head -1 IDHS_child_care_provider_list.csv
"Site name","Address","Zip Code","Phone","Fax","IDHS Provider ID"
```

So, we'll have to add "Zip Code", "Fax", and "IDHS Provider ID"
to ```CPS_Early_Childhood_Portal_Scrape.csv```, and we'll have to add "Program Name",
"Length of Day" to ```IDHS_child_care_provider_list.csv```.

```console
> cd examples
> sed '1 s/$/,"Zip Code","Fax","IDHS Provider ID"/' CPS_Early_Childhood_Portal_Scrape.csv > input_1a.csv
> sed '2,$s/$/,,,/' input_1a.csv > input_1b.csv
```

```console
> sed '1 s/$/,"Program Name","Length of Day"/' IDHS_child_care_provider_list.csv > input_2a.csv
> sed '2,$s/$/,,/' input_2a.csv > input_2b.csv
```

Now, we reorder the columns in the second file to align to the first.

```console
> csvcut -c "Site name","Address","Phone","Program Name","Length of Day","Zip Code","Fax","IDHS Provider ID" \
         input_2b.csv > input_2c.csv
```

And we are finally ready to stack.

```console
> csvstack -g CPS_Early_Childhood_Portal_Scrape.csv,IDHS_child_care_provider_list.csv \
           -n source \
           input_1b.csv input_2c.csv > input.csv
```

#### Dedupe it!
And now we can dedupe

```console
> cat input.csv | csvdedupe --field_names "Site name" Address "Zip Code" Phone > output.csv
```

Let's sort the output by duplicate IDs, and we are ready to open it in your favorite spreadsheet program.

```console
> csvsort -c "Cluster ID" output.csv > sorted.csv
```

## Errors and Bugs

If something is not behaving intuitively, it is a bug, and should be reported.
Report it [here](https://github.com/dedupeio/csvdedupe/issues).

## Patches and Pull Requests
We welcome your ideas! You can make suggestions in the form of [github issues](https://github.com/dedupeio/csvdedupe/issues) (bug reports, feature requests, general questions), or you can submit a code contribution via a pull request.

How to contribute code:

- Fork the project.
- Make your feature addition or bug fix.
- Send us a pull request with a description of your work! Don't worry if it isn't perfect - think of a PR as a start of a conversation, rather than a finished product.

## Copyright and Attribution

Copyright (c) 2016 DataMade. Released under the [MIT License](https://github.com/dedupeio/csvdedupe/blob/master/LICENSE.md).
