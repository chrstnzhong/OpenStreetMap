
# OpenStreetMap Data Case Study Writeup

## Map Area  
San Francisco, CA, USA  
I currently live around this area and am interested to see how others have mapped it out and how I can improve the map.  
Link to map area: http://www.openstreetmap.org/export#map=14/37.7921/-122.4212

## Auditing the Data  
When auditing the data, I encountered a few inconsistencies with the data:  
1. Street Names  
    * "Street" had many different variations which included both capitalized letters and abbreviations (i.e. St., St, st., street, Street) while "boulevard" and "avenue" were abbreviated in some cases.
2. Postal Codes
    * There were a few postal codes that were written in a different format (e.g. "CA:94103", "94103-3124", "CA 94108").
    * All SF postal codes start with '941' and are within the range of 94101 to 94199. A few postal codes did not fall within this range, with the most popular one being 90214, which is an LA postal code.
3. State Name
    * There were a few variations on how to include the state name, including unabbreviated  and lowercase letters (e.g. "ca", "California"). The majority of the state name was in the abbreviated format "CA".
4. City Name
    * For the most part the city name was consistent throughout the data, except for one instance in which it was misspelled ("San Francicsco") and one with lowercase ("san Francisco").
    
#### Auditing Street Names
I used the following functions to audit the street names. I wanted to find any variations that were not in the expected list of street types.
```Python
street_expected = ["Street", "Avenue", "Boulevard", "Drive", "Court", "Place", "Square", "Lane", "Road", 
            "Trail", "Parkway", "Commons"]

def audit_street_type(street_types, street_name):
    m = street_type_re.search(street_name)
    if m:
        street_type = m.group()
        if street_type not in street_expected:
            street_types[street_type].add(street_name)

def is_street_name(elem):
    return (elem.attrib['k'] == "addr:street")

def audit_street_name(osmfile):
    osm_file = open(osmfile, "r")
    street_types = defaultdict(set)
    for event, elem in ET.iterparse(osm_file, events=("start",)):
        if elem.tag == "node" or elem.tag == "way":
            for tag in elem.iter("tag"):
                if is_street_name(tag):
                    audit_street_type(street_types, tag.attrib['v'])
    osm_file.close()
    return street_types
```
#### Auditing Postal Codes
For postal codes I wanted to find any postal codes that were not within the SF postal code range (94101-94199) as well as any longer variations on how the postal code was written.
```Python
postal_code_expected = str(range(94101,94200))

def audit_postal_code(osmfile):
    postal_codes = []
    for event, elem in ET.iterparse(osmfile):
        if elem.tag == "tag":
            if elem.attrib['k'] == "addr:postcode":
                if elem.attrib['v'] not in postal_code_expected:
                    postal_codes.append([elem.attrib['v']])
    return postal_codes
```
#### Auditing State Name
I wanted to see all variations of how the state name was written in order to find the most common variation. "CA" was the most common.
```Python
def audit_state_name(osmfile):
    state_names = {}
    for event, elem in ET.iterparse(osmfile):
        if elem.tag == "tag":
            if elem.attrib['k'] == "addr:state":
                if elem.attrib['v'] not in state_names:
                    state_names[elem.attrib['v']] = 1
                else:
                    state_names[elem.attrib['v']] += 1
    return state_names
```

#### Auditing City Name
Similar to how I audited the state name, I wanted to see all ways in which the city name was included to find the most common way the city name was written (i.e. "San Francisco" vs "SF"). "San Francisco" was the most common way.
```Python
def audit_city_name(osmfile):
    city_names = {}
    for event, elem in ET.iterparse(osmfile):
        if elem.tag == "tag":
            if elem.attrib['k'] == "addr:city":
                if elem.attrib['v'] not in city_names:
                    city_names[elem.attrib['v']] = 1
                else:
                    city_names[elem.attrib['v']] += 1
    return city_names
```

## Cleaning the Data
To address the inconsistencies I found within the data I used the following code to clean up street names, postal codes, state name, and city name and write the changes to a new file. 
* Street names: I changed all formats to the unabbreviated formats. 
* Postal codes: for those I knew were SF postal codes, I changed the longer formats to the 5 digit format. I did not change the postal codes for those that were outside of SF as I did not know what they should be. 
* State name: I changed all instances to the abbreviated format "CA".
* City name: I changed all instances to "San Francisco".
```python
def update_file(input_file,new_file):
    """Creates a new OSM XML file for updated state name, city name, postal codes, and street names"""
    import codecs
    import re
    with open(new_file,'w') as outfile:
        for event, elem in ET.iterparse(input_file):
            if elem.tag == "tag":
                if elem.attrib['k'] == "addr:state":
                    elem.set('v','CA')
                elif elem.attrib['k'] == "addr:city":
                    elem.set('v','San Francisco')
                elif elem.attrib['k'] == "addr:postcode":
                    p = postal_code_re.search(elem.attrib['v'])
                    if p:
                        postal_code = p.group()
                        elem.set('v',postal_code)
                elif elem.attrib['k'] == "addr:street":
                    m = street_type_re.search(elem.attrib['v'])
                    if m:
                        street_type = m.group()
                    if street_type in street_mapping.keys():
                        name = re.sub(street_type_re,street_mapping[street_type],elem.attrib['v'])
                        elem.set('v',name)
        outfile.write(ET.tostring(elem, encoding='UTF-8'))
        outfile.close()
```

## Exploring the Data Using SQL
#### Number of Ways
```SQL
SELECT count(*) 
FROM ways;
```
32967
#### Number of Nodes
```SQL
SELECT count(*) 
FROM nodes;
```
331614
#### Number of Unique Users
```SQL
SELECT COUNT(DISTINCT(uid)) 
FROM (SELECT uid FROM nodes UNION ALL SELECT uid FROM ways);
```
700
#### Top 10 Contributing Users
```SQL
SELECT user, COUNT(user) 
FROM (SELECT user FROM nodes UNION ALL SELECT user FROM ways) 
GROUP BY user 
ORDER BY COUNT(user) 
DESC LIMIT 10;
```
```SQL
Rub21              137616
ediyes             114541
Luis36995          36622
KindredCoda        10579
chetan-sfimport    5423
saikofish          5131
manings_sfheight   4803
Gregory Arenius    4768
BharataHS_sfimport 4131
oba510             3624
```
#### Top 5 Streets Top Contributing User Edits
```SQL
SELECT ways_tags.value 
FROM ways_tags 
JOIN ways ON ways_tags.id = ways.id 
WHERE ways.user = 'Rub21' AND ways_tags.key = 'street' 
GROUP BY ways_tags.value
ORDER BY COUNT(ways_tags.values)
LIMIT 5;
```
```SQL
Howard Street    3
Folsom Street    2
Franklin Street  2
Greenwich Street 2
Vallejo Street   2
```
#### Top 10 Amenities
```SQL
SELECT value, COUNT(*) 
FROM nodes_tags 
WHERE key='amenity' 
GROUP BY value 
ORDER BY COUNT(*) DESC 
LIMIT 10;
```
```SQL
restaurant      686
post_box        260
cafe            225
bicycle_parking 180
bench           150
bar             109
pub             87
bank            79
fast_food       79
car_sharing     72
```
#### Streets with the Most Restaurants
```SQL
SELECT value, COUNT(*) 
FROM nodes_tags 
WHERE id IN (SELECT id FROM nodes_tags WHERE value = 'restaurant') AND key = 'street' 
GROUP BY value 
ORDER BY count(*) DESC 
LIMIT 10;
```
```SQL
Polk Street     26
Geary Street    19
2nd Street      14
Columbus Avenue 13
Chestnut Street 11
Van Ness Avenue 11
Kearny Street   10
Larkin Street   10
Mission Street  10
Stockton Street 10
 ```

## Suggestions for Improving the Data
An issue that needs to be addressed with open source maps is the completeness of the maps. Major city centers are more likely to be complete due the amount of traffic that goes through them. However it is difficult to know how complete the outskirts of the city are, where the population is less dense. Those who contribute to OpenStreetMap may have a better understanding of amenities within the city center with very limited knowledge of what is outside of the city center. Having incomplete information in these areas gives us a skewed view of exactly what amenities and how many of these amenities are within a certain location.  

One improvement may be to suggest to OpenStreetMap users to update areas that seem less complete. The site could suggest to the user to update an area that is near the area the user is currently updating. The user may have an idea of what is around but may not have thought to actively update that area. However this may be difficult to implement as the map doesn't know what it doesn't know, in that an area could actually be complete but seem empty because there really are not many amenities in the area. The map may be able to determine whether or not an area is complete based on its proximity to a city center. An area's proximity to a city center could indicate the approximate density of the area (i.e. areas closer to the city center would be more dense than areas further from the city center) and thus help determine whether or not more amenities need to be mapped in the area. However an issue with applying this method is that not all cities are made equal and varying cities in varying locations with have differing densities.
