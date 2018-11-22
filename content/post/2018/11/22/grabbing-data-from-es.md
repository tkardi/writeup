---
title: "Downloading data from the ES"
date: 2018-11-22T04:42:00+02:00
draft: false
---

Just yesterday in a very impatient mood I accidentally deleted some data
in the (luckily dev) database. Well, the deletion itself was not accidental,
accidental was the fact that I did it yesterday of all possible days. It was
a very fine identifier column that would have been gone and done with anyway,
but not just yet. Since it was an auto-generated serial number it
proved impossible to regenerate it and get the same values
assigned to the already-existing 1M+ rows. But luckily, as this data is
cached to an ElasticSearch index in a sudden thrush of an idea try to extract it
from there, save as a csv file, and then import that back to the database and
restore the lost identifiers.

And with a bit of Python it was pretty straightforward. Use `requests` for doing
HTTP requests and `csv` for csv-writing (if on py2 and unicode-strings are hell
then `import unicodecsv as csv` can help)

{{< highlight python >}}
import csv
import requests
{{</ highlight >}}

Define a search method that we'll use for both: searching and scrolling through
the remainder of the results

{{< highlight python >}}
def search(indexname='', query='*:*', scroll_id=None):
    if scroll_id:
        # this is a repeating query, simply request the next scroll
        _url = '%s/_search/scroll' % url
        params = dict(
            scroll_id=scroll_id
        )
    else:
        # this is a first-time query
        _url = '/'.join([url, indexname, '_search'])
        params = dict(
            pretty='false',
            size=10000,
            q=query
        )

    # also, pre-first-query tell ES to keep this scroll open for 1 min
    params['scroll']='1m'

    r = requests.get(_url, params=params)
    r.raise_for_status()
    return r.json()
{{</ highlight >}}

And a main to direct the search querys and manage csv writing.

{{< highlight python >}}
def main(outfile, indexname, outfields='*', query='*:*'):
    r = search(indexname, query)
    scroll_id=r['_scroll_id']
    scroll_size=r['hits']['total']
    assert scroll_size > 0, "Nothing found for query '%s'" % q

    if outfields == '*':
        fieldnames = r['hits']['hits']['_source'][0].keys()
    else:
        fieldnames = outfields

    # write mode on outfile
    with open(outfile, 'w') as f:
        c = csv.DictWriter(
            f, delimiter=',', quotechar='"',
            quoting=csv.QUOTE_NONNUMERIC, fieldnames=fieldnames
        )
        c.writeheader()
        for hit in r['hits']['hits']:
            c.writerow(
                {k: v for k, v in hit['_source'].items() if k in fieldnames}
            )

    while scroll_size > 0:
        # this will run as long as there's "hits in the response hits"
        # request the next scroll from the query
        r = search(indexname, query, scroll_id)
        scroll_id = r['_scroll_id']
        scroll_size = len(r['hits']['hits'])

        # append to the csv file
        with open(outfile, 'a') as f:
            c = csv.DictWriter(
                f, delimiter=',', quotechar='"',
                quoting=csv.QUOTE_NONNUMERIC, fieldnames=fieldnames
            )
            for hit in r['hits']['hits']:
                c.writerow(
                    {k: v for k, v in hit['_source'].items() if k in fieldnames}
                )
{{</ highlight >}}

And then we can use it to dump to a csv file `foobar.csv` two properties `foo`
and `bar` from the ES index `your-ES-index-name` available at `url` with

{{< highlight python >}}
url = 'http://127.0.0.1:9200'
main('foobars.csv', 'your-ES-index-name', ['foo', 'bar'], '*:*')
{{</ highlight >}}
