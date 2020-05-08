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

## Training and Using

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

`csvdedupe` returns your input file with an additional column called `Cluster ID`, that either is the numeric id (zero-indexed) of a cluster of grouped records or an `x` if csvdedupe believes the record doesn't belong to any cluster. `csvdedupe` attempts to identify all the rows in the csv that refer to the same thing. Each group of such records are called a cluster. The trained model is also stored in a binary file called `learned_settings`.

Read more about the options and seetings on the root project [csvdedupe][https://github.com/dedupeio/csvdedupe].

## Copyright and Attribution

Copyright (c) 2016 DataMade. Released under the [MIT License](https://github.com/dedupeio/csvdedupe/blob/master/LICENSE.md).
