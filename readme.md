# Laboratory 2 - Diving deeper with Neo4j

## Introduction

Travail de Jamal Albadri & Guillaume Pin

### Neo4j

user: neo4j

password: program-prague-nikita-finish-union-9500

## Create docker machine

To create the docker machine containing neo4j I used the following command :

```shell
 docker run --name advdaba_labo2 -p7474:7474 -p7687:7687 -v $HOME/neo4j/logs:/logs -v $HOME/neo4j/data:/data -v $HOME/neo4j/import:/var/lib/neo4j/import --memory="3g" neo4j:latest
```

## Load data

Before loading the data, I had to work with the data to exploit it. The file is in JSON format but neo4j with the basic configuration I have can only import CSV. I checked only and it is possible to import JSON thanks to a neo4j module called APOC but since the configuration given by the professor for the docker machine did not include it I tried to follow an approach using CSV. To do so, I tried to parse the JSON file to CSV using different approach like jq, json2csv but without success. I couldn not do it synchronously due to the necessity to stream the parsing in a file to avoid using RAM to store the 16GB+ file.

But before parsing the data to CSV, I realized that the JSON filed contains type that are not valid for standard JSON : 

```shell
{ 
    "_id" : "53e99784b7602d9701f3e3f5", 
    "title" : "3GIO.", 
    "venue" : {
        "type" : NumberInt(0)
    }, 
    "year" : NumberInt(2011), 
    "keywords" : [

    ], 
    "n_citation" : NumberInt(0), 
    "lang" : "en"
},
```

I had to use a script to get rid of these : 

```python
import json
import re

def preprocess_json(json_file, cleaned_file):
    with open(json_file, 'r', encoding='utf-8') as input_file:
        with open(cleaned_file, 'w', encoding='utf-8') as output_file:
            for line in input_file:
                # Remove MongoDB specific syntax (NumberInt, NumberLong, etc.)
                cleaned_line = re.sub(r'NumberInt\((\d+)\)', r'\1', line)
                cleaned_line = re.sub(r'NumberLong\((\d+)\)', r'\1', cleaned_line)
                output_file.write(cleaned_line)

preprocess_json('dblpv13.json', 'cleaned_output.json')
```

After doing that, I created a docker machine with mongodb and imported my json file into the database.

I tried to work on the data by flattening the arrays and objects using mongosh :

```shell
db.mycollection.aggregate([ { $project: { _id: 1, title: 1, authors: { $reduce: { input: "$authors", initialValue: "", in: { $concat: [ "$$value", { $cond: [{ $eq: ["$$value", ""] }, "", ";"] }, "$$this.name"] } } }, venue_id: "$venue._id", venue_name_d: "$venue.name_d", venue_type: "$venue.type", venue_raw: "$venue.raw", year: 1, keywords: { $reduce: { input: "$keywords", initialValue: "", in: { $concat: [ "$$value", { $cond: [{ $eq: ["$$value", ""] }, "", ";"] }, "$$this"] } } }, fos: { $reduce: { input: "$fos", initialValue: "", in: { $concat: [ "$$value", { $cond: [{ $eq: ["$$value", ""] }, "", ";"] }, "$$this"] } } }, n_citation: 1, page_start: 1, page_end: 1, lang: 1, volume: 1, issue: 1, issn: 1, isbn: 1, doi: 1, pdf: 1, url: { $reduce: { input: "$url", initialValue: "", in: { $concat: [ "$$value", { $cond: [{ $eq: ["$$value", ""] }, "", ";"] }, "$$this"] } } }, abstract: 1, references: { $reduce: { input: "$references", initialValue: "", in: { $concat: [ "$$value", { $cond: [{ $eq: ["$$value", ""] }, "", ";"] }, "$$this"] } } } } }, { $out: "collection" }] )
```

Finally, I exported the collection as a CSV file and moved it back to my machine, in the import folder of neo4j,  from the docker machine : 

```shell
mongoexport --db db --collection collection --type=csv --fields _id,title,authors,venue_id,venue_name_d,venue_type,venue_raw,year,keywords,fos,n_citation,page_start,page_end,lang,volume,issue,issn,isbn,doi,pdf,url,abstract,references --out output.csv
```

When trying to import to neo4j I had an error with certain node containing double quote so I decided to work with the data to remove them by using : 

```shell
sed 's/\\\"//g' out.csv > output.csv
sed -i "s/"//g" output.csv
```

To import the data to neo4j, I used the following cypher script :

```
CREATE INDEX IF NOT EXISTS FOR (a:Article) ON (a._id);
CREATE INDEX IF NOT EXISTS FOR (au:Author) ON (au._id);

:auto LOAD CSV WITH HEADERS FROM "file:///output.csv" AS row
WITH row LIMIT 1000000

CALL {
  WITH row
  CREATE (:Article {
    _id: COALESCE(row._id, ""),
    title: COALESCE(row.title, ""),
    venue_type: COALESCE(row.venue_type, ""),
    venue_raw: COALESCE(row.venue_raw, ""),
    year: COALESCE(row.year, ""),
    keywords: COALESCE(row.keywords, ""),
    fos: COALESCE(row.fos, ""),
    n_citation: COALESCE(row.n_citation, ""),
    page_start: COALESCE(row.page_start, ""),
    page_end: COALESCE(row.page_end, ""),
    lang: COALESCE(row.lang, ""),
    volume: COALESCE(row.volume, ""),
    issue: COALESCE(row.issue, ""),
    issn: COALESCE(row.issn, ""),
    isbn: COALESCE(row.isbn, ""),
    doi: COALESCE(row.doi, ""),
    pdf: COALESCE(row.pdf, ""),
    url: COALESCE(row.url, ""),
    abstract: COALESCE(row.abstract, ""),
    references: COALESCE(row.references, "")
  })
  FOREACH (authorName IN split(row.authors, ';') |
    CREATE (author:Author { name: authorName })
    MERGE (author)-[:AUTHORED]->(:Article { _id: COALESCE(row._id, "") })
  )
  FOREACH (reference_id IN split(row.references, ';') |
    CREATE (article1:Article { _id: COALESCE(row._id, "") })
    CREATE (article2:Article { _id: COALESCE(reference_id, "") })
    MERGE (article1)-[:CITES]->(article2)
  )
} IN TRANSACTIONS;
```

I had an issue with my device storage and could not load the 5 millions entries but I succeed by limiting the entries to 1'000'000 in a few minutes.

Somehow after using :auto and the transactions, I lost the timestamp displayed as an output after the command succeeds.

I spent a long time trying to deal with the format of the data and the cypher script that I did not have time to deploy everything on k8s.

## References

https://aura.support.neo4j.com/hc/en-us/articles/1500012376402-Using-apoc-to-conditional-loading-large-scale-data-set-from-JSON-or-CSV-files

https://neo4j.com/developer/guide-import-csv/#_optimizing_load_csv_for_performance

https://www.microsoft.com/en-us/research/project/open-academic-graph/

https://www.aminer.org/citation

https://neo4j.com/blog/bulk-data-import-neo4j-3-0/

https://chat.openai.com/
