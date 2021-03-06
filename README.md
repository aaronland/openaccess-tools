# go-smithsonian-openaccess

Go package for working with the Smithsonian Open Access release

## Important

This is work in progress. Proper documentation to follow.

## Data sources

This package and the tools it exports support two types of data sources for the Smithsonian Open Access: A local file system and an AWS S3 bucket. Under the hood the code is using the GoCloud [blob](https://godoc.org/gocloud.dev/blob) abstraction layer so other [storage services](https://gocloud.dev/howto/blob/) could be supported but currently they are not.

Access to the data on a local file system is presumed to be from a checkout of the [OpenAccess](https://github.com/Smithsonian/OpenAccess) GitHub repository. That repo has grown sufficiently large that [it can be difficult](https://github.com/Smithsonian/OpenAccess/issues/7) to successfully download a copy of the data.

The data itself also lives in a Smithsonian-operated AWS S3 bucket so this code has been updated to retrieve data from there if asked to. For a number of reasons specific to the Smithsonian retrieving data from their S3 bucket does not fit neatly in to the `GoCloud` abstraction layer but efforts have been made to hide those details from users of this code.

Most of the examples below assume a local Git checkout. For example:

```
$> ./bin/emit -bucket-uri file:///usr/local/OpenAccess metadata/objects/NMAH
```

In order to retrieve data from the Smithsonian-operated S3 bucket you would change the `-bucket-uri` flag to:

```
$> ./bin/emit -bucket-uri 's3://smithsonian-open-access?region=us-west-2' metadata/objects/NMAH
```

Or the following, which is included as a convenience method:

```
$> ./bin/emit -bucket-uri 'si://' metadata/objects/NMAH
```

A by-product of this work is that the code is also able to retrieve data from any other S3 bucket. For example:

```
$> ./bin/emit -bucket-uri 's3://YOUR-OPENACCESS-BUCKET?region=us-east-1' metadata/objects/NMAH
```

As of this writing the code to retrieve data from S3 buckets (other than the Smithsonian's) assumes that those buckets allow public access and have public directory listings enabled.

## Tools

To build binary versions of these tools run the `cli` Makefile target. For example:

```
> make cli
go build -mod vendor -o bin/clone cmd/clone/main.go
go build -mod vendor -o bin/emit cmd/emit/main.go
go build -mod vendor -o bin/findingaid cmd/findingaid/main.go
go build -mod vendor -o bin/location cmd/location/main.go
go build -mod vendor -o bin/placename cmd/placename/main.go
```

### clone

A command-line tool to clone OpenAccess data to a target destination.

This tool was written principally to clone OpenAccess data from the Smithsonian's `smithsonian-open-access` S3 bucket to a local filesystem but it can be used to clone data to and from any supported `GoCloud.blob` source.

```
$> ./bin/clone -h
Usage:
  ./bin/clone [options] [path1 path2 ... pathN]

Options:
  -compress
    	Compress files in the target bucket using bzip2 encoding. Files will be appended with a '.bz2' suffix.
  -force
    	Clone files even if they are present in target bucket and MD5 hashes between source and target buckets match.
  -source-bucket-uri string
    	A valid GoCloud bucket URI. Valid schemes are: file://, s3:// and si:// which is signals that data should be retrieved from the Smithsonian's 'smithsonian-open-access' S3 bucket. (default "si://")
  -target-bucket-uri string
    	A valid GoCloud bucket URI. Valid schemes are: file://, s3://.
  -workers int
    	The maximum number of concurrent workers. This is used to prevent filehandle exhaustion. (default 10)
```

For example:

```
$> ./bin/clone \
	-source-bucket-uri si:// \
	-target-bucket-uri file:///tmp \
	metadata/chndm
	
...time passes

$> ls -al /tmp/metadata/edan/chndm/*.txt 
-rw-------  1 user  wheel   571870 Nov 21 11:31 /tmp/metadata/edan/chndm/00.txt
-rw-------  1 user  wheel   569577 Nov 21 11:31 /tmp/metadata/edan/chndm/01.txt
-rw-------  1 user  wheel   492463 Nov 21 11:31 /tmp/metadata/edan/chndm/02.txt
-rw-------  1 user  wheel   480150 Nov 21 11:31 /tmp/metadata/edan/chndm/03.txt
-rw-------  1 user  wheel   647755 Nov 21 11:31 /tmp/metadata/edan/chndm/04.txt
...
-rw-------  1 user  wheel   491210 Nov 21 11:31 /tmp/metadata/edan/chndm/fd.txt
-rw-------  1 user  wheel   622405 Nov 21 11:31 /tmp/metadata/edan/chndm/fe.txt
-rw-------  1 user  wheel   510866 Nov 21 11:31 /tmp/metadata/edan/chndm/ff.txt
```

Or:

```
$> ./bin/clone \
	-source-bucket-uri file:///tmp \
	-target-bucket-uri file:///tmp/debug \
	metadata/edan/chndm
	
...less time passes

$> ls -al /tmp/debug/metadata/edan/chndm/*.txt 
-rw-------  1 user  wheel   571870 Nov 21 11:31 /tmp/debug/metadata/edan/chndm/00.txt
-rw-------  1 user  wheel   569577 Nov 21 11:31 /tmp/debug/metadata/edan/chndm/01.txt
-rw-------  1 user  wheel   492463 Nov 21 11:31 /tmp/debug/metadata/edan/chndm/02.txt
-rw-------  1 user  wheel   480150 Nov 21 11:31 /tmp/debug/metadata/edan/chndm/03.txt
-rw-------  1 user  wheel   647755 Nov 21 11:31 /tmp/debug/metadata/edan/chndm/04.txt
...
-rw-------  1 user  wheel   491210 Nov 21 11:31 /tmp/debug/metadata/edan/chndm/fd.txt
-rw-------  1 user  wheel   622405 Nov 21 11:31 /tmp/debug/metadata/edan/chndm/fe.txt
-rw-------  1 user  wheel   510866 Nov 21 11:31 /tmp/debug/metadata/edan/chndm/ff.txt
```

And then later on:

```
$> ./bin/emit \
	-json \
	-format-json \
	-bucket-uri file:///tmp/metadata/edan \
	chndm

[{
  "id": "edanmdm-chndm_1931-45-37",
  "version": "",
  "unitCode": "CHNDM",
  "linkedId": "0",
  "type": "edanmdm",
  "content": {
    "descriptiveNonRepeating": {
      "record_ID": "chndm_1931-45-37",
      "online_media": {
        "mediaCount": 1,
  ...and so on
}]
```

#### Notes

* If no extra URI or URIs (for example `metadata/chndm`) are specified then the code will attempt to clone everything in the "source" bucket recursively.

* Under the hood this code is using the [GoCloud blob abstraction layer](https://gocloud.dev/howto/blob/). The default behaviour for the abstraction is to assume restrictive permissions when creating new files. Unfortunately, as of this writing, there is no common way for assigning permissions using the `GoCloud blob` abstraction so this is something you'll need to account for separately from this tool.

### emit

A command-line tool for parsing and emitting individual records from a directory containing compressed and line-delimited Smithsonian OpenAccess JSON files.

```
$> ./bin/emit -h
Usage:
  ./bin/emit [options] [path1 path2 ... pathN]

Options:
  -bucket-uri string
    	A valid GoCloud bucket URI. Valid schemes are: file://, s3:// and si:// which is signals that data should be retrieved from the Smithsonian's 'smithsonian-open-access' S3 bucket.
  -format-json
    	Format JSON output for each record.
  -json
    	Emit a JSON list.
  -null
    	Emit to /dev/null
  -oembed
    	Emit results as OEmbed records
  -query value
    	One or more {PATH}={REGEXP} parameters for filtering records.
  -query-mode string
    	Specify how query filtering should be evaluated. Valid modes are: ALL, ANY (default "ALL")
  -stats
    	Display timings and statistics.
  -stdout
    	Emit to STDOUT (default true)
  -validate-edan
    	Ensure each record is a valid EDAN document.
  -validate-json
    	Ensure each record is valid JSON.
  -workers int
    	The maximum number of concurrent workers. This is used to prevent filehandle exhaustion. (default 10)
```

For example, processing every record in the OpenAccess dataset ensuring it is valid JSON and emitting it to `/dev/null`:

```
> go run -mod vendor cmd/emit/main.go -bucket-uri file:///usr/local/OpenAccess \
  -stdout=false \
  -validate-json \  		
  -null \
  -stats \
  -workers 20 \
  metadata/objects

2020/06/26 10:19:17 Processed 11620642 records in 12m1.141284159s
```

Or processing everything in the [Air and Space](https://airandspace.si.edu/collections) collection as JSON, passing the result to the `jq` tool and searching for things with "space" in the title:

```
$> go run -mod vendor cmd/emit/main.go -bucket-uri file:///usr/local/OpenAccess \
   -json \
   -validate-json \  		   
   metadata/objects/NASM/ \
   | jq '.[]["title"]' \
   | grep -i 'space' \
   | sort

"Medal, NASA Space Flight, Sally Ride"
"Medal, STS-7, Smithsonian National Air and Space Museum, Sally Ride"
"Mirror, Primary Backup, Hubble Space Telescope"
"Model, 1:5, Hubble Space Telescope"
"Model, Space Shuttle, Delta-Wing High Cross-Range Orbiter Concept"
"Model, Space Shuttle, Final Orbiter Concept"
"Model, Space Shuttle, North American Rockwell Final Design, 1:15"
"Model, Space Shuttle, Straight-Wing Low Cross-Range Orbiter Concept"
"Model, Wind Tunnel, Convair Space Shuttle, 0.006 scale"
"Orbiter, Space Shuttle, OV-103, Discovery"
"Space Food, Beef and Vegetables, Mercury, Friendship 7"
"Spacecraft, Mariner 10, Flight Spare"
"Spacecraft, New Horizons, Mock-up, model"
"Suit, SpaceShipOne, Mike Melvill"
```

Or doing the same, but for [things about kittens](https://collection.cooperhewitt.org/objects/18382391/) in the [Cooper Hewitt](https://collection.cooperhewitt.org) collection:

```
$> go run -mod vendor cmd/emit/main.go -bucket-uri file:///usr/local/OpenAccess \
   -json \
   -validate-json \  		      
   -stats \
   metadata/objects/CHNDM/ \
   | jq '.[]["title"]' \
   | grep -i 'kitten' \
   | sort

2020/06/26 09:45:15 Processed 43695 records in 4.175884858s
"Cat and kitten"
"Tabby's Kittens"
```

Or something similar by not emitting a JSON list but formatting each record (as JSON) and filtering for the words "title" and "kitten":

```
$> go run -mod vendor cmd/emit/main.go -bucket-uri file:///usr/local/OpenAccess \
   -format-json \
   -validate-json=false \
   -stats \
   metadata/objects/CHNDM \
   | grep '"title"' \
   | grep -i 'kitten' \
   | sort
   
2020/06/26 10:02:59 Processed 43695 records in 5.045081835s
  "title": "Cat and kitten"
  "title": "Tabby\u0027s Kittens"
```

#### Inline queries

You can also specify inline queries by passing a `-query` parameter which is a string in the format of:

```
{PATH}={REGULAR EXPRESSION}
```

Paths follow the dot notation syntax used by the [tidwall/gjson](https://github.com/tidwall/gjson) package and regular expressions are any valid [Go language regular expression](https://golang.org/pkg/regexp/). Successful path lookups will be treated as a list of candidates and each candidate's string value will be tested against the regular expression's [MatchString](https://golang.org/pkg/regexp/#Regexp.MatchString) method.

For example:

```
$> go run -mod vendor cmd/emit/main.go -bucket-uri file:///usr/local/OpenAccess \
   -json \
   -query 'title=cats?\s+' \
   metadata/objects/CHNDM \
   | jq '.[]["title"]'
   
"View of Moat Mountain from Wildcat Brook, Jackson, New Hampshire, Looking Southwest"
"Near Falls of Wildcat Brook, Jackson, New Hampshire"
```

You can pass multiple `-query` parameters:

```
$> go run -mod vendor cmd/emit/main.go -bucket-uri file:///usr/local/OpenAccess \
   -json \
   -query 'title=cats?\s+' \
   -query 'title=(?i)^view' \
   metadata/objects/CHNDM \
   | jq '.[]["title"]'
   
"View of Moat Mountain from Wildcat Brook, Jackson, New Hampshire, Looking Southwest"
```

The default query mode is to ensure that all queries match but you can also specify that only one or more queries need to match by passing the `-query-mode ANY` flag:

```
$> go run -mod vendor cmd/emit/main.go -bucket-uri file:///usr/local/OpenAccess \
   -json \
   -query 'title=cats?\s+' \
   -query 'title=(?i)^view' \
   -query-mode ANY \
   metadata/objects/CHNDM \
   | jq '.[]["title"]'
   
"View of Santi Giovanni e Paolo a Celio, Rome"
"View of a Morning Room Interior"
"View of the Louvre from the River"
"View of the Acropolis, Athens"
"View of Santi Giovanni e Paolo a Celio, Rome"
"Views Representing the Most Considerable Transactions in the Siege of a Place, from Twelve of the Most REmarkable Sieges and Battles in Europe"
"View of Shiba Coast (Shibaura no fukei) From the Series One Hundred Famous views of Edo"
"View of Florence, Plate from \"Scelta di XXIV Vedute delle principali contrade, piazze, chiese, e palazzi della Città di Firenze\""
"View of Venice, Italy"
"View Across a River"
"View of the Canadian Falls and Goat Island"

...and so on
```

Did you know that there are 61 (out of 11 million) objects in the Smithsonian collection with the word "kitten" in their title?

```
$> go run -mod vendor cmd/emit/main.go -bucket-uri file:///usr/local/OpenAccess \
   -json \
   -query 'title=(?i)kitten' \
   -stats \
   -workers 50 \
   metadata/objects \
   | jq '.[]["title"]'
   
2020/06/26 18:22:04 Processed 62 records in 5m9.567738657s
"Cat and kitten"
"Tabby's Kittens"
"Ye Kitten (Number 17 May 1944)"
"The Kitten (No. 15 March 1944)"
"Three kittens on a stool"
"I'll Never Go Back On My Word; Leave My Kitten Alone"
"Untitled (Two Kittens)"
"Kitten Number Nine"
"Kitten No. Six"
"Bashful Baby Blues; Kitten On the Keys"
"Let's Make Believe We're Sweethearts; Three Naughty Kittens"
"Kittens playing"
"Take Me Back Again; Listen Kitten"
"Kittens"
"Diga Diga Do; Kitten WIth the Big Green Eyes, The"
"I Ain't Nothin' But a Tomcat's Kitten; I'm On My Way"
"Figurine, Kitten Small plastic"
"All the Time; Leave My Kitten Alone"
"Kitten On the Keys; That Place Down the Road Apiece"
"One Dime Blues; Three Little Kittens Rag"
"Kitten No. Eleven"
"I Ain't Nothin' But a Tomcat's Kitten; I'm On My Way"
"Weaker Kitten No. 2/64/41"
"Reward of Merit with Boy and Girl Playing with Cat and Kittens"
"Two Dollar Rag; Kitten on the Keys"
"The Kitten (No. 13 January 1944)"
"Kittens Playing with Camera"
"Reward of Merit with Two White Kittens in Basket"
"Doug and Toad - Kitten on Stump, 1942"
"I'll Never Go Back On My Word; Leave My Kitten Alone"
"The Kitten (No. 40 Sept. 1953)"
"Diga Diga Do; Kitten With the Big Green Eyes, The"
"The Kitten's Breakfast"
"Bunch Of Keys, A; Kitten On the Keys"
"The Kitten (No. 14 February 1944)"
"Figurine, Siamese Kitten"
"Tom Kitten"
"Young boy and his kitten"
"The Young Kittens"
"Little Kittens Learning Abc"
"Jump Jump of Holiday House. Three Little Kittens."
"The Kitten (Number 25 November 1946)"
"Kitten in Shoe"
"The Color Kittens"
"All the Time; Leave My Kitten Alone"
"Kitten on a Stool"
"Super Kitten; We'd Better Stop"
"Kitten mitten roller derby button"
"Little Girl holding Kitten"
"The Kitten (No. 39 May 27, 1953)"
"The Kitten (Number 18 June 1944)"
"My Love Is a Kitten; Strange Little Melody, The"
"Live and Let Live; Tom Cat's Kitten"
"Live and Let Live; Tom Cat's Kitten"
"One Dime Blues; Three Little Kittens Rag"
"The Kitten (No. 47 Dec. 1954)"
"Kittens Playing with Camera"
"\"Okimono\" Figure Of A Cat And Three Kittens"
"Mummy Of \"Kitten\""
"Plicate Kitten's Paw"
"Atlantic Kitten's Paw"
"Drosophila arawakana kittensis"
```

#### OEmbed

It is also possible to emit OpenAccess records as [OEmbed](https://oembed.com/) documents of type "photo". An OEmbed record will be created for each media object of type "Screen Image" or "Images" associated with an OpenAccess record. OpenAccess records that do not have an suitable media objects will be excluded.

For example:

```
$> go run -mod vendor cmd/emit/main.go -bucket-uri file:///usr/local/OpenAccess \
   -json \
   -oembed \
   metadata/objects/NASM \
   | jq
   
[
{
  "version": "1.0",
  "type": "photo",
  "width": -1,
  "height": -1,
  "title": "Clerget 9 A Diesel, Radial 9 Engine (Gift of the Musee de L' Air)",
  "url": "https://ids.si.edu/ids/download?id=NASM-A19721334000-NASM2016-04025_screen",
  "author_name": "Clerget, Blin and Cie",
  "author_url": "https://airandspace.si.edu/collection/id/nasm_A19721334000",
  "provider_name": "National Air and Space Museum",
  "provider_url": "https://airandspace.si.edu",
  "object_uri": "si://nasm/o/A19721334000"
},
{
  "version": "1.0",
  "type": "photo",
  "width": -1,
  "height": -1,
  "title": "Swagger Stick, Royal Flying Corps (Gift of Eloise and John Charlton)",
  "url": "https://ids.si.edu/ids/download?id=NASM-A19830196000_PS01_screen",
  "author_name": "Lt. Wes D. Archer",
  "author_url": "https://airandspace.si.edu/collection/id/nasm_A19830196000",
  "provider_name": "National Air and Space Museum",
  "provider_url": "https://airandspace.si.edu",
  "object_uri": "si://nasm/o/A19830196000"
}
  ... and so on
```

* The OEmbed record `title` property will be constructed in the form of "{OBJECT TITLE} ({OBJECT CREDIT LINE})".

* The OEmbed record `author_name` property will be constructed using the OpenAccess record's `content.freetext.name` or `content.freetext.manufacturer` properties, in that order. If neither are present the `author_name` property will be constructed in the form of "Collection of {SMITHSONIAN UNIT NAME}".

* The OEmbed record `author_url` property will that object's URL on the web.

* The OEmbed record `width` and `height` properties are both set to "-1" to indicate that image dimensions are [not available at this time](https://github.com/Smithsonian/OpenAccess/issues/2).

* The OEmbed record will contain a non-standard `object_uri` string that is compatiable with [RFC6570 URI Templates](https://tools.ietf.org/html/rfc6570). It is constructed in the form of `si://{SMITHSONIAN_UNIT}/o/{NORMALIZAED_EDAN_OBJECT_ID}`. The `object_uri` property should still be considered experimental. It may change or be removed in future releases.

* `{NORMALIZAED_EDAN_OBJECT_ID}` strings are derived from the OpenAccess `id` property. The normalization rules are: Remove the leading `edanmdm-{SMITHSONIAN_UNIT}_` prefix and replace all instances of the `.` character the with a `_` character. For example the string `edanmdm-nmaahc_2017.30.9` will be normalized as `2017_30_9`.

### findingaid

A command-line tool for emitting a CSV document mapping individual record identifiers to their corresponding OpenAccess JSON file and line number, produced from a directory containing compressed and line-delimited Smithsonian OpenAccess JSON files.

```
$> ./bin/findingaid -h
Usage:
  ./bin/findingaid [options] [path1 path2 ... pathN]

Options:
  -bucket-uri string
    	A valid GoCloud bucket URI. Valid schemes are: file://, s3:// and si:// which is signals that data should be retrieved from the Smithsonian's 'smithsonian-open-access' S3 bucket.
  -csv-header
    	Include a CSV header row in the output (default true)
  -include-all
    	Include all OpenAccess identifiers
  -include-guid content.descriptiveNonRepeating.guid
    	Include the OpenAccess content.descriptiveNonRepeating.guid identifier
  -include-openaccess-id id
    	Include the OpenAccess id identifier
  -include-record-id content.descriptiveNonRepeating.record_ID
    	Include the OpenAccess content.descriptiveNonRepeating.record_ID identifier (default true)
  -include-record-link content.descriptiveNonRepeating.record_link
    	Include the OpenAccess content.descriptiveNonRepeating.record_link identifier
  -null
    	Emit to /dev/null
  -query value
    	One or more {PATH}={REGEXP} parameters for filtering records.
  -query-mode string
    	Specify how query filtering should be evaluated. Valid modes are: ALL, ANY (default "ALL")
  -stats
    	Display timings and statistics.
  -stdout
    	Emit to STDOUT (default true)
  -workers int
    	The maximum number of concurrent workers. This is used to prevent filehandle exhaustion. (default 10)
```

For example:

```
$> go run -mod vendor cmd/findingaid/main.go -bucket-uri file:///usr/local/OpenAccess \
	metadata/objects/SAAM 

id,path,line_number
saam_1971.439.94,metadata/objects/SAAM/00.txt.bz2,1
saam_1971.439.92,metadata/objects/SAAM/08.txt.bz2,1
saam_1915.5.1,metadata/objects/SAAM/00.txt.bz2,2
saam_1971.439.78,metadata/objects/SAAM/03.txt.bz2,1
saam_XX32,metadata/objects/SAAM/12.txt.bz2,1
saam_1983.90.173,metadata/objects/SAAM/00.txt.bz2,3
saam_1970.335.1,metadata/objects/SAAM/03.txt.bz2,2
saam_1971.439.97,metadata/objects/SAAM/0d.txt.bz2,1
saam_1968.155.158,metadata/objects/SAAM/12.txt.bz2,2
saam_1967.14.149,metadata/objects/SAAM/08.txt.bz2,2
saam_1979.98.188,metadata/objects/SAAM/02.txt.bz2,1
saam_1985.66.295_540,metadata/objects/SAAM/00.txt.bz2,4
saam_1970.334,metadata/objects/SAAM/03.txt.bz2,3
saam_1968.19.12,metadata/objects/SAAM/0d.txt.bz2,2
... and so on
```

By default only the `OpenAccess content.descriptiveNonRepeating.record_ID` identifier is included in the finding aid. You can include other identifiers with their corresponding command-line flag or enable include all identifiers by passing the `-include-all` flag. For example:

```
$> go run -mod vendor cmd/findingaid/main.go -bucket-uri file:///usr/local/OpenAccess \
   -include-all \
   metadata/objects/NMAAHC
   
id,path,line_number
http://n2t.net/ark:/65665/fd53f870fc2-73af-4c50-b1c5-a3fd2829ad1f,metadata/objects/NMAAHC/ff.txt.bz2,1
nmaahc_2014.72.2,metadata/objects/NMAAHC/ff.txt.bz2,1
https://nmaahc.si.edu/object/nmaahc_2014.72.2,metadata/objects/NMAAHC/ff.txt.bz2,1
edanmdm-nmaahc_2014.72.2,metadata/objects/NMAAHC/ff.txt.bz2,1
http://n2t.net/ark:/65665/fd5343a21ed-73d9-4014-a34c-b175b84168c8,metadata/objects/NMAAHC/21.txt.bz2,1
nmaahc_2014.75.130,metadata/objects/NMAAHC/21.txt.bz2,1
https://nmaahc.si.edu/object/nmaahc_2014.75.130,metadata/objects/NMAAHC/21.txt.bz2,1
edanmdm-nmaahc_2014.75.130,metadata/objects/NMAAHC/21.txt.bz2,1
http://n2t.net/ark:/65665/fd59212a6e2-b745-4eb9-84ad-4368ffea8223,metadata/objects/NMAAHC/17.txt.bz2,1
nmaahc_2016.140.1.3,metadata/objects/NMAAHC/17.txt.bz2,1
https://nmaahc.si.edu/object/nmaahc_2016.140.1.3,metadata/objects/NMAAHC/17.txt.bz2,1
edanmdm-nmaahc_2016.140.1.3,metadata/objects/NMAAHC/17.txt.bz2,1
http://n2t.net/ark:/65665/fd599a84051-37d5-49d4-98d3-9052e5cbcea9,metadata/objects/NMAAHC/22.txt.bz2,1
nmaahc_2012.30.3,metadata/objects/NMAAHC/22.txt.bz2,1
https://nmaahc.si.edu/object/nmaahc_2012.30.3,metadata/objects/NMAAHC/22.txt.bz2,1
edanmdm-nmaahc_2012.30.3,metadata/objects/NMAAHC/22.txt.bz2,1
http://n2t.net/ark:/65665/fd53a114ad8-2cc2-4ce2-bbd0-6dd09cc715df,metadata/objects/NMAAHC/0c.txt.bz2,1
nmaahc_2013.133.1.4,metadata/objects/NMAAHC/0c.txt.bz2,1
https://nmaahc.si.edu/object/nmaahc_2013.133.1.4,metadata/objects/NMAAHC/0c.txt.bz2,1
edanmdm-nmaahc_2013.133.1.4,metadata/objects/NMAAHC/0c.txt.bz2,1
http://n2t.net/ark:/65665/fd5d302d893-ae7c-4b4d-93bb-59f87237d23a,metadata/objects/NMAAHC/1c.txt.bz2,1
nmaahc_2014.222.2,metadata/objects/NMAAHC/1c.txt.bz2,1
https://nmaahc.si.edu/object/nmaahc_2014.222.2,metadata/objects/NMAAHC/1c.txt.bz2,1
edanmdm-nmaahc_2014.222.2,metadata/objects/NMAAHC/1c.txt.bz2,1
http://n2t.net/ark:/65665/fd5ab09d12b-42bc-40f2-9557-b924d182723e,metadata/objects/NMAAHC/ff.txt.bz2,2
nmaahc_2016.166.17,metadata/objects/NMAAHC/ff.txt.bz2,2
https://nmaahc.si.edu/object/nmaahc_2016.166.17,metadata/objects/NMAAHC/ff.txt.bz2,2
edanmdm-nmaahc_2016.166.17,metadata/objects/NMAAHC/ff.txt.bz2,2
http://n2t.net/ark:/65665/fd588400c0f-66c3-4259-999d-57f112e05479,metadata/objects/NMAAHC/2b.txt.bz2,1
nmaahc_2014.263.5,metadata/objects/NMAAHC/2b.txt.bz2,1
https://nmaahc.si.edu/object/nmaahc_2014.263.5,metadata/objects/NMAAHC/2b.txt.bz2,1
... and so on
```

The `findingaid` tool also supports inline queries (described above). For example there are 4044 records with the word "panda" in their title:

```
go run -mod vendor cmd/findingaid/main.go -bucket-uri file:///usr/local/OpenAccess \
   -query 'title=(?i)pandas?' \
   -workers 50 \
   metadata/objects/ \
   > pandas.csv

time passes...

$> wc -l pandas.csv
    4044 pandas.csv

$> less pandas.csv
id,path,line_number
nmah_1333041,metadata/objects/NMAH/17.txt.bz2,75
nmah_1195220,metadata/objects/NMAH/1f.txt.bz2,520
nmah_1065733,metadata/objects/NMAH/32.txt.bz2,393
nmah_414524,metadata/objects/NMAH/43.txt.bz2,3302
nmah_1298355,metadata/objects/NMAH/2a.txt.bz2,4794
nmah_1333042,metadata/objects/NMAH/69.txt.bz2,4331
nmah_903687,metadata/objects/NMAH/71.txt.bz2,3133
nmah_1465552,metadata/objects/NMAH/d1.txt.bz2,137
nmah_1449233,metadata/objects/NMAH/aa.txt.bz2,4518
nmah_334375,metadata/objects/NMAH/bd.txt.bz2,2143
nmah_414787,metadata/objects/NMAH/cf.txt.bz2,2140
nmnhanthropology_8357155,metadata/objects/NMNHANTHRO/27.txt.bz2,785
nmnhanthropology_8394769,metadata/objects/NMNHANTHRO/03.txt.bz2,1232
nmnhanthropology_8426012,metadata/objects/NMNHANTHRO/04.txt.bz2,1441
nmnhanthropology_8413868,metadata/objects/NMNHANTHRO/0a.txt.bz2,1447
... and so on
```

### location

A command-line tool for parsing line-delimited Smithsonian OpenAccess JSON files and emiting place data as a stream of CSV records.

```
$> ./bin/location -h
Usage:
  ./bin/location [options]

Options:
  -null
    	Emit to /dev/null
  -stdout
    	Emit to STDOUT (default true)
```

For example:

```
$> ./bin/emit \
	-bucket-uri file:///usr/local/OpenAccess metadata/objects/NMAH \

   | ./bin/location

edanmdm-nmah_715051,content.freetext.place,place made,"United States: New York, New York City"
edanmdm-nmah_580165,content.freetext.place,place made,United States
edanmdm-nmah_598790,content.freetext.place,place made,"United Kingdom: England, Longport"
edanmdm-nmah_580114,content.freetext.place,place made,United States: New Jersey
edanmdm-nmah_670543,content.freetext.place,place made,United States
edanmdm-nmah_570097,content.freetext.place,place made,United Kingdom: England
edanmdm-nmah_415366,content.freetext.place,place made,Germany
...and so on
edanmdm-nmah_383309,content.freetext.place,associated place,United States
edanmdm-nmah_1957071,content.freetext.place,place made,Russia
edanmdm-nmah_1957077,content.freetext.place,place made,Russia
edanmdm-nmah_1957190,content.freetext.place,place made,Russia
edanmdm-nmah_1408250,content.freetext.place,place made,"United States: District of Columbia, Washington"
edanmdm-nmah_1321602,content.freetext.place,place made,"France: Île-de-France, Paris"
```

The column in the CSV output are:

| Index | Value | Example |
| --- | --- | --- |
| 0 | OpenAccess record ID | edanmdm-nmah_715051 |
| 1 | Path to the property used to lookup place data | content.freetext.place |
| 2 | Label associated with place data | place made |
| 3 | Place name | "United States: New York, New York City" |

### placename

A command-line tool for extracting only placename data from a CSV stream produced by the `location` tool.

```
$> ./bin/placename -h
Usage:
  ./bin/placename [options]

Options:
  -unique
    	Only unique emit placename strings once. (default true)
```

For example:

```
$> ./bin/emit -bucket-uri file:///usr/local/OpenAccess metadata/objects/NMAH \

   | ./bin/location \
   
   | ./bin/placename \
   
   | wc -l
   
12164
```

Or:

```
$> ./bin/emit -bucket-uri file:///usr/local/OpenAccess metadata/objects/ \

   | ./bin/location \
   
   | ./bin/placename

United States
Washington (D.C.)
Florida
Miami (Fla.)
France
USA
Italy
Europe
Italy or Spain
France, Europe
probably Venice, Italy
Milan, Italy
France or Italy
Florence, Italy

...time passes

From top of pass to Hoja Verde., Tamaulipas, Mexico, North America
Perto Dom Pedro II, Paraná, Brazil, South America
Zealand: peat-bog at Søgärd., Denmark, Europe
Tarumã Alta, 14 km NW of Manaus., Manaus, Brazil, South America - Neotropics
Woods near Taxodium swamp, 2 miles south of Eagletown, McCurtain Co., Oklahoma, United States, North America
Yarmouth County. Deep water of St. John (Wilson's) Lake., Nova Scotia, Canada, North America
½ mi. S. Olivet., Osage, Kansas, United States, North America
Range of low hills ca. 20 km west of Redenção, near Córrego São João and Troncamento Santa Teresa, Conceição do Araguaia, Brazil, South America - Neotropics
Tatama. Santa Cecilia. Cordillera Occidental. Vertiente Occidental, Caldas, Colombia, South America - Neotropics
San Rafael Ranch - on banky rivers. Cameron Co, Texas, United States, North America
Sultanabad, Khorassan., Khorasan [obsolete], Iran, Asia-Temperate
Pointe Du Lac, comte du St-Maurice: sur les sables du lac St-Pierre., Quebec, Canada, North America

...2.5M records later

Limburg
Japão
Lado Enclave (Congo Free State)
Mauritanie
Igboho (Nigeria)
Accra Plains
Hollywood (Fla.)
Broward County (Fla.)
GrÃ-Bretanha
América latina
Cousin
Peace River Watershed (B.C. and Alta.)
Peace River Watershed
```

## See also

* https://github.com/Smithsonian/OpenAccess
* https://gocloud.dev/howto/blob/
* https://github.com/aaronland/go-jsonl/
* https://github.com/aaronland/go-json-query/
