## Project 3 - OpenStreetMap Data Wrangling
##### Author: Nikolas Thorun

This project is based on the data wrangling from a map exported from OpenStreetMap and consists of four phases: Download and data investigation, iterative data wrangling, insertion of treated data in the database and database queries.
The map chosen was that of the city of Southampton, England, to avoid the obstacles of trying to work with UNICODE, since city maps in Brazil have special characters such as _á, ã, ç_ and etc., which are not supported by the standard Python 2 * encoding * The map file is just over 65 MB unzipped. <br/>
The map of Southampton can be accessed at: https://www.openstreetmap.org/node/2478628079#map=14/50.9025/-1.4042 <br/>
The database to be used is MongoDB.


```python
#import necessary libraries
import xml.etree.cElementTree as ET
import pprint
import re
import codecs
import json

#map file location
filename = 'D:/Udacity/P3/map/southampton_england.osm'
```

Using part of the code generously given by Shannon Bradshaw, here is the first scan of the map to identify the most common types of roads. Once identified, the types with the highest incidence are manually entered in the *expected* list. All types of roads that are not included in the list are printed below.


```python
from collections import defaultdict

street_type_re = re.compile(r'\b\S+\.?$', re.IGNORECASE)

expected = ["Avenue", "Bridge", "Buildings", "Centre", "Close", "Court", "Crescent", "Drive", "Gardens", 
            "Grove", "Hill", "Lane", "Mews", "Place", "Road", "Square", "Street", "Terrace", "Walk", "Way"]

def audit_street_type(street_types, street_name):
    m = street_type_re.search(street_name)
    if m:
        street_type = m.group()
        if street_type not in expected:
            street_types[street_type].add(street_name)


def is_street_name(elem):
    return (elem.attrib['k'] == "addr:street")


def audit(osmfile):
    osm_file = codecs.open(osmfile, "r", encoding="utf8")
    street_types = defaultdict(set)
    for event, elem in ET.iterparse(osm_file, events=("start",)):

        if elem.tag == "node" or elem.tag == "way":
            for tag in elem.iter("tag"):
                if is_street_name(tag):
                    audit_street_type(street_types, tag.attrib['v'])
    osm_file.close()
    return street_types


st_types = audit(filename)
pprint.pprint(dict(st_types))
```

    {'387': {'387'},
     'Broadway': {'The Broadway', 'Midanbury Broadway'},
     'Cloisters': {'The Cloisters'},
     'Cottages': {'Honeysuckle Cottages'},
     'Dell': {'Holly Dell'},
     'Drove': {'Coxford Drove', 'York Drove'},
     'East': {'Millbrook Road East', 'Bitterne Road East', 'Bassett Crescent East'},
     'Esplanade': {'Western Esplanade'},
     'Estate': {'Belgrave Industrial Estate'},
     'Finches': {'The Finches'},
     'Firs': {'The Firs'},
     'Green': {'Lulworth Green', 'Chiltern Green', 'Edwin Jones Green'},
     'Greenways': {'Greenways'},
     'High-Rise': {'Ferry High-Rise'},
     'Holt': {'Aspen Holt'},
     'House': {'Beech House'},
     'Loop': {'Spitfire Loop'},
     'Mayflowers': {'The Mayflowers'},
     'Meadow': {'Bassett Meadow'},
     'Mount': {'The Mount'},
     'North': {'Manor Road North', 'Osborne Road North'},
     'Parade': {'Harbour Parade', 'Marine Parade'},
     'Park': {'Antelope Park', 'Portswood Park'},
     'Polygon': {'The Polygon'},
     'Precinct': {'Shirley Precinct'},
     'Quay': {'Shamrock Quay', 'Town Quay'},
     'Queensway': {'Queensway'},
     'Raod': {'Bluebell Raod'},
     'Rd': {'Grange Rd', 'Hythe Rd', 'Bellevue Rd'},
     'Redhill': {'Redhill'},
     'Rise': {'Overcliff Rise'},
     'S': {'S'},
     'Saltmead': {'Saltmead'},
     'South': {'Osborne Road South', 'Albert Road South'},
     'Street)': {'Western Esplanade (corner of Fitzhugh Street)'},
     'View': {'Pond View', 'Park View'},
     'Village': {'Bassett Green Road / Bassett Green Village', 'Bitterne Village'},
     'West': {'Bassett Crescent West', 'Millbrook Road West', 'Bitterne Road West'},
     'Westal': {'Bitterne Road Westal'},
     'access': {'Emergency department access'},
     're': {'Royal Crescent Road student re'},
     'road': {'bluebell road', 'Bluebell road'}}
    

Here we realize the incidence of some wrong names, like 'road' and 'Raod' and others that types that are just not so common. There is also the incidence of cases such as 'Greenways', 'Saltmead' and 'Redhill', which do not have types, but can unfold in ways of the same name and of another type, as shown in the figure below.
![Redhill](Images/redhill.png?raw=true "Redhill")

In addition, names like 'Royal Crescent Road student re' and 'bluebell road' need updates not just in the type of way, so they have been added to an update list made just for them.
In the function below, problematic ways and types of problematic ways are identified and have their names modified from the *mapping* and *road_mapping* dictionaries. At the end, the names are shown before the update and after the update.


```python
def update_name(name, mapping):
    
    m = street_type_re.search(name)
    
    if name in hand_update:
        name = name.replace(name, road_mapping[name])
    elif m:
        street_type = m.group()
        if street_type in mapping:
            name = name.replace(street_type, mapping[street_type])
  
    return name

mapping = { "Raod" : "Road",
            "Rd" : "Road",
            "road" : "Road",
            "Westal" : "West"
            }

road_mapping = { "Royal Crescent Road student re" : "Royal Crescent Road",
                 "Emergency department access" : "Emergency Department Access",
                 "Western Esplanade (corner of Fitzhugh Street)" : "Western Esplanade",
                 "bluebell road" : "Bluebell Road"
                }

hand_update = ["Royal Crescent Road student re", "Emergency department access", 
               "Western Esplanade (corner of Fitzhugh Street)", "bluebell road"]

for st_type, ways in st_types.items():
        for name in ways:
            better_name = update_name(name, mapping)
            print(name, "=>", better_name)
            
```

    Osborne Road South => Osborne Road South
    Albert Road South => Albert Road South
    Bassett Crescent West => Bassett Crescent West
    Millbrook Road West => Millbrook Road West
    Bitterne Road West => Bitterne Road West
    Millbrook Road East => Millbrook Road East
    Bitterne Road East => Bitterne Road East
    Bassett Crescent East => Bassett Crescent East
    Bassett Green Road / Bassett Green Village => Bassett Green Road / Bassett Green Village
    Bitterne Village => Bitterne Village
    S => S
    Shamrock Quay => Shamrock Quay
    Town Quay => Town Quay
    Manor Road North => Manor Road North
    Osborne Road North => Osborne Road North
    Shirley Precinct => Shirley Precinct
    The Polygon => The Polygon
    Grange Rd => Grange Road
    Hythe Rd => Hythe Road
    Bellevue Rd => Bellevue Road
    Western Esplanade => Western Esplanade
    Royal Crescent Road student re => Royal Crescent Road
    Ferry High-Rise => Ferry High-Rise
    Queensway => Queensway
    Pond View => Pond View
    Park View => Park View
    Lulworth Green => Lulworth Green
    Chiltern Green => Chiltern Green
    Edwin Jones Green => Edwin Jones Green
    Bitterne Road Westal => Bitterne Road West
    Antelope Park => Antelope Park
    Portswood Park => Portswood Park
    Emergency department access => Emergency Department Access
    Western Esplanade (corner of Fitzhugh Street) => Western Esplanade
    The Broadway => The Broadway
    Midanbury Broadway => Midanbury Broadway
    Harbour Parade => Harbour Parade
    Marine Parade => Marine Parade
    The Finches => The Finches
    Saltmead => Saltmead
    Bluebell Raod => Bluebell Road
    bluebell road => Bluebell Road
    Bluebell road => Bluebell Road
    Honeysuckle Cottages => Honeysuckle Cottages
    The Cloisters => The Cloisters
    The Mayflowers => The Mayflowers
    Aspen Holt => Aspen Holt
    Coxford Drove => Coxford Drove
    York Drove => York Drove
    Greenways => Greenways
    Holly Dell => Holly Dell
    The Firs => The Firs
    Bassett Meadow => Bassett Meadow
    Beech House => Beech House
    Belgrave Industrial Estate => Belgrave Industrial Estate
    387 => 387
    The Mount => The Mount
    Spitfire Loop => Spitfire Loop
    Redhill => Redhill
    Overcliff Rise => Overcliff Rise
    

We can see that the types of unusual ways have not changed and the ways with errors have been corrected, with the exception of the 'S' and '387' routes, which I believe are data entry errors, as there aren't any ways with these names in the city .

The following code transforms the XML file data from the map file into JSON type, to be inserted in MongoDB. In this phase the updated names are inserted in the variable *data*, which will then be read by the database.


```python
lower = re.compile(r'^([a-z]|_)*$')
lower_colon = re.compile(r'^([a-z]|_)*:([a-z]|_)*$')
problemchars = re.compile(r'[=\+/&<>;\'"\?%#$@\,\. \t\r\n]')

CREATED = [ "version", "changeset", "timestamp", "user", "uid"]


def shape_element(element):
    created = {}
    node = {}
    nodes = []
    adress = {}
    posi = []
    if element.tag == "node" or element.tag == "way" :
        node['id'] = element.attrib['id']
        node['visible'] = "true"
        node['type'] = element.tag
        if element.tag == "node" :
            node['pos'] = [float(element.attrib['lat']), float(element.attrib['lon'])]
        created['changeset'] = element.attrib['changeset']
        created['user'] = element.attrib['user']
        created['version'] = element.attrib['version']
        created['uid'] = element.attrib['uid']
        created['timestamp'] = element.attrib['timestamp']
        node['created'] = created
        for tag in element.iter("tag"):
            if tag.attrib['k'] == "addr:housenumber":
                adress['housenumber'] = tag.attrib['v']
            elif  tag.attrib['k'] == "addr:street":
                m = street_type_re.search(tag.attrib['v'])
                if m.group() not in expected:
                    updated_name = update_name(tag.attrib['v'], mapping)
                    adress['street'] = updated_name
                else:
                    adress['street'] = tag.attrib['v'] 
        for nd in element.iter("nd"):
            if nd.attrib['ref']:
                nodes.append(nd.attrib['ref'])
            node['address'] = adress
            node['node_refs'] = nodes
        
        return node
    else:
        return None


def process_map(file_in, pretty = False):
    file_out = "D:/Udacity/P3/map/southampton_england.json".format(file_in)
    data = []
    with codecs.open(file_out, "w") as fo:
        for _, element in ET.iterparse(file_in):
            el = shape_element(element)
            if el:
                data.append(el)
                if pretty:
                    fo.write(json.dumps(el, indent=2)+"\n")
                else:
                    fo.write(json.dumps(el) + "\n")
    return data

data = process_map(filename, False)
```


```python
#imports 'data' variable to the database
from pymongo import MongoClient
client = MongoClient("mongodb://localhost:27017")
db = client.cities
db.southampton.drop()
db.southampton.insert_many(data)
```




    <pymongo.results.InsertManyResult at 0x286bdb3e408>



With the data in the database, we can start querying. Below, I listed some of the questions I would like answers to:
* How many documents were inserted in the database? <br/>
* How many of these documents have a way? <br/>
* Has the data of type 'node' and 'way' been entered correctly? <br/>
* Who are the 10 users who contributed most to this map? How many contributions did they make? <br/>
* Which 5 streets have the highest incidence on the map? How often do they appear?

The following query shows how many documents have been inserted into the database.


```python
print("Documents inserted into the database:")
pprint.pprint(db.southampton.count_documents({}))
```

    Documents inserted into the database:
    324806
    

The queries below show how many documents are of the type *way*, *node* and how many documents of the type *way* have the street name.


```python
print("Documents type node:")
pprint.pprint(db.southampton.count_documents({"type" : "node"}))
```

    Documents type node:
    273331
    


```python
print("Documents type way:")
pprint.pprint(db.southampton.count_documents({"type" : "way"}))
```

    Documents type way:
    51475
    


```python
print("Documents type way with street name:")
pprint.pprint(db.southampton.count_documents({"address.street" : {"$exists" : 1}}))
```

    Documents type way with street name:
    22781
    

The queries below show examples of documents like *node* and *way*, proving that they were inserted correctly.
Here, the **pprint** library is important to facilitate data visualization.


```python
pprint.pprint(db.southampton.find_one({"type" : "node"}))
```

    {'_id': ObjectId('5e88cc6f075e7d49b61d5bdd'),
     'created': {'changeset': '8139974',
                 'timestamp': '2011-05-14T11:45:29Z',
                 'uid': '260682',
                 'user': 'monxton',
                 'version': '5'},
     'id': '132707',
     'pos': [50.9454657, -1.4775675],
     'type': 'node',
     'visible': 'true'}
    


```python
pprint.pprint(db.southampton.find_one({"type" : "way", "address.street" : {"$exists" : 1}}))
```

    {'_id': ObjectId('5e88cc71075e7d49b6219185'),
     'address': {'housenumber': 'Berth 106', 'street': 'Herbert Walker Avenue'},
     'created': {'changeset': '35345080',
                 'timestamp': '2015-11-16T09:22:22Z',
                 'uid': '1569426',
                 'user': 'Harjit (CabMyRide)',
                 'version': '9'},
     'id': '10517683',
     'node_refs': ['90588898', '90588899', '90588900', '90588901', '90588898'],
     'type': 'way',
     'visible': 'true'}
    

The query below shows the ten users who contributed most to this part of the map and the number of contributions by each.


```python
resultado = db.southampton.aggregate([ 
                                     {"$group" : { "_id" : "$created.user", "count" : {"$sum" : 1}}},
                                     {"$sort" : {"count" : -1}},{"$limit": 10} 
                                     ])

for document in resultado:
    pprint.pprint(document)
```

    {'_id': 'Chris Baines', 'count': 107158}
    {'_id': 'Harjit (CabMyRide)', 'count': 24971}
    {'_id': '0123456789', 'count': 23764}
    {'_id': 'Nick Austin', 'count': 17681}
    {'_id': 'pcman1985', 'count': 14158}
    {'_id': 'Deanna Earley', 'count': 13438}
    {'_id': 'Arjan Sahota', 'count': 12948}
    {'_id': 'Kuldip (CabMyRide)', 'count': 9619}
    {'_id': 'Andy Street', 'count': 9189}
    {'_id': 'Harry Cutts', 'count': 6676}
    

The query below shows the five routes with the highest incidence on the map and 'None', since most documents do not have the variable 'address.street' filled in.


```python
resultado = db.southampton.aggregate([ 
                                     {"$group" : { "_id" : "$address.street", "count" : {"$sum" : 1}}},
                                     {"$sort" : {"count" : -1}},{"$limit": 6} 
                                     ])

for document in resultado:
    pprint.pprint(document)
```

    {'_id': None, 'count': 302025}
    {'_id': 'Burgess Road', 'count': 371}
    {'_id': 'Portswood Road', 'count': 349}
    {'_id': 'Winchester Road', 'count': 319}
    {'_id': 'Honeysuckle Road', 'count': 272}
    {'_id': 'Hill Lane', 'count': 250}
    

#### Bibliography
* Python documentation and tutorials: https://docs.python.org/2/
* MongoDB documentation and tutorials: https://docs.mongodb.com
* RegEx help: http://regexr.com
